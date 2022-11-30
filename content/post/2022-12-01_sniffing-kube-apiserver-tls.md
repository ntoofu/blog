---
title: "パケットキャプチャからKubernetes APIのTLS通信を解析する"
---

この記事は[ニフクラ](https://www.nifcloud.com/) 等を提供している、[富士通クラウドテクノロジーズ Advent Calendar 2022](https://qiita.com/advent-calendar/2022/fjct) の 1 日目の記事です。

---

インフラ関連のトラブルシューティングの際にはパケットキャプチャを頼りにすることが多いのですが、
Kubernetesのトラブルシューティングの際に思うように調査できなかったことがあったため、
パケットキャプチャを利用した調査方法について調べてみました。

意地になってなんとか実現したという感じで、半分以上はあまり現実的な方法ではないので、
読み物として読んでいただくか、結論だけ知りたい人は 「まとめと所感」 の箇所まで読み飛ばしていただければと思います。

# 経緯

Kubernetesクラスタ内では様々なコントローラーが動作しており、大抵は `kube-apiserver` へ Kubernetes API の呼び出しをしています。
Kuberntes APIは送信元と送信先が相互にTLSによって認証されており、
そのためクライアント証明書の期限切れなどのトラブルが起こり得るのですが、
その原因箇所の特定がうまくできないということがありました。

うまくできなかった理由としては、以下のような要因が挙げられます。

* Podから `kube-apiserver` へのアクセスはNATされて送信元がNodeのアドレスになっている
* クライアント証明書の期限切れになる前の段階で "期限切れが近づいている" という形で検出しており、
  ハンドシェイクに失敗しているわけではなかったため、ログからの調査も難しかった
* 通信がTLS 1.3で保護されており、クライアント証明書の内容を確認できない
  * TLS 1.3 では、ハンドシェイクの途中から暗号化が始まる
* SSLKEYLOGFILEの出力をサポートしていない
  * [TLSのパケットキャプチャ方法](https://ntoofu.github.io/blog/post/pcap-for-tls-traffic/) の中でも触れているように、
    セッション鍵を記録するSSLKEYLOGFILEが出力できれば、パケットキャプチャしたTLSセッションを復号して分析できる
  * しかし、SSLKEYLOGFILEを出力するかどうかはアプリケーションの実装次第であり、kube-apiserverは現時点で対応していない
    * [golang標準のTLSライブラリ](https://pkg.go.dev/crypto/tls#example-Config-KeyLogWriter)はSSLKEYLOGFILEを出力する機能を持っているが、
      ライブラリを利用するアプリケーション側で `KeyLogWriter` を指定しないと機能しない

この時は結局、直感で怪しいと思ったコントローラーについて調べて原因を特定できたのですが、
似たようなトラブルがあったときのためにkube-apiserverの通信内容をパケットキャプチャから解析する手段を確立しておきたいと感じました。


# 手順

## eBPFによるSSLKEYLOGFILEの抽出

![image](https://user-images.githubusercontent.com/22923783/204931981-ca3b98cc-6abd-494b-ae99-3191749977cf.png)

golang標準のTLSライブラリ自体はSSLKEYLOGFILEの出力機能を持っていますが、
これが有効に働くのは `KeyLogWriter` が指定されている場合のみであり、
`kube-apiserver` はこれに対応していません。

しかし、Linuxでは動作中のプログラムに対して、特定の箇所が実行されるごとに好きな処理を走らせる方法が存在しているので、
これによってSSLKEYLOGFILE相当の情報を得ることを考えましょう。
そのような方法としてイメージしやすいところでは、gdbのようなデバッガによるブレークポイントの設定などが挙げられますが、ここではeBPFとuprobeを利用することにします。

(
eBPFはユーザーが記述したプログラムをカーネル空間で安全に動作させる仕組みであり、カーネル空間で処理が完結することによる効率化・高速化が狙えることから、
ネットワークの処理に活用されたり、カーネル内部の処理のプロファイリングに利用したり、活用が広がっています。
eBPFを利用する場合、eBPFプログラムを用意した上でこれをカーネルに渡し、そのeBPFプログラムをどのタイミングで呼び出させるかという指定をします（アタッチする）。
このアタッチ先には様々なものがあり、その一つにuprobeが挙げられます。
uprobeは任意のユーザ空間のプログラムの任意の箇所で別の処理を呼び出せるようにするための仕組みで、その際に呼び出す処理としてeBPFプログラムを利用できるということになります。
)

golangの標準のTLSライブラリでは、 `KeyLogWriter` の指定有無に関わらず、
SSLKEYLOGFILEの情報を書き出すタイミングで [`writeKeyLog`](https://cs.opensource.google/go/go/+/refs/tags/go1.19.3:src/crypto/tls/common.go;l=1336) という内部関数を呼び出しているため、
uprobeでこの関数呼び出しをフックし、関数の引数を参照してSSLKEYLOGFILE相当の情報を出力するeBPFプログラムを用意すればやりたいことができるはずです。

eBPFプログラムの作成に加え、それをアタッチしたり結果を処理する部分も含めた一連の流れを簡単に実装する手段として、bpftraceというツールがあります。
ただし、golangの場合ABIが独自であることもあり、そのまま使うのは難しいので [go-bpf-gen](https://github.com/stevenjohnstone/go-bpf-gen) を使います。
この中にはまさに今回必要としている `writeKeyLog` をフックしてSSLKEYLOGFILEを得るスクリプト `tlssecrets.bt` を生成する手順が含まれているので、それを使ってみましょう。

コンテナはホスト（Kubernetesノード）のカーネルをそのまま使うため、コンテナ内のプロセスが対象であったとしても、
bpftraceなどのツールはホストのOS上にインストールして動作させれば問題ありません。
uprobeで指定する実行ファイルパスには、コンテナ内で実行されるファイルのホスト上のパスを指定すればよく、
containerd利用環境では例えば以下のようにすればすぐにわかると思います。

```
$ sudo sh -c 'ls /var/lib/containerd/io.containerd.snapshotter.v1.overlayfs/snapshots/*/fs/usr/local/bin/kube-apiserver'
/var/lib/containerd/io.containerd.snapshotter.v1.overlayfs/snapshots/3/fs/usr/local/bin/kube-apiserver
```

## 関数位置の特定

では `go-bpf-gen` を使う準備も整ったので、早速使ってみると、以下のように上手くいきません。

```
$ sudo go-bpf-gen templates/tlssecrets.bt /var/lib/containerd/io.containerd.snapshotter.v1.overlayfs/snapshots/3/fs/usr/local/bin/kube-apiserver | sudo bpftrace -

2022/12/01 06:25:48 couldn't get regs abi (no symbol section). falling back to stack calling convention
No probes to attach
```

`no symbol section` というところからわかるように、`k8s.gcr.io` で配布されているコンテナイメージ内の `kube-apiserver` は、
symbolがstripされた状態で配布されているようです。
そして、調べた限りデバッグ用に別途シンボルが配布されているということもないようでした。
（この件については既に [指摘](https://github.com/kubernetes/kubernetes/issues/73986) があるので、今後改善されるかもしれません。）

```
$ sudo nm /var/lib/containerd/io.containerd.snapshotter.v1.overlayfs/snapshots/3/fs/usr/local/bin/kube-apiserver

nm: /var/lib/containerd/io.containerd.snapshotter.v1.overlayfs/snapshots/3/fs/usr/local/bin/kube-apiserver: no symbols
```

こうなってしまうと、 `go-bpf-gen` が生成するbpftraceスクリプト内でuprobeの設定先として指定されている関数名に対応するアドレスが特定できず、
uprobeを設定することができません。

かといって、トラブルシューティングのためにデバッグ用にシンボルを残した状態で `kube-apiserver` を自前でビルドし、差し替えるというのも大変です。
そこで、何とかして `writeKeyLog` 関数の位置を特定してみます。

まず `kube-apiserver` のビルド環境のバージョンを確認します。

```
$ sudo go version /var/lib/containerd/io.containerd.snapshotter.v1.overlayfs/snapshots/3/fs/usr/local/bin/kube-apiserver

/var/lib/containerd/io.containerd.snapshotter.v1.overlayfs/snapshots/3/fs/usr/local/bin/kube-apiserver: go1.19.1
```

このバージョンのgoビルド環境を用意し（配布されているコンテナイメージを使えば容易です）、TLSライブラリを利用した簡単なプログラムをビルドします。
ここでは、[ドキュメント内のexample](https://pkg.go.dev/crypto/tls#example-Dial) を利用しました。

```
$ docker run -it -v $PWD:/work -w /work --rm golang:1.19.1 go build tls_example.go
```

ここで生成されるバイナリのにはシンボルテーブルが残っており、かつ `writeKeyLog` 関数も含まれるため、
`writeKeyLog` に対応する箇所のバイト列が特定できます。

```
$ objdump -d tls_example

(略)
0000000000580880 <crypto/tls.(*Config).writeKeyLog>:
  580880:       4c 8d 64 24 f8          lea    -0x8(%rsp),%r12
  580885:       4d 3b 66 10             cmp    0x10(%r14),%r12
  580889:       0f 86 0d 02 00 00       jbe    580a9c <crypto/tls.(*Config).writeKeyLog+0x21c>
  58088f:       48 81 ec 88 00 00 00    sub    $0x88,%rsp
  580896:       48 89 ac 24 80 00 00    mov    %rbp,0x80(%rsp)
  58089d:       00
  58089e:       48 8d ac 24 80 00 00    lea    0x80(%rsp),%rbp
  5808a5:       00
  5808a6:       48 89 9c 24 98 00 00    mov    %rbx,0x98(%rsp)
  5808ad:       00
  5808ae:       48 89 bc 24 a8 00 00    mov    %rdi,0xa8(%rsp)
  5808b5:       00
  5808b6:       4c 89 8c 24 c0 00 00    mov    %r9,0xc0(%rsp)
  5808bd:       00
  5808be:       48 83 b8 28 01 00 00    cmpq   $0x0,0x128(%rax)
  5808c5:       00
  5808c6:       0f 84 bc 01 00 00       je     580a88 <crypto/tls.(*Config).writeKeyLog+0x208>
  5808cc:       48 89 84 24 90 00 00    mov    %rax,0x90(%rsp)
  5808d3:       00
  5808d4:       4c 89 94 24 c8 00 00    mov    %r10,0xc8(%rsp)
  5808db:       00
  5808dc:       4c 89 9c 24 d0 00 00    mov    %r11,0xd0(%rsp)
  5808e3:       00
  5808e4:       48 89 b4 24 b0 00 00    mov    %rsi,0xb0(%rsp)
  5808eb:       00
  5808ec:       4c 89 84 24 b8 00 00    mov    %r8,0xb8(%rsp)
  5808f3:       00
  5808f4:       4c 89 8c 24 c0 00 00    mov    %r9,0xc0(%rsp)
  5808fb:       00
  5808fc:       48 89 bc 24 a8 00 00    mov    %rdi,0xa8(%rsp)
  580903:       00
  580904:       44 0f 11 7c 24 50       movups %xmm15,0x50(%rsp)
  58090a:       44 0f 11 7c 24 60       movups %xmm15,0x60(%rsp)
  580910:       44 0f 11 7c 24 70       movups %xmm15,0x70(%rsp)
  580916:       48 89 d8                mov    %rbx,%rax
  580919:       48 89 cb                mov    %rcx,%rbx
  58091c:       0f 1f 40 00             nopl   0x0(%rax)
  580920:       e8 db b7 e8 ff          callq  40c100 <runtime.convTstring>
(略)
```

`0x580920` の手前までは、関数内での相対ジャンプ以外でアドレスに関する情報がないため、
(goのバージョンが同じであれば)別のプログラムにおいても同じバイト列の並びになっていることが期待できます。
そこで、その部分のみを抜き出し、 `kube-apiserver` との比較を試みます。

まず、プログラム本体を含むセクションである `.text` のアドレスとオフセットを確認し、
`objdump` で見つけた問題の箇所がファイル内でどの位置になるかを特定し、抜き出します。

```
$ readelf -S tls_example

There are 36 section headers, starting at offset 0x270:

セクションヘッダ:
  [番] 名前              タイプ           アドレス          オフセット
       サイズ            EntSize          フラグ Link  情報  整列
  [ 0]                   NULL             0000000000000000  00000000
       0000000000000000  0000000000000000           0     0     0
  [ 1] .text             PROGBITS         0000000000401000  00001000
       00000000001aadf7  0000000000000000  AX       0     0     32
  [ 2] .plt              PROGBITS         00000000005abe00  001abe00
       0000000000000210  0000000000000010  AX       0     0     16
(略)

$ dd if=tls_example of=wkl.bin bs=1 skip=$((0x580880 - 0x401000 + 0x1000)) count=$((0x580920 - 0x580880))

160+0 レコード入力
160+0 レコード出力
160 bytes copied, 0.00203572 s, 78.6 kB/s
```

ちょうどよいツールが思い当らなかったので、雑にバイナリ列比較スクリプトを用意し、
`writeKeyLog` に対応するバイト列の `kube-apiserver` での位置を特定します。

```
$ cat << "EOF" > find.py
import sys

def find(pattern, source):
    results = []
    offset = 0
    while True:
        i = source[offset:].find(pattern)
        if i >= 0:
            results.append(offset+i)
            offset += i + 1
        else:
            return results

with open(sys.argv[1], 'rb') as f1, open(sys.argv[2], 'rb') as f2:
    results = find(pattern=f1.read(), source=f2.read())
print(results)
EOF

$ sudo python find.py wkl.bin /var/lib/containerd/io.containerd.snapshotter.v1.overlayfs/snapshots/3/fs/usr/local/bin/kube-apiserver

[2708320]

$ printf "0x%x" 2708320

0x295360
```

ちゃんと一意に見つかったようです。


## パケットキャプチャと解析

今度こそ `go-bpf-gen` を使ってSSLKEYLOGFILEを取れるはずです。
先ほどの実行では `couldn't get regs abi` というメッセージも出ており、
ABIを理解されていないと引数の解釈が正しくできないことになってしまうので、
一旦 `tls_example` を使ってbpftraceスクリプトを生成します。

```
$ go-bpf-gen templates/tlssecrets.bt /work/tls_example | tee ~/tlssecrets.bt

// capture TLS secrets for use with wireshark.
uprobe:/work/tls_example:"crypto/tls.(*Config).writeKeyLog" {
         // func (c *Config) writeKeyLog(label string, clientRandom, secret []byte) error
         $label = str(reg("bx"), reg("cx"));
         // slices are passed as a pointer, length and then capacity so skip a register
         $clientRandom = buf(reg("di"), reg("si"));
         $secret = buf(reg("r9"), reg("r10"));

         printf("%s %rx %rx\n", $label, $clientRandom, $secret);
}
```

これを編集し、以下のようにします。

```diff
  // capture TLS secrets for use with wireshark.
- uprobe:/work/tls_example:"crypto/tls.(*Config).writeKeyLog" {
+ uprobe:/var/lib/containerd/io.containerd.snapshotter.v1.overlayfs/snapshots/3/fs/usr/local/bin/kube-apiserver:0x295360 {
           // func (c *Config) writeKeyLog(label string, clientRandom, secret []byte) error
```

(
※ `0x295360` はバイナリファイル内からバイナリ列を検索して見つけた位置であるため、ファイル内のオフセットであり、本来アドレスではありません。
`.text` のロード先のアドレスやオフセットに基づきアドレスを特定する必要があると思いましたが、
何故かシンボルテーブルの無いバイナリにuprobeを設定する場合、ファイルのオフセットを設定しないとうまくいきませんでした。
bpftraceの仕様なのかバグなのか、原因は不明です。
)

これを利用して、SSLKEYLOGFILEを作成します。

```
$ sudo bpftrace --unsafe tlssecrets.bt | sed '1d;s/\\x//g' > sslkeylogfile
```

同時に、パケットキャプチャも実施します。
`kube-apiserver` はホストのデフォルトのネットワークネームスペースで動いている（Podの `spec.hostNetwork` が `true`）ので、
通常通りtcpdumpなどでパケットキャプチャできます。
（そうではないPodなどの場合は、
[Topotestsを利用してルーティングについて勉強する](https://ntoofu.github.io/blog/post/how-to-learn-routing-with-topotests/) の
"パケットキャプチャ" の項目で紹介しているような `nsenter` を利用する方法が個人的にはおすすめです。）

```
tcpdump -i lo -w kube-apiserver.pcap tcp port 6443
```

(ここでは同じノード上のPodからの通信を見るため、 `lo` を指定)

最後に、 `go-bpf-gen` のREADMEの説明通り、 `editcap` で pcap ファイルに SSLKEYLOGFILE を埋め込んで、
それをWriresharkで解析すれば、TLS通信を復号状態で確認することができます。

```
editcap --inject-secrets tls,sslkeylogfile kube-apiserver.pcap decrypted.pcap
```

![image](https://user-images.githubusercontent.com/22923783/204927205-1b5b4778-995f-4a8f-9b81-611f263dda74.png)

## 環境情報

* bpftrace v0.16.0-46-gf78d
* Ubuntu 22.04 LTS
  * kernel: 5.15.0-33-generic

# まとめと所感

* SSLKEYLOGFILEを出力しないプログラムでも、eBPF + uprobeなどを利用して生成することができ、キャプチャしたTLS通信の復号が可能
  * bpftraceや(goプログラムの場合)[go-bpf-gen](https://github.com/stevenjohnstone/go-bpf-gen)などを使えば難しくはない
* デバッグ用にシンボルが残っていなかったりすると、eBPFを利用する難易度が跳ね上がる
  * 今回やったように、諦めなければなんとかなったりするが、大変だし条件次第で通用しないこともあり得る
  * stripしないかシンボルだけ別途取得できるようにしておくなど、出来れば事前に備えておいた方が良さそう

---

この記事は[富士通クラウドテクノロジーズ Advent Calendar 2022](https://qiita.com/advent-calendar/2022/fjct) の 1 日目の記事でした。

明日は [@SogoK](https://qiita.com/SogoK) さんが OpenID Connectクライアント実装に関する記事を投稿してくださるようです。
[昨年のアドベントカレンダーで社内CTFを開催したことを紹介しました](https://tech.fjct.fujitsu.com/entry/2021/12/06/094339)が、
その中でOpenID Connectによる認証を利用していたり、他の業務の中でも利用しているので、個人的にも関心の高い話題です。お楽しみに！

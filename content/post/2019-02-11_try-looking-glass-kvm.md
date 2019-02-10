+++
title = "Looking Glass (KVM)を「例のグラボ」で試した"
date = "2019-02-11T05:06:06+09:00"
+++

[Looking Glass](https://looking-glass.hostfission.com/) というものを「例のグラボ」で試してみた。

## Looking Glass とは

* 同名のものが色々ある気がするが, 今試すのは [これ](https://looking-glass.hostfission.com/)
    * 最近よく耳にするのはホログラフィックディスプレイである
      [Looking Glass](https://lookingglassfactory.com/)だが, これの話ではない
    * ネットワーク関連とかだと指定したホストにICMPパケットを投げてくれるサービスをLooking Glassと
      言うが, それの話でもない
* KVM (Kernel-based Virtual Machine) にグラフィックスカードをPass-throughで見せているときに,
  そのグラフィックスカードの出力をKVMホスト上で覗き見るソフトウェア
    * 低遅延であることが大きな特長
* KVMによる仮想化環境で, 今の所ゲストがWindows10の場合にのみ利用可能

### 背景

* Linuxデスクトップ上にKVMでWindows10をゲスト動作させ, そこでDirectXを利用するようなゲームを行う...
  といったユースケースが主に想定される
* LinuxでWindowsゲームを行う場合, 以下のような選択肢が主に考えられる (Looking Glassはこの2番めの方法)
    * [Proton](https://github.com/ValveSoftware/Proton)
        * Windows APIを変換することでWindowsアプリケーションを
          Linux等のOSで動かそうとする [Wine](https://www.winehq.org/) をforkしている
        * Vulkan APIというクロスプラットフォームな低レイヤーのグラフィックスAPIを利用して,
          DirectXを実装([DXVK](https://github.com/doitsujin/dxvk))することで,
          Linux上でDirectXを処理できるようにしている
    * Linux上で仮想マシンを動作させ, WindowsをゲストOSとして利用する
        * DirectXの処理をさせるため, ホスト上のグラフィックスカードをパススルーでゲストOSに見せる
    * ~~WindowsゲームがDirectXではなくVulkanを用いて作られるようになるのを待つ~~
* Protonについては試していないのでなんともわからないが, 仮想マシンにグラフィックスカードを
  パススルーする場合, そのグラフィックスカードの画面出力を利用することになる
    * つまり, KVMホストのLinuxでも画面出力を利用する場合は, Linux用とゲストのWindows用に
      2つのディスプレイ（又は1つのディスプレイの2つの画面入力）を消費しないといけない
    * **この点を遅延をほとんど生まずに解消できるのがLooking Glassのメリット**
        * 但し, 後述の通り実際には画面の接続だけはしたほうがよかったりする

### 動作原理

* パススルーしたグラフィックスカードのフレームバッファ（各種レンダリングの結果とも言える,
  実際に画面へ出力するデータを格納している）の中身を, KVMホストから参照してKVMホスト側の
  GUI環境に描画する
    * DXGI Desktop Duplicationという, 画面キャプチャに用いるDirectX関連のAPIを使って,
      フレームをメインメモリ上にコピーするプログラムをゲストWindows上で動作させる
    * ゲストOSにてメインメモリ上に取ってきたフレームは,
      ivshmem (Inter-VM shared memory) というKVMの機能を使って, KVMホストから参照できるようにする
    * KVMホストのGUI環境でクライアントのソフトウェアを動作させ, ivshmemを参照してフレームを描画する

{{< mermaid align="left" >}}
graph LR
  subgraph KVM_host
    subgraph guest_Windows
      agent(("Agent Software<br/>(looking-glass-host)"))
      game(("DirectX Application<br/>(such as games)"))
      subgraph passthrough_vga
        fb2(frame buffer)
      end
    end
    subgraph host_vga
      fb1(frame buffer)
    end
    client(("Client Software<br/>(looking-glass-client)"))
    ivshmem(ivshmem)
  end
  monitor1[Monitor]
  monitor2[Monitor]

  agent -- "DXGI Desktop Duplication API" --> fb2
  fb2 ==> agent
  game == render ==> fb2
  agent ==> ivshmem
  client ==> fb1
  client --> ivshmem
  ivshmem ==> client
  fb1 == output ==> monitor1
  fb2 -. output .-> monitor2
{{< /mermaid >}}

## 例のグラボ とは

事の顛末はちゃんと把握していないが, マイニング向けとして画面出力端子のないグラフィックスカードの
中古品が, 2018年末から2019年始にかけて非常に安価で大量に出回っている。
これが実は [この記事](https://media.dmm-make.com/item/4515/)のように, 画面出力端子を
復活させられるということもあって一部界隈で盛り上がっている。

### 自分の場合

ノリで1枚購入した。大きくわけて改造には

1. HDMI端子の解放（ブラケットの加工）
2. HDMI画面出力の解放（チップコンデンサのはんだ付け）
3. 2番目以降の画面出力の解放（ブラケットの加工, 端子の実装, 回路素子の実装）

とステップがあり, この2番目には1005サイズのチップコンデンサ等が必要で, その調達に
時間がかかっているので1番目の加工のみで組み込み, Looking Glassを使うことにした。

Looking Glassを使う場合, パススルーに使うグラフィックスカードの画面出力そのものは必要なく,
画面出力をするために「グラフィックスカードに画面が接続されている」ことを認識さえすればよい。

これはまさに「例のグラボ」改造の1段階目を経た状態でモニタを接続だけすれば実現される状態に
他ならない。（詳しくは知らないが, どうやら2段階目を経ないと画面出力そのものはなされないが,
回路に手を加えずとも画面の接続の検出などはできるらしく, 1段階目までで条件はクリアされる。）

ちなみに, ブラケット加工はハンドニブラがあれば簡単だった。

![](https://pbs.twimg.com/media/DxMxO-wU0AIKiZb.jpg:large)

## 手順

[Quickstart Guide](https://looking-glass.hostfission.com/quickstart) に従うのみではあるが,
流れと注意点のみ説明する。

### Windows ゲストでの作業

* ivshmemを利用するためのドライバをインストールする
* 動作に必要となるゲスト上で動作させるソフトウェア `looking-glass-host.exe` をインストール
    * バイナリをダウンロードして適当に配置するだけ
    * 利用形態次第だが, OS起動時に自動起動するようにしたほうが便利なため, レジストリをいじって設定

### KVM ホスト(Linux)での作業

* クライアントである `looking-glass-client` をビルドする
    * Quickstart GuideはDebian前提でパッケージを書いているが,
      他のdistroでも（少なくともGentoo Linuxでは）対応するものをインストールすればビルドはできる
* VMにivshmemの設定をする
    * libvirtを利用していたので, `virsh edit` コマンドを使ってXMLを編集
* ivshmemの設定
    * `/dev/shm` 以下にファイルを作成してパーミッション等を設定するだけ

### 注意点

* 軽く触れている通り, パススルーするグラフィックスカードは画面が接続されている認識がないと
  上手く行かない可能性が高いため, 実際に使わなくともモニタを接続したほうが良い
    * ダミーのドングルがあればそれでも良いはずだが, 試してはいない
* Windowsは仮想のグラフィックスカード(QXL等)とパススルーしたグラフィックスカードを認識することに
  なるが, 自分の環境では後者に対応する画面をプライマリにしないと `looking-glass-host` が上手く
  動作しなかった
* `looking-glass-client` がSpiceクライアントの機能も持っているため,
  キーボード操作は `looking-glass-client` 上で自然に行える
    * ただし, マウスに関しては現時点で動作が不安定であり,
      FAQにあるようにパススルーしてしまったほうが賢明
    * 音声に関しては `looking-glass-client` では実装していないので,
      `virt-manager` 等の音声転送にも対応した Spiceクライアントを併用するのも手
      (但し, `looking-glass-client` より先に接続しておかないとうまくいかない)

## 所感

* 低遅延を謳うだけあり, 遅延は気にならなかった。(コアなゲーマーでは無いからかもしれない)
* ivshmem というのを初めて知った, 面白そう

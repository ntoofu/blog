+++
title = "Topotestsを利用してルーティングについて勉強する"
date = "2022-06-23T02:18:18+09:00"
+++

ルーティングについての社内輪読会の発表のために、デモ環境の構築を [Topotests](http://docs.frrouting.org/projects/dev-guide/en/latest/topotests.html) で実施し、とても便利だったので紹介する。

# 背景

## デモ環境について

ルーティングプロトコルについて学ぶ中で、実際に動かしてその動作を確かめたいと考えることもあると思う。
実際に設定をして期待したとおりに経路情報が伝播するのか、障害時に代替の経路に切り替わってくれるのか、やり取りされるパケットの中身がどのようになっているのかなど、見たい部分もたくさんあるだろう。

しかし、複雑なルーティングプロトコルの動きを見るためには、ルーターが複数台(それも2,3台ではなく5台以上)欲しくなってくるのはほぼ間違いない。
これらを物理機器として準備するのは場合によっては難しく、x86などの一般的なコンピューター上でソフトウェアとして動作するもの（いわゆる仮想ルーター）を使うほうが、ハードルが低い。

ただし、ソフトウェア的に環境を作る場合、ネットワークのトポロジを模擬する部分に工夫が必要になってくる。
例えば、ルーター1つごとに仮想マシンを立てて複数の仮想NICを割り当てたり、コンテナなどのLinuxのNamespaceの機能とvethやbridgeなどの仮想ネットワーク機能を上手く使うといった具合だ。
そうなってくると、結局ソフトウェア的に環境を作るのも、相応の手間が必要になってしまう。

## Topotests

こうした問題を解消できる一つの方法として、[Topotests](http://docs.frrouting.org/projects/dev-guide/en/latest/topotests.html) を利用する方法がある。

Topotestsは、[FRRouting](https://frrouting.org/)(FRR)というLinux/Unixで動作するオープンソースのルーティングプロトコルスイートのプロジェクトの中で、そのテストを行うための[ライブラリ](https://github.com/FRRouting/frr/tree/master/tests/topotests/lib)である。
前述のLinuxのNamespaceの機能と仮想ネットワーク機能を利用して、ネットワークトポロジを再現し、ビルドしたFRRを動作させ、期待する動作をするか（正しくルートが学習されるか、メモリリークがないか等）というテストを、Topotestsによって実現している。
（トポロジ再現のキモは [micronet](https://github.com/FRRouting/frr/blob/8684ca8fd598340427d4b0f4ea359ff068124df2/tests/topotests/lib/micronet.py) という部分に集約されており、見ての通りiproute2コマンドを組み立てて実行している。）

Topotestsを用いたテストのために準備するトポロジの定義は、Pythonのコードの中に書くこともできるが、[トポロジを定義したJSONファイルを用意することもできる](http://docs.frrouting.org/projects/dev-guide/en/latest/topotests-jsontopo.html)。
多くのルーターとその間のリンクを定義すること考えると、Pythonのコードとして定義したほうがループなどの制御構文が利用できて簡単にも思えるが、ルーティング用のデーモンに渡す設定ファイルを別途準備する必要もあるため、意外と面倒になる。
JSONファイルで準備する場合、それらの設定まで同じJSONの中で表現できるため、総じて見るとよりシンプルになる。

また、[ドキュメントの中](http://docs.frrouting.org/projects/dev-guide/en/latest/topotests.html#writing-a-new-test)にも書かれている通り、`--topology-only` オプションを指定すれば、ネットワークトポロジの再現とFRRを動作させるところまで実行しつつ、テストは実施しないようにもできる。

したがって、Topotestsを使えば、JSONで定義したネットワークトポロジを、FRRとLinuxのNamespaceと仮想ネットワーク機能を用いて再現することができるということになる。

# 手順

## コンテナの用意

FRR自体のテストのためではなく、ルーティングの学習用に任意のトポロジを再現するためにTopotestsを使うという観点で考えると、[現時点](https://github.com/FRRouting/frr/tree/faa8c700e61ddbdb278c12787785df4eeecb23ff) では、[UbuntuベースのDockerイメージ](https://github.com/FRRouting/frr/blob/faa8c700e61ddbdb278c12787785df4eeecb23ff/docker/ubuntu20-ci/Dockerfile)をビルドして用いるのが良い。
(Dockerhubの `frrouting/frr` イメージは、pytest等のライブラリが入っていないため、ドキュメントに書かれているような方法でTopotestsを実行できないため。 `frrouting/topotests` というイメージもあるが、都度FRRをソースコードからビルドすることを前提としており、FRR自体のテストを考えていないのであればむしろ使いにくいと思う。)

ビルドはFRRのリポジトリのルートでdocker buildすれば良いだけ。
```
git clone https://github.com/FRRouting/frr
cd frr
docker build -t frr -f docker/ubuntu20-ci/Dockerfile .
```

## トポロジとテストの準備

トポロジを定義したJSONファイルを用意する。
[ドキュメント](http://docs.frrouting.org/projects/dev-guide/en/latest/topotests-jsontopo.html)や、[Gitリポジトリ内のTopotestsを利用したテスト](https://github.com/FRRouting/frr/tree/master/tests/topotests)で使っているJSONファイルを参考にすると良い。

また、そのトポロジを読み込むテストを用意する。
[example](https://github.com/FRRouting/frr/tree/faa8c700e61ddbdb278c12787785df4eeecb23ff/tests/topotests/example_topojson_test)の中にあるものを参考に作ると手っ取り早い。
テストケースが無いと、pytest経由で呼び出したときに実行するテストがないとみなされ何もせずに終了してしまうので、中身は空でも良いので `test_` で始まる関数を1つは用意する。

例えば、[こんな感じ](https://gist.github.com/ntoofu/901e300a0c92b84e88a35e05103dbe30)。

## 実行

準備ができたら、用意したファイルをマウントしつつ先にビルドしたコンテナを実行する。

```
docker run -v $PWD:/home/frr/frr/tests/topotests/mytest:ro --privileged --rm -it frr /bin/bash
```

あとは、コンテナ内で

```
cd /home/frr/frr/tests/topotests
sudo -E python3 -m pytest -s --topology-only mytest/test.py
```

のような感じで、用意していたテストファイルを実行する。
実行するとプロンプト画面が表示される。ここでvtyshで使えるコマンドを入力すると、作成されたルーター全てでそのコマンドを実行した結果が表示される。

```
unet> show ip route
------ Host: r1 ------
Codes: K - kernel route, C - connected, S - static, R - RIP,
       O - OSPF, I - IS-IS, B - BGP, E - EIGRP, N - NHRP,
       T - Table, v - VNC, V - VNC-Direct, A - Babel, D - SHARP,
       F - PBR, f - OpenFabric,
       > - selected route, * - FIB route, q - queued, r - rejected, b - backup
       t - trapped, o - offload failure

C>* 10.100.1.17/32 is directly connected, lo, 00:01:22
C>* 169.254.100.0/30 is directly connected, r1-r2-eth0, 00:01:22
C>* 169.254.100.4/30 is directly connected, r1-r4-eth1, 00:01:22
S>* 172.16.0.0/16 [1/0] unreachable (blackhole), weight 1, 00:01:22
------- End: r1 ------
------ Host: r2 ------
Codes: K - kernel route, C - connected, S - static, R - RIP,
       O - OSPF, I - IS-IS, B - BGP, E - EIGRP, N - NHRP,
       T - Table, v - VNC, V - VNC-Direct, A - Babel, D - SHARP,
       F - PBR, f - OpenFabric,
       > - selected route, * - FIB route, q - queued, r - rejected, b - backup
       t - trapped, o - offload failure

C>* 10.100.2.17/32 is directly connected, lo, 00:01:22
C>* 169.254.100.0/30 is directly connected, r2-r1-eth0, 00:01:22
C>* 169.254.100.8/30 is directly connected, r2-r3-eth1, 00:01:22
B>* 172.16.0.0/16 [20/0] via 169.254.100.1, r2-r1-eth0, weight 1, 00:01:20
------- End: r2 ------
------ Host: r3 ------
Codes: K - kernel route, C - connected, S - static, R - RIP,
       O - OSPF, I - IS-IS, B - BGP, E - EIGRP, N - NHRP,
       T - Table, v - VNC, V - VNC-Direct, A - Babel, D - SHARP,
       F - PBR, f - OpenFabric,
       > - selected route, * - FIB route, q - queued, r - rejected, b - backup
       t - trapped, o - offload failure

C>* 10.100.3.17/32 is directly connected, lo, 00:01:22
C>* 169.254.100.8/30 is directly connected, r3-r2-eth0, 00:01:22
C>* 169.254.100.12/30 is directly connected, r3-r5-eth1, 00:01:22
B>* 172.16.0.0/16 [20/0] via 169.254.100.9, r3-r2-eth0, weight 1, 00:01:20
------- End: r3 ------
------ Host: r4 ------
Codes: K - kernel route, C - connected, S - static, R - RIP,
       O - OSPF, I - IS-IS, B - BGP, E - EIGRP, N - NHRP,
       T - Table, v - VNC, V - VNC-Direct, A - Babel, D - SHARP,
       F - PBR, f - OpenFabric,
       > - selected route, * - FIB route, q - queued, r - rejected, b - backup
       t - trapped, o - offload failure

C>* 10.100.4.17/32 is directly connected, lo, 00:01:22
C>* 169.254.100.4/30 is directly connected, r4-r1-eth0, 00:01:22
C>* 169.254.100.16/30 is directly connected, r4-r5-eth1, 00:01:22
B>* 172.16.0.0/16 [20/0] via 169.254.100.5, r4-r1-eth0, weight 1, 00:01:20
------- End: r4 ------
------ Host: r5 ------
Codes: K - kernel route, C - connected, S - static, R - RIP,
       O - OSPF, I - IS-IS, B - BGP, E - EIGRP, N - NHRP,
       T - Table, v - VNC, V - VNC-Direct, A - Babel, D - SHARP,
       F - PBR, f - OpenFabric,
       > - selected route, * - FIB route, q - queued, r - rejected, b - backup
       t - trapped, o - offload failure

C>* 10.100.5.17/32 is directly connected, lo, 00:01:22
C>* 169.254.100.12/30 is directly connected, r5-r3-eth0, 00:01:22
C>* 169.254.100.16/30 is directly connected, r5-r4-eth1, 00:01:22
C>* 169.254.100.20/30 is directly connected, r5-r6-eth2, 00:01:22
B>* 172.16.0.0/16 [20/0] via 169.254.100.17, r5-r4-eth1, weight 1, 00:01:20
------- End: r5 ------
------ Host: r6 ------
Codes: K - kernel route, C - connected, S - static, R - RIP,
       O - OSPF, I - IS-IS, B - BGP, E - EIGRP, N - NHRP,
       T - Table, v - VNC, V - VNC-Direct, A - Babel, D - SHARP,
       F - PBR, f - OpenFabric,
       > - selected route, * - FIB route, q - queued, r - rejected, b - backup
       t - trapped, o - offload failure

C>* 10.100.6.17/32 is directly connected, lo, 00:01:22
C>* 169.254.100.20/30 is directly connected, r6-r5-eth0, 00:01:22
B>* 172.16.0.0/16 [20/0] via 169.254.100.21, r6-r5-eth0, weight 1, 00:01:20
------- End: r6 ------
```

特定のルーターだけで実行したい場合は、最初にルーター名（トポロジ定義のJSONの中に出てくる名前）を指定すれば良い。

```
unet> r5 r6 show ip bgp
------ Host: r5 ------
BGP table version is 1, local router ID is 10.100.5.17, vrf id 0
Default local pref 100, local AS 65005
Status codes:  s suppressed, d damped, h history, * valid, > best, = multipath,
               i internal, r RIB-failure, S Stale, R Removed
Nexthop codes: @NNN nexthop's vrf id, < announce-nh-self
Origin codes:  i - IGP, e - EGP, ? - incomplete
RPKI validation codes: V valid, I invalid, N Not found

   Network          Next Hop            Metric LocPrf Weight Path
*  172.16.0.0/16    169.254.100.13                         0 65003 65002 65001 ?
*>                  169.254.100.17                         0 65004 65001 ?

Displayed  1 routes and 2 total paths
------- End: r5 ------
------ Host: r6 ------
BGP table version is 1, local router ID is 10.100.6.17, vrf id 0
Default local pref 100, local AS 65006
Status codes:  s suppressed, d damped, h history, * valid, > best, = multipath,
               i internal, r RIB-failure, S Stale, R Removed
Nexthop codes: @NNN nexthop's vrf id, < announce-nh-self
Origin codes:  i - IGP, e - EGP, ? - incomplete
RPKI validation codes: V valid, I invalid, N Not found

   Network          Next Hop            Metric LocPrf Weight Path
*> 172.16.0.0/16    169.254.100.21                         0 65005 65004 65001 ?

Displayed  1 routes and 1 total paths
------- End: r6 ------
```

## 個々のルーターの設定変更

基本的にJSONファイルの定義でルーターに設定は入っているが、設定変更時の動きを確認したい場合などを考えれば、個別に設定変更をしたいこともあると思う。
しかし、先のプロンプトでは `configure` を叩いて設定変更するのは上手く行かない。
（少なくとも自分の環境ではそうだった。）

Topotestsのドキュメントにも記載がある方法だが、Namespaceに直接アクセスしてしまえば、個々のルーターのvtyshを直接呼び出すこともできるので、設定変更などはここから行うと良い。
（pytest実行してたどり着いたプロンプト画面は終了せずそのままで、別のシェルなどから操作する。）

```
sudo nsenter -a -t ${対象ルーターのプロセスID} vtysh
```

プロセスIDは、Topotests呼び出した環境の `/tmp/topotests/mytest.test/${ルーター名}.pid` のようなところに格納されているが、今回Dockerを使っていて `docker exec` を挟む必要がある。
それも面倒なので、 `ps` コマンドで `zebra` などのFRRのプロセスを探して直接Namespaceにアクセスしに行くほうが楽かもしれない。

## パケットキャプチャ

パケットキャプチャして確認したい場合も同様にNamespaceに入って `tcpdump` などを呼べば良い。
先にビルドしたfrrのコンテナにはtcpdumpが入っていないが、 `nsenter -a` の変わりに `nsenter -n` などとすれば、対象プロセスのnetnsを使いつつファイルシステムなどはデフォルトのものを使うことになるため、ホストにtcpdumpが入っていればそれをそのまま使える。

```
$ sudo nsenter -n -t ${対象ルーターのプロセスID} bash --norc

bash-5.1# ip l
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN mode DEFAULT group default qlen 1000
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
2: sit0@NONE: <NOARP> mtu 1480 qdisc noop state DOWN mode DEFAULT group default qlen 1000
    link/sit 0.0.0.0 brd 0.0.0.0
13: r6-r5-eth0@if14: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue state UP mode DEFAULT group default qlen 1000
    link/ether 2a:01:3f:dd:e9:e0 brd ff:ff:ff:ff:ff:ff link-netnsid 0

bash-5.1# tcpdump -i r6-r5-eth0 tcp port 179
dropped privs to pcap
tcpdump: verbose output suppressed, use -v[v]... for full protocol decode
listening on r6-r5-eth0, link-type EN10MB (Ethernet), snapshot length 262144 bytes
02:08:43.924630 IP 169.254.100.22.43644 > 169.254.100.21.bgp: Flags [P.], seq 246667836:246667855, ack 634689374, win 126, options [nop,nop,TS val 1090370389 ecr 2792112341], length 19: BGP
02:08:43.924805 IP 169.254.100.21.bgp > 169.254.100.22.43644: Flags [P.], seq 1:20, ack 19, win 128, options [nop,nop,TS val 2792115341 ecr 1090370389], length 19: BGP
02:08:43.924814 IP 169.254.100.22.43644 > 169.254.100.21.bgp: Flags [.], ack 20, win 126, options [nop,nop,TS val 1090370389 ecr 2792115341], length 0
```

# まとめ

* ルーティングの勉強とかするにはたくさんルーターが必要で環境構築が大変
* Linuxホスト上にトポロジを作ってFRRをたくさん動かしてテストするFRR内のライブラリ、Topotestsが使える
* JSONファイル用意したらトポロジが作れて、各ルーターの設定をいじったりパケットキャプチャしたりもできる

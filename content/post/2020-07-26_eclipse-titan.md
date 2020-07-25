+++
title = "Eclipse TitanでTTCN-3を試した"
date = "2020-07-26T03:14:47+09:00"
+++

[TTCN-3](https://en.wikipedia.org/wiki/TTCN-3)について気になったので試してみた。

# TTCN-3 について

## TTCN-3 とは

* 通信関係のconformance testing (仕様や標準などに準拠しているかを調べるテストのこと) 用の言語のひとつ
    * テレコム系とかで使われていたりする？（推測）
* プロトコル通りに動作するかを確かめたりするためのものなので, 通信のやり取りを柔軟に記述できる
    * 対象とするレイヤ次第ではIPパケットやEthernetフレームのレベルで操作でき, 自由にヘッダを設定して送信したりできる
    * 指定したパターンを満たすメッセージ(フレームやパケット等)を受け取ったら…という条件分岐も簡単に書ける

## TTCN-3 の実行モデル

* parallelなテスト実行をサポートしていて, 1つのテストで複数のホスト間のやり取りを記述することが出来る
    * componentと呼ばれるある種のオブジェクトとそのcomponent間のやりとりをテストに記述でき, componentをそれぞれ別のホストに割り当てて動作させることが出来る
    * テスト用の実行ファイルを各テスト実行用ホスト(host controller)に配置して実行すると, 制御用ホスト(main controller)にTCP接続し, 制御用ホスト上に存在する設定ファイルに基づいてテストが実行され, テストの結果やログは制御用ホストに収集される

## TTCN-3 でできそうなこと

* ネットワーク機器やNFVなどが期待通りの動作をしているかのテスト
    * 例えばファイアウォールを挟んだ2ホスト間で通信を行い, 期待通りにパケットを通過・遮断させるのか
* あるプロトコルを実装したソフトウェアが, そのプロトコルでの特定のリクエスト等に対して期待した応答をするか
    * 例えばDNSサーバーに適当なクエリを投げ, 期待した応答が得られるか
        * [こちらの書籍](https://yakecsson.booth.pm/items/1571827)の中では例としてそのような題材を扱っていた

# Eclipse Titanについて

* TTCN-3自体は標準化された言語であり, その処理系の1つとしてEclipse Titanが存在する
* その名の通り[Eclipse Foundation](https://www.eclipse.org/)傘下のプロジェクト
* TTCN-3のファイルをEclipse Titanでコンパイル(トランスパイル?)するとC++のコードになり, これをコンパイルしてテスト実行用のバイナリを得る…といった流れになる
　
TTCN-3の書き方も含めて, 使い方については https://projects.eclipse.org/projects/tools.titan/downloads にまとめて公開されているPDFなどを参照するのが良い。

# 試してみた

https://github.com/ntoofu/ttcn3-titan-experiments に試したものを置いた。以下で簡単に解説しておく。

2つのテストを書いているが, いずれも `docker-compose up` すれば実行できるように書いている。

## [`01-tcp_syn_response`](https://github.com/ntoofu/ttcn3-titan-experiments/blob/aa681128f08536ea084342bf989844ae8ea34b81/01-tcp_syn_response/tcp_syn_response.ttcn)

{{< mermaid align="left" >}}
graph TD
  subgraph mc
    MTC
  end
  subgraph sut
    nc
  end

  MTC -- SYN --> nc
  nc -. SYN+ACK .-> MTC
{{< /mermaid >}}

* `nc` を実行させておいたコンテナにSYNを送信し, SYN+ACKが返ってくることを確認するテスト
    * ここではncとしたが, 基本的にはnc(そこに至るまでの経路を含む)が正常に応答するかを確かめていることになる
* [testcase](https://github.com/ntoofu/ttcn3-titan-experiments/blob/aa681128f08536ea084342bf989844ae8ea34b81/01-tcp_syn_response/tcp_syn_response.ttcn#L16-L33)がテストケースを記述する部分で, SYNを送信[(L19)](https://github.com/ntoofu/ttcn3-titan-experiments/blob/aa681128f08536ea084342bf989844ae8ea34b81/01-tcp_syn_response/tcp_syn_response.ttcn#L19)した後に, 3パターンの分岐を書いている
    * SYN+ACKを受信するか[(L22)](https://github.com/ntoofu/ttcn3-titan-experiments/blob/aa681128f08536ea084342bf989844ae8ea34b81/01-tcp_syn_response/tcp_syn_response.ttcn#L22) → この場合のみtestはpass
    * それ以外の何かを受信するか[(L25)](https://github.com/ntoofu/ttcn3-titan-experiments/blob/aa681128f08536ea084342bf989844ae8ea34b81/01-tcp_syn_response/tcp_syn_response.ttcn#L25)
    * タイムアウト[(L28)](https://github.com/ntoofu/ttcn3-titan-experiments/blob/aa681128f08536ea084342bf989844ae8ea34b81/01-tcp_syn_response/tcp_syn_response.ttcn#L28)
* 送受信するものは `PDU_TCP` のtemplateとして定義している[(L35-L48)](https://github.com/ntoofu/ttcn3-titan-experiments/blob/aa681128f08536ea084342bf989844ae8ea34b81/01-tcp_syn_response/tcp_syn_response.ttcn#L35-L48),[(L59-L72)](https://github.com/ntoofu/ttcn3-titan-experiments/blob/aa681128f08536ea084342bf989844ae8ea34b81/01-tcp_syn_response/tcp_syn_response.ttcn#L59-L72)
    * 省略の `omit` や, 任意の値を許容する `?` が使える
* testcaseは `runs on MTC` の通り `MTC` という名のcomponent[(L83-L85)](https://github.com/ntoofu/ttcn3-titan-experiments/blob/aa681128f08536ea084342bf989844ae8ea34b81/01-tcp_syn_response/tcp_syn_response.ttcn#L83-L85)上で動作する
    * `MTC` には `TcpPort` というタイプの port が指定されている
    * portを使うことで, それより下層の実装は気にせずにメッセージを送受信できる
        * 例えば, [UDPasp](https://github.com/eclipse/titan.TestPorts.UDPasp)を使えば, そのportにはデータグラム部のみ渡せば勝手にUDPパケットとして別途設定ファイルなどから設定したIPアドレス, ポート番号を使ってデータを送ってくれる
    * portは通常 https://github.com/eclipse?q=titan.TestPort にあるようなものを利用できるが, ここでは適切なものがなかったため, translation portという仕組み([参考](https://www.eclipse.org/forums/index.php/m/1799025/))でEthernetフレーム用のportを拡張してportを定義している

## [`02-tcp_handshake`](https://github.com/ntoofu/ttcn3-titan-experiments/blob/aa681128f08536ea084342bf989844ae8ea34b81/02-tcp_handshake/tcp_handshake.ttcn)

{{< mermaid align="left" >}}
graph TD
  subgraph mc
    main_controller
  end
  subgraph mtc
    MTC
  end
  subgraph ptc1
    client
  end
  subgraph ptc2
    server
  end

  MTC === client
  MTC === server
  client -- SYN --> server
  server -- SYN+ACK --> client
  client -- ACK --> server
{{< /mermaid >}}

* host controllerの間で3way handshakeに相当する通信を行わせ, きちんとパケットがやりとり出来るかを確認
    * ここではコンテナで実施しているのでわかりにくいが, ptc1とptc2の間の経路をテストしており, 実際には間にファイアウォールなどが入ることをイメージしている
* createの第二引数で,component実行するホストを指定している[(L18-L19)](https://github.com/ntoofu/ttcn3-titan-experiments/blob/aa681128f08536ea084342bf989844ae8ea34b81/02-tcp_handshake/tcp_handshake.ttcn#L18-L19)
* testcase自体は `MTC` 上で実施されるため, `client`, `server` での送受信のイベントを制御するためにinternal portを用意している[(L261-L265)](https://github.com/ntoofu/ttcn3-titan-experiments/blob/aa681128f08536ea084342bf989844ae8ea34b81/02-tcp_handshake/tcp_handshake.ttcn#L261-L265)
    * `MTC` の `CP1`, `CP2` ポートをそれぞれ `client`, `server` の `CP` portとつなぎ[(L20-L21)](https://github.com/ntoofu/ttcn3-titan-experiments/blob/aa681128f08536ea084342bf989844ae8ea34b81/02-tcp_handshake/tcp_handshake.ttcn#L20-L21), 同一視出来るように
        * もっと良い書き方があるかも…
    * `client`, `server` では `CP` portと `Tcp_P` portを相互に横流し[(L89-L106)](https://github.com/ntoofu/ttcn3-titan-experiments/blob/aa681128f08536ea084342bf989844ae8ea34b81/02-tcp_handshake/tcp_handshake.ttcn#L89-L106), 同一視出来るように
* `TcpPort` 内部で使っているIPアドレスなどの設定値をcomponent間で変えるために, translation portにおいて設定変更用のメッセージを追加している[(L276)](https://github.com/ntoofu/ttcn3-titan-experiments/blob/aa681128f08536ea084342bf989844ae8ea34b81/02-tcp_handshake/tcp_handshake.ttcn#L276)

# 所感

## 良いところ

* シンプルに低いレイヤでのテストが書けるし, 複数ホストに分散させて実行できる
    * 同等のことをpythonでfabric等を駆使してやってみたことがあるが, 滅茶苦茶面倒
    * コード全体はそれなりの長さはあったが, componentやport関連の部分は, ちょっとtestcaseを修正・追加する場合は触らないので, 頻繁にいじる部分で言えばかなりコンパクト
* 継続して開発されてるようなので, その点は安心感がある

## 気になったところ

* 情報の入手性が悪い
    * 日本語情報は先に紹介した同人誌以外ほとんど見かけなかった
    * 英語でも, 解説記事はあまり見かけず, 仕様書のような堅い内容のPDFファイルだったり, 解説しているスライドだったり
    * ただ, Eclipseのフォーラムが参考になったりはするし, 一度質問してみたら親切に回答して下さる方がいて助かった
        * アカウント作ってEclipseのContribution Agreementに同意したりと手間ではある…
* 仕方ないことだが, 学習コストはそこそこかかりそうな印象がある
    * 独特な概念がある
    * 情報の入手性の悪さも影響している

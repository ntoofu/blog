+++
title = "Container Network Interfaceについて調べた"
date = "2019-01-13T02:57:28+09:00"
draft = true
+++

社内のKubernetes勉強会に参加しており, 去年の夏頃からContainer Network Interface (CNI)に
関心があったのでその説明をすることになった。
その際の資料をここに書いておく。


## Container Network Interface (CNI) の基本

### CNIとは何か

* Linux container内のnetwork interfaceを設定するプラグインを書くための
  仕様とライブラリからなるCNCFプロジェクト
* containerのネットワーク接続性を作ることと削除時の後始末だけを考えているため,
  仕様は単純でサポート状況も良い(k8s専用というわけでもない)
* [GitHub](https://github.com/containernetworking/cni)

### CNIが生まれてきた背景

* Linux containerが急速に発展しcontainer runtimeやオーケストレータが複数出てきた
* 環境依存しやすいネットワーク周りをplugableにする方法は確立されていなかった
* それぞれが独自に問題を解決する無駄を避けるべく, CNIが共通のインターフェイスを定める
  * 元はと言えば `rkt` 用に考えられたものらしい

### CNI plugins

* CNIに準拠したpluginはたくさんあり,
  [READMEにも書いてある](https://github.com/containernetworking/cni#3rd-party-plugins)
* Golangで書かれたものは多いが, CNIは言語については規定していないため何で書いても良い

## CNI仕様

* [SPEC.md](https://github.com/containernetworking/cni/blob/master/SPEC.md)に仕様が記されている
* CNI からすれば container = Linux network namespace (netns)
  (runtimeによりnetnsを作る単位が異なるが, CNIとしてはcontainer = network namespaceとみなす)
  * そもそも, Linux containerから見えるネットワークインターフェイスは
    コンテナ用のnetns内のinterfaceでしかない
* "network" ごとに, どのpluginを用いるかなどの設定をCNIが定めるJSON形式で用意する
  * `name`: host内で一意な, networkを表す名前
  * `type`: そのnetworkで使うCNI pluginの実行ファイル名を指定
  * `cniVersion`: CNI specのバージョン
  * `ipam`: I/FへのIPアドレス割り当てを管理するIPAM pluginの設定
  * `dns`: container(netns)に対するDNSを設定する
  * その他, plugin固有の設定 (e.g. host側bridge interface名など)
* runtimeはCNIに従い, コンテナを希望のネットワークに接続するためpluginを呼び出す
* Layer2レベルでの疎通以上の, IPレベルでの設定について処理を共通化すべく
  IPAM pluginも用意されており(自作も可), アドレス設定などはそちらに任せる

{{< mermaid align="left" >}}
graph TD
  subgraph host
    subgraph container(netns) A
      if_a1[NIC]
      if_a2[NIC]
    end
    subgraph container(netns) B
      if_b1[NIC]
    end
  end
  nw1((network 1))
  nw2((network 2))
  conf1(config)
  conf2(config)
  r[runtime]
  cni[CNI plugin]
  ipam[IPAM plugin]

  nw1 --- conf1
  nw2 --- conf2

  r == invoke ==> cni
  cni == invoke ==> ipam

  conf1 -. indicate .-> cni
  conf1 -. indicate .-> ipam

  if_a1 --- nw1
  if_b1 --- nw2
  if_a2 --- nw2

  cni -- create --> if_a1
  ipam -- setup --> if_a1
{{< /mermaid >}}

* CNI pluginは...
  * runtimeから実行可能なファイルとして用意する
  * 次の操作に対応する
      * ADD: containerをネットワークに加える
          * container(=netns)にI/Fを作る
          * hostに対して必要な変更を加える
          * (適切なIPAM pluginを実行しIPアドレスのI/Fへのアサインとルートテーブルの調整をする)
      * DEL: ネットワークからcontainerを外す
          * 指定されたcontainer(netns)の指定されたI/Fが持っているリソースを全て削除する
      * CHECK: containerのネットワーク設定が想定通りか確認する
      * VERSION: サポートするCNI specのバージョン情報を返す
  * 操作及びそれに必要なパラメータは環境変数と標準入力を使って受け渡しする
      * `CNI_COMMAND`: 操作を指定 (`ADD`, `DEL`, `CHECK`, `VERSION`)
      * `CNI_CONTAINERID`: container ID = rutimeがcontainer(netns)に割り当てるユニークなID
      * `CNI_NETNS`: netnsに対応するファイルを指定
      * `CNI_IFNAME`: コンテナ内に作成されるI/Fの名前
      * `CNI_ARGS`: 追加の引数 ( `FOO=BAR;ABC=123` のようなKey-Value形式で渡す )
      * `CNI_PATH`: CNI pluginの実行ファイルの検索パス
      * 標準入力: networkごとに定義されたJSON形式のconfigを受け渡す
  * 結果は I/Fリスト, I/FごとのIP設定の情報, DNS情報 などをJSON形式で標準出力から返すのと,
    成否自体はreturn codeで示す
* version 0.3.0以降, pluginのchainも定められ, 独自の設定をしやすくなった
  * network configirationを連ねて作ったconfiguration listsを用意し, runtimeはこれに従って
    複数のpluginを順に呼び出す
  * runtimeは前のpluginの結果を `prevResult` 渡すため, 結果に応じた挙動ができる
  * ユースケース: アドレス割り当て実施後, ホスト側のiptableをいじってNAT設定を変える等

{{< mermaid align="left" >}}
sequenceDiagram
  participant Kernel
  participant Runtime
  participant Plugin1
  participant IPAM Plugin
  participant Plugin2
  Runtime->>Kernel: create netns
  Runtime->>Plugin1: "ADD" (netns, I/F name, network config, ...)
  Plugin1->>IPAM Plugin: "ADD" (netns, I/F name, network config, ...)
  IPAM Plugin-->>Plugin1: Result
  Plugin1-->>Runtime: Result
  Runtime->>Plugin2: "ADD" (netns, I/F name, network config(+prevResult), ...)
  Plugin2-->>Runtime: Result
{{< /mermaid >}}

## 実験: Docker container に CNI plugin でインターフェイスを足す

1. テスト用のbridgeインターフェイスをhost上に準備

    ```
    $ sudo ip link add test-bridge type bridge

    $ ip link show test-bridge
    23: test-bridge: <BROADCAST,MULTICAST> mtu 1500 qdisc noop state DOWN mode DEFAULT group default qlen 1000
        link/ether 46:16:ad:0e:cd:6a brd ff:ff:ff:ff:ff:ff
    ```

2. CNI pluginの準備

    ```
    $ git clone https://github.com/containernetworking/plugins.git

    $ cd plugins && ./build_linux.sh
    ```

3. テスト用コンテナを動かす(別のターミナルなどで)

    ```
    $ docker run -it --rm --name cni_test alpine:latest /bin/sh -c 'watch -n 5 ip addr'

    Every 5s: ip addr                                           2019-01-14 13:28:33

    1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN qlen 1000
        link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
        inet 127.0.0.1/8 scope host lo
            valid_lft forever preferred_lft forever
    2: sit0@NONE: <NOARP> mtu 1480 qdisc noop state DOWN qlen 1000
        link/sit 0.0.0.0 brd 0.0.0.0
    20: eth0@if21: <BROADCAST,MULTICAST,UP,LOWER_UP,M-DOWN> mtu 1500 qdisc noqueue state UP 
        link/ether 02:42:ac:11:00:02 brd ff:ff:ff:ff:ff:ff
        inet 172.17.0.2/16 brd 172.17.255.255 scope global eth0
            valid_lft forever preferred_lft forever
    ```

4. テスト用コンテナのnetnsにアクセスしてみる

    ```
    $ docker inspect cni_test | jq -r '.[].NetworkSettings.SandboxKey'
    /var/run/docker/netns/bf1e67c802ee

    $ sudo mkdir -p /var/run/netns

    $ sudo ln -s /var/run/docker/netns/bf1e67c802ee /var/run/netns/cni_test

    $ sudo ip netns exec cni_test ip addr
    ```

5. CNI pluginを呼び出してみる

    ```
    $ echo '{"cniVersion": "0.4.0", "name": "test", "type": "bridge", "bridge": "test-bridge", "ipam": { "type": "static", "addresses": [ { "address": "192.168.0.42/24" } ] } }' | CNI_COMMAND=ADD CNI_CONTAINERID=hoge CNI_NETNS=/var/run/docker/netns/bf1e67c802ee CNI_IFNAME=test-if CNI_PATH=$PWD/bin sudo -E $PWD/bin/bridge
    {
        "cniVersion": "0.4.0",
        "interfaces": [
            {
                "name": "test-bridge",
                "mac": "32:b2:ce:3d:ec:f1"
            },
            {
                "name": "veth38f6229d",
                "mac": "32:b2:ce:3d:ec:f1"
            },
            {
                "name": "test-if",
                "mac": "c6:26:ea:d0:71:8f",
                "sandbox": "/var/run/docker/netns/bf1e67c802ee"
            }
        ],
        "ips": [
            {
                "version": "4",
                "interface": 2,
                "address": "192.168.0.42/24"
            }
        ],
        "dns": {}
    }
    ```

6. コンテナ内にインターフェイスが増えていることを確認(手順3の出力を再度確認)

    ```
    Every 5s: ip addr                                                                                2019-01-14 13:56:54

    1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN qlen 1000
        link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
        inet 127.0.0.1/8 scope host lo
           valid_lft forever preferred_lft forever
    2: sit0@NONE: <NOARP> mtu 1480 qdisc noop state DOWN qlen 1000
        link/sit 0.0.0.0 brd 0.0.0.0
    4: test-if@if24: <BROADCAST,MULTICAST,UP,LOWER_UP,M-DOWN> mtu 1500 qdisc noqueue state UP 
        link/ether c6:26:ea:d0:71:8f brd ff:ff:ff:ff:ff:ff
        inet 192.168.0.42/24 scope global test-if
           valid_lft forever preferred_lft forever
    20: eth0@if21: <BROADCAST,MULTICAST,UP,LOWER_UP,M-DOWN> mtu 1500 qdisc noqueue state UP 
        link/ether 02:42:ac:11:00:02 brd ff:ff:ff:ff:ff:ff
        inet 172.17.0.2/16 brd 172.17.255.255 scope global eth0
           valid_lft forever preferred_lft forever
    ```

## Kubernetes と CNI

### Kubernetesのネットワーク

* Kubernetesのネットワーク実装に対する要件
  * 全てのcontainerは互いにNAT無しに通信できる
  * 全てのnode対containerの通信はNAT無しに可能
  * containerから見えるIPは他から見えるIPと同一
* Container間(Pod内): `localhost` として通信可能
  * Pod単位でnetwork namespaceを作っている
  * Kubernetestは "IP-per-pod" model = PodごとにIPが付く
      * (Container間でportの衝突は気にしないといけない)
* Pod間(Node内, Node間): PodについたIPアドレスで通信できる
* 要件を満たす範囲で, 実装は[複数](https://kubernetes.io/docs/concepts/cluster-administration/addons/)ある


### ユースケース

* kubeletのNetworkPluginとして
  * ???
* NetworkAttachmentDefinition

### 利用方法

## CNI plugin 各論

* 一部独断でpickupして説明
  * Flannel
  * Project Calico
  * Multus

### Flannel

* VXLANによる L2 over L3 ネットワークを実現する
  * Control planeの情報は etcd を使って共有
* [Flannel CNI plugin](https://github.com/containernetworking/plugins/tree/master/plugins/meta/flannel)
  は Flannelのネットワーク情報を反映したconfigを生成して `bridge` CNI pluginなどを呼び出すだけ
  * デフォルトでは, `bridge` CNI plugin + `host-local` IPAM plugin を呼び出す
* 通常の設定では, node内のbridgeネットワーク `cbr0` にそれぞれ異なるsubnetが割り当てられるが,
  nodeからはFlunnelの仮想I/Fを通じてL2接続しているように見える
  * Nodeの異なるPod間ではL2の接続性は無い？（多分）

### Project Calico

* Nodeごとに作られる仮想ルーターがPodのIPアドレスをiBGPで広報しあい, L3接続性を得る
* 得られるメリット
  * パフォーマンス面での優位性(カプセル化しないため)
  * 物理機器との連携 (e.g. [CalicoによるKubernetesピュアL3ネットワーキング](https://techblog.yahoo.co.jp/infrastructure/kubernetes_calico_networking/))
  * IPベースのアクセス制御がしやすい

### Multus

* Container(netns)内に複数のI/Fを作るためのplugin
* 設定に基づいて他のCNI pluginを呼び出すCNI plugin
* Network Function Virtualization方面からの需要？

## 参考にした情報源まとめ
* [containernetworking/cni](https://github.com/containernetworking/cni): CNIのGitHub rpository
* [CNI プラグインを手作業で利用する](https://qiita.com/yuanying/items/68b2a32b4d217e679955)
* [Dockerを支える技術](https://www.slideshare.net/enakai/docker-34668707): Dockerにおけるnetwork namespace関連について記述あり
* [Chaining CNI Plugins](https://karampok.me/posts/chained-plugins-cni/)
* [Cluster Networking - Kubernetes](https://kubernetes.io/docs/concepts/cluster-administration/networking/): KubernetesのPod-to-Podのネットワークのコンセプトについて
* [マルチホストでのDocker Container間通信 第3回: Kubernetesのネットワーク(CNI, kube-proxy, kube-dns)](https://tech.uzabase.com/entry/2017/09/12/164756)
* [How to Understand and Set Up Kubernetes Networking](https://dzone.com/articles/how-to-understand-and-setup-kubernetes-networking): Multus CNIのセットアップ関連など
* [Attaching To Multiple Networks](https://kubevirt.io/2018/attaching-to-multiple-networks.html)NetworkAttachmentDefinitionまわりの説明を含む
* [FlannelのVXLANバックエンドの仕組み](http://enakai00.hatenablog.com/entry/2015/04/02/173739): Flannel自体の動きについてわかりやすい
* [Kubernetes Network Deep Dive (NodePort, ClusterIP, Flannel)](https://qiita.com/sugimount/items/ed07a3e77a6d4ab409a8): KubernetesにおけるFlannelの挙動を細かく説明している

## 調べ足りないこと

* kube-proxy mode
* network policy


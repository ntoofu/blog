+++
title = "CNIプラグインを活用してKubernetesのPodをVLANにつなぐ"
date = "2020-12-21T01:46:18+09:00"
+++

この記事は[富士通クラウドテクノロジーズ Advent Calendar 2020](https://qiita.com/advent-calendar/2020/fjct)の21日目の記事です。

# TL;DR

* KubernetesでPodを外部のVLANを利用したネットワークと接続したい場合の一つの方法を検証した
* [Multus CNI](https://github.com/intel/multus-cni) と `bridge` CNI pluginを活用する
* 都度ノードを自ら設定したりする必要はなく、manifestを適切に与えるだけでPodを好きなVLANに直接(L2レベルでの疎通がある状態で)繋げられるようになる

# 背景

## PodをVLANに繋ぎたい動機

業務ではインフラの管理を中心に担当しているが、IaaSとしてサービス提供している環境（いわばクラウドの裏側）の管理であり、クラウド上のシステムではないため、泥臭いことを沢山する必要がある。例えばDHCPサーバーを構築してホストのアドレスを管理できるようにしたり、ユーザーに提供しているサーバーインスタンスが利用するネットワーク機能（ファイアウォール、ロードバランサー、VPN等）を提供することなどがある。

それら必要なものには仮想マシンとしてデプロイしているものも多くあるが、自動でダウンタイムなく安全に更新できるような構成を取れていないものも残っており、管理上負担になっている。コンポーネントによって手順も異なるため、手がかかっている。

そこで、それらをKubernetesの上に立て直すことで、デプロイメントやアップデートの方法を共通化し、リリースサイクルを高速化していくことが出来ないかを考えている。

しかし、KubernetesではPodのネットワークにはある程度の制約があるため、先に例に上げたようなある種の泥臭いことをKubernetes上でやろうとすると、一筋縄ではいかない。何も考えずにクラスタを構築すれば、PodはFlannelによって作られるオーバーレイネットワークのみに接続していて、DHCPによるアドレス払い出しを行うことも、ネットワーク間のパケット転送に関わることもできない。

そこで、今回はまずIEEE 802.1QのVLANに自由にちょっかいを掛けられるようにする検証をしてみた。

## CNIプラグイン

コンテナはランタイムの乱立と標準化の中で、コンテナ（より厳密にはLinux network namespace）のネットワークへの接続の実装を柔軟に取り回せるよう、インターフェイスとして[Container Network Interface](https://github.com/containernetworking/cni)が定められている。そして、そのインターフェイスに準拠して作られた実装がCNIプラグインである。（CNIについての詳細は[過去にも調べて軽くまとめている](https://ntoofu.github.io/blog/post/container-network-interface/)。）

KubernetesもこのCNIに対応していて、適切に設定することでPodネットワークの実装を切り替えたり出来る。（このあたりはクラスタ構築をする際に設定が必要なため、クラスタ構築をしたことがあれば心当たりがあると思う。）

今回はCNIプラグインとして以下で説明するMultus CNIとbridgeプラグインを使うことで、目的を果たせることを検証していく。
### Multus CNI

これも[過去に少し調べて書いている](https://ntoofu.github.io/blog/post/try-multus-cni/)が、簡単に言えば複数のCNIプラグインを呼び出せるようにするためのCNIプラグインで、コンテナ（Kubernetes的にはPod）の中に複数のネットワークインターフェイスを与えることなどが可能になる。

### bridge CNI plugin

標準的なCNIプラグインとして https://github.com/containernetworking/ の下で[提供](https://github.com/containernetworking/plugins/blob/master/plugins/main/bridge/)されている。主なところとしては

* Linux bridge interfaceを（無ければ）ホストに作成する
* コンテナが利用するnetns(network namespace)内と、ホストを結ぶLinux veth interfaceを作成する
* （必要に応じて）IPアドレスをnetns内のインターフェイスに割り当てる
* 設定によってはこのbridgeを介して外部にSNAT(Masquerade)でIPによる通信が出来るようホストを設定する

といったものになっている。直接使うことはあまりないかもしれないが、実は今なにも考えずにflannelを利用すれば[内部的にbridgeプラグインを呼び出し](https://github.com/containernetworking/plugins/blob/master/plugins/meta/flannel/flannel_linux.go#L59)ている。

更にこのbridgeプラグインは、Linux bridge interfaceの [vlan_filtering](https://developers.redhat.com/blog/2017/09/14/vlan-filter-support-on-bridge/) 利用できるようになっている。余談だが、[このPR](https://github.com/containernetworking/plugins/pull/231)で実装されたようなので、以前検証した時にはなく、今回自分でプラグインを作成または拡張しようとして、参考にソースコードを読み込んでいて気がついた。

これを上手く使えば、必要なネットワークの定義などをほとんどKubernetesのmanifestに完結させて行うことが出来るはずなので、それを確認していく。

### 他のプラグイン

https://github.com/containernetworking/plugins/ の中には、他にもプラグインがあるのでそれについても少し言及しておく。

* vlan: 今回とかなり近いことが可能だが、VLAN IDごとにvlan typeの仮想インターフェイスを作成し、それをコンテナ(厳密にはnetwork namespace)に直接渡してしまう。そのため、同一Node上で複数のPodが同一VLAN IDを利用できない。
* ipvlan, macvlan: どちらもvlanと名前に含まれているが、Linuxにおけるipvlan, macvlanタイプの仮想インターフェイスのことを指しており、IEEE 802.1QのVLAN tagを付け外しすることはない。


# 検証

## テスト用ネットワークの準備

* ノード対してNICを追加し、untagされていないフレームを転送するよう(トランクポートとなるよう)スイッチを設定
    * 画像はテスト環境に利用していたVMware vSphereのUIからのスクリーンショット

        ![](/images/k8s-vlan-vswitch.png)
    * `admin-md-...` が kubernetesのノードで、testvmがkubernetes外部のホストであり、特定のVLANを利用しているもの（疎通の確認に利用する）
    * 画像にはないが、kubernetesのノードには通常の通信で利用するインターフェイスが付いている
* （以降の例の中では、`trunk0` という名前で追加されたNICが認識されたとする）
* 仮想スイッチなどを利用する場合は、設定を要確認
    * vSphereの場合、vNICに割り当てたMACアドレス以外のものを利用して通信することになるため、promiscuousとforged transmitを許可する必要がある

## ノードの準備

* 接続用のbridgeを事前に作っておく
    * CNIプラグインが作成してくれるが、物理インターフェイスをbridgeに参加させてくれないので、事前にやっておく
    * 各ノードで以下を実行しておく

        ```shell
        ip link add vlanbr0 type bridge vlan_filtering 1  # vlanbr0というbridgeをvlan_filtering有効で作成
        ip link set dev trunk0 master vlanbr0  # 物理インターフェイスtrunk0とbridge vlanbr0を接続
        bridge vlan add dev trunk0 vid 1-4094  # 物理インターフェイスtrunk0で任意のtag付きフレームを受け取るように
        ```
        * Cluster APIで構築していたので、 `cluster.yaml` の `KubeadmConfigTemplate` の `.spec.template.spec.preKubeadmCommands` あたりで実行するようにしたが問題はなかった
        * いずれにせよVLANに接続したいPodを配置しうるノード全てで事前実行が必要なので、自動でできるようにしておくべきだろう
* 必要なCNIプラグインを導入しておく
    * `/opt/cni/bin` 配下に配置
    * ある程度新しい(vlanをサポートする) `bridge` プラグインが入っていること
    * IPアドレスの割り当てもさせるのであれば、 `static` プラグインが入っていること
        * アドレス割り当てさせないことも可能([L2-only configuration](https://www.cni.dev/plugins/main/bridge/#example-l2-only-configuration))だが、vlanのサポートよりさらに新し目のbridgeプラグインを用意しておく必要がありそう

## Multusの設定

* クラスタ自体は構築済みで、flannelが導入されている状態を仮定する
* Multusをインストールするために https://github.com/intel/multus-cni/blob/d4e86998253d2c0d44f3d8317887ff7c926e7756/images/multus-daemonset.yml を apply する

    ```shell
    kubectl apply -f https://raw.githubusercontent.com/intel/multus-cni/master/images/multus-daemonset.yml
    ```
* 以下のMultus用の設定を作成し、こちらもapply

    ```yaml
    apiVersion: "k8s.cni.cncf.io/v1"
    kind: NetworkAttachmentDefinition
    metadata:
      name: vlan-2001
    spec:
      config: '{
          "cniVersion": "0.3.1",
          "plugins": [
            {
              "type": "bridge",
              "bridge": "vlanbr0",
              "vlan": 2001,
              "ipam": {
                "type": "static"
              }
            }
          ]
        }'
    ---
    apiVersion: "k8s.cni.cncf.io/v1"
    kind: NetworkAttachmentDefinition
    metadata:
      name: vlan-2002
    spec:
      config: '{
          "cniVersion": "0.3.1",
          "plugins": [
            {
              "type": "bridge",
              "bridge": "vlanbr0",
              "vlan": 2002,
              "ipam": {
                "type": "static"
              }
            }
          ]
        }'
    ---
    ```

## テスト用Podによる動作確認

### 準備

* 以下のmanifestをapplyしてVLANに足を出すテスト用のpodを4つ作成

    ```yaml
    apiVersion: v1
    kind: Pod
    metadata:
      name: pod1a
      annotations:
        k8s.v1.cni.cncf.io/networks: '[{"name": "vlan-2001", "ips": ["10.20.1.11/24"]}]'
    spec:
      containers:
        - name: alpine
          image: alpine
          command: ["tail", "-f", "/dev/null"]
    ---
    apiVersion: v1
    kind: Pod
    metadata:
      name: pod1b
      annotations:
        k8s.v1.cni.cncf.io/networks: '[{"name": "vlan-2001", "ips": ["10.20.1.12/24"]}]'
    spec:
      containers:
        - name: alpine
          image: alpine
          command: ["tail", "-f", "/dev/null"]
    ---
    apiVersion: v1
    kind: Pod
    metadata:
      name: pod1c
      annotations:
        k8s.v1.cni.cncf.io/networks: '[{"name": "vlan-2001", "ips": ["10.20.1.13/24"]}]'
    spec:
      containers:
        - name: alpine
          image: alpine
          command: ["tail", "-f", "/dev/null"]
    ---
    apiVersion: v1
    kind: Pod
    metadata:
      name: pod2
      annotations:
        k8s.v1.cni.cncf.io/networks: '[{"name": "vlan-2002", "ips": ["10.20.2.11/24"]}]'
    spec:
      containers:
        - name: alpine
          image: alpine
          command: ["tail", "-f", "/dev/null"]
    ```
    * `k8s.v1.cni.cncf.io/networks` のannotationが肝

### 確認

* podの配置を確認

    ```
    $ kubectl get pod -o wide
    NAME    READY   STATUS    RESTARTS   AGE   IP         NODE                          NOMINATED NODE   READINESS GATES
    pod1a   1/1     Running   0          10s   10.9.3.8   admin-md-0-5644c5f488-lz9sh   <none>           <none>
    pod1b   1/1     Running   0          10s   10.9.2.7   admin-md-0-5644c5f488-t6qzp   <none>           <none>
    pod1c   1/1     Running   0          10s   10.9.2.8   admin-md-0-5644c5f488-t6qzp   <none>           <none>
    pod2    1/1     Running   0          10s   10.9.3.9   admin-md-0-5644c5f488-lz9sh   <none>           <none>
    ```

つまり、以下のような状態になっている。

![](/images/k8s-vlan-node-topology.svg)

(二重線の箇所はタグ付きのフレームがやり取りされるイメージ)

* インターフェイスを確認

    ```
    $ kubectl exec -it pod1a -- ip a
    1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN qlen 1000
        link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
        inet 127.0.0.1/8 scope host lo
           valid_lft forever preferred_lft forever
        inet6 ::1/128 scope host
           valid_lft forever preferred_lft forever
    3: eth0@if18: <BROADCAST,MULTICAST,UP,LOWER_UP,M-DOWN> mtu 1450 qdisc noqueue state UP
        link/ether 8e:c7:8f:53:de:0b brd ff:ff:ff:ff:ff:ff
        inet 10.9.3.8/24 brd 10.9.3.255 scope global eth0
           valid_lft forever preferred_lft forever
        inet6 fe80::8cc7:8fff:fe53:de0b/64 scope link
           valid_lft forever preferred_lft forever
    5: net1@if19: <BROADCAST,MULTICAST,UP,LOWER_UP,M-DOWN> mtu 1500 qdisc noqueue state UP
        link/ether a2:a1:03:d8:90:bb brd ff:ff:ff:ff:ff:ff
        inet 10.20.1.11/24 brd 10.20.1.255 scope global net1
           valid_lft forever preferred_lft forever
        inet6 fe80::a0a1:3ff:fed8:90bb/64 scope link
           valid_lft forever preferred_lft forever
    ```
    * `net1` というのが出来ていてIPアドレスも指定したものが付いている
        * `eth0` の方はpod network用でflannel経由になる

* arpingによる外部との疎通テスト

    ```
    $ kubectl exec -it pod1a -- arping -I net1 -c 1 10.20.1.1
    ARPING 10.20.1.1 from 10.20.1.11 net1
    Unicast reply from 10.20.1.1 [00:50:56:8f:6e:6e] 0.386ms
    Sent 1 probe(s) (0 broadcast(s))
    Received 1 response(s) (0 request(s), 0 broadcast(s))

    $ kubectl exec -it pod1b -- arping -I net1 -c 1 10.20.1.1
    ARPING 10.20.1.1 from 10.20.1.12 net1
    Unicast reply from 10.20.1.1 [00:50:56:8f:6e:6e] 0.393ms
    Sent 1 probe(s) (0 broadcast(s))
    Received 1 response(s) (0 request(s), 0 broadcast(s))

    $ kubectl exec -it pod1c -- arping -I net1 -c 1 10.20.1.1
    ARPING 10.20.1.1 from 10.20.1.13 net1
    Unicast reply from 10.20.1.1 [00:50:56:8f:6e:6e] 0.088ms
    Sent 1 probe(s) (0 broadcast(s))
    Received 1 response(s) (0 request(s), 0 broadcast(s))

    $ kubectl exec -it pod2 -- arping -I net1 -c 1 10.20.2.1
    ARPING 10.20.2.1 from 10.20.2.11 net1
    Unicast reply from 10.20.2.1 [00:50:56:8f:03:0f] 1.131ms
    Sent 1 probe(s) (0 broadcast(s))
    Received 1 response(s) (0 request(s), 0 broadcast(s))

    $ kubectl exec -it pod2 -- arping -I net1 -c 1 10.20.1.1
    ARPING 10.20.1.1 from 10.20.2.11 net1
    Sent 1 probe(s) (0 broadcast(s))
    Received 0 response(s) (0 request(s), 0 broadcast(s))
    command terminated with exit code 1
    ```
    * `.1` のアドレスはテスト用の仮想マシンのもので、ちゃんと狙ったVLANと疎通できている
    * 逆に狙っていないVLAN側からの応答はなく、意図しない疎通もなさそう（一応別途テスト用仮想マシンでのパケットキャプチャもして確認済み）
    * ARPで確認しており、L2での疎通があることが分かる
        * `00:50:56` で始まるMACアドレスはテスト用VMのものであり、proxy ARPとかでもない
* arpingによるpod間疎通テスト

    ```
    $ kubectl exec -it pod1b -- ip a show dev net1
    5: net1@if17: <BROADCAST,MULTICAST,UP,LOWER_UP,M-DOWN> mtu 1500 qdisc noqueue state UP
        link/ether 8a:ba:6d:1b:db:14 brd ff:ff:ff:ff:ff:ff
        inet 10.20.1.12/24 brd 10.20.1.255 scope global net1
            valid_lft forever preferred_lft forever
        inet6 fe80::88ba:6dff:fe1b:db14/64 scope link
            valid_lft forever preferred_lft forever

    $ kubectl exec -it pod1a -- arping -I net1 -c 1 10.20.1.12
    ARPING 10.20.1.12 from 10.20.1.11 net1
    Unicast reply from 10.20.1.12 [8a:ba:6d:1b:db:14] 0.600ms
    Sent 1 probe(s) (0 broadcast(s))
    Received 1 response(s) (0 request(s), 0 broadcast(s))

    $ kubectl exec -it pod1c -- arping -I net1 -c 1 10.20.1.12
    ARPING 10.20.1.12 from 10.20.1.13 net1
    Unicast reply from 10.20.1.12 [8a:ba:6d:1b:db:14] 0.015ms
    Sent 1 probe(s) (0 broadcast(s))
    Received 1 response(s) (0 request(s), 0 broadcast(s))
    ```
    * Node間、Node内いずれもPod間でARP応答を得ている
* ノードでの見え方

    ```
    root@admin-md-0-5644c5f488-lz9sh:~# bridge vlan show
    port  vlan ids
    trunk0   1
             2
    (中略)
           4093
           4094

    vlanbr0	 1 PVID Egress Untagged

    cni0	 1 PVID Egress Untagged

    veth72b16477	 1 PVID Egress Untagged

    veth6bcdbf48	 1 PVID Egress Untagged

    veth4bedd573	 1 Egress Untagged
       2001 PVID Egress Untagged

    vethe2060431	 1 PVID Egress Untagged

    vethe37ab17d	 1 Egress Untagged
       2002 PVID Egress Untagged

    root@admin-md-0-5644c5f488-lz9sh:~# tcpdump -i vlanbr0 -e -nn arp
    21:03:11.980833 5a:d2:1d:28:a9:f2 > 00:50:56:8f:03:0f, ethertype 802.1Q (0x8100), length 46: vlan 2002, p 0, ethertype ARP, Request who-has 10.20.2.1 (00:50:56:8f:03:0f) tell 10.20.2.11, length 28
    21:03:11.981278 00:50:56:8f:03:0f > 5a:d2:1d:28:a9:f2, ethertype 802.1Q (0x8100), length 64: vlan 2002, p 0, ethertype ARP, Reply 10.20.2.1 is-at 00:50:56:8f:03:0f, length 46
    21:03:12.638034 a2:a1:03:d8:90:bb > 00:50:56:8f:6e:6e, ethertype 802.1Q (0x8100), length 46: vlan 2001, p 0, ethertype ARP, Request who-has 10.20.1.1 (00:50:56:8f:6e:6e) tell 10.20.1.11, length 28
    21:03:12.638448 00:50:56:8f:6e:6e > a2:a1:03:d8:90:bb, ethertype 802.1Q (0x8100), length 64: vlan 2001, p 0, ethertype ARP, Reply 10.20.1.1 is-at 00:50:56:8f:6e:6e, length 46
    ```

# 少し気になるがまだ調べられていないこと

* パフォーマンスについては未検証
    * あまり期待しないほうがよいとは思っている
* [firewall](https://github.com/containernetworking/plugins/tree/master/plugins/meta/firewall)プラグインなどを併せて使うことはできるのか
* Deployment内で複数のPodなどを持つようにした場合に、VLAN接続用インターフェイスでアドレス競合しないようにする方法

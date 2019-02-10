+++
title = "Multus CNI pluginをKubernetesで試した"
date = "2019-02-10T19:12:51+09:00"
+++

[先日の記事](/post/container-network-interface/)でも軽くふれた [Multus CNI plugin](https://github.com/intel/multus-cni) を実際に使ってみた。

（盛んに開発されているため, 執筆時から内容が変わっている可能性があるので注意）

## Multus CNI について

* CNI plugin を複数呼び出すことで, container に複数のNetwork interfaceを与えるCNI plugin
* Kubernetes と連携し, Pod の Manifest に応じて利用するNetworkを切り替えたりも可能

### Kubernetes連携時の動作の仕組み

* 各ノードのCNI設定ファイル(デフォルト: `/etc/cni/net.d/`)を設定し, Multusが呼び出されるようにする
    * 現時点では`DaemonSet`として作成したPodを使ってノードに対するCNI設定ファイル配置を実現している
        * `nfvpe/multus` というコンテナイメージに, 設定ファイルを配置するべきホストのディレクトリを
      volumeとしてマウントさせる
        * 設定内容はコンテナ内のファイルにあるので必要に応じてdaemon用コンテナに`ConfigMap`を
      volumeマウントさせる
    * 半年ほど前に触ったときは, ホスト側の設定(`/etc/cni/net.d/`)を直接編集する手順が案内されていた
    (と思う)ので, また変わるかもしれない
        * 軽く見た限り, Calicoやflannelも同じようなことをしていそう
* 各ノードにMultusを含む利用したいCNI pluginバイナリを配置しておき, KubeletがMultusを呼び出し,
  Multusが1つ以上のCNI pluginを呼び出す
    * Multus CNI pluginのバイナリ配置については前述の `DaemonSet` がやってくれる
* Podには必ず1つ共通のネットワーク (="Default network") が `eth0` として設定される
    * おそらくKubernetesのPod Networkとしての要件を満たすため
* Custom Resource Definition (CRD) という Kubenetes が提供しているAPI拡張の仕組みを利用し,
  Default network以外のネットワークを定義し, PodはManifestにてこれを指定する
* Multus CNI pluginはそれ自体がKubernetes APIを呼び出し, CRDを参照してネットワークに対して
  どのCNI pluginをどういうconfigで呼び出すかを決定する

## セットアップ

ここでは, `kubeadm` のインストールが完了し, Kubenetesクラスタ構築直前の状態からの構築を想定する。
([Creating a single master cluster with kubeadm](https://kubernetes.io/docs/setup/independent/create-cluster-kubeadm/)の, "Installing a pod network add-on" の直前の状態)
今回, Default networkには[Project Calico](https://www.projectcalico.org/)を利用してみる。

基本的には [Quickstart Guide](https://github.com/intel/multus-cni/blob/master/doc/quickstart.md)
([執筆時のバージョン](https://github.com/intel/multus-cni/blob/515e7eb92c14e014a8d239f511591d8eaea5f245/doc/quickstart.md))
に従って進める。

### Master node 構築

* 通常通り `kubeadm init` により構築するが,
  後述の理由により `--pod-network-cidr=172.18.0.0/16` を指定する

      ```
      qwert@k8s-master-1:~$ sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
      (略)
      Your Kubernetes master has initialized successfully!
      (略)
      ```
      ```
      qwert@k8s-master-1:~$ mkdir -p $HOME/.kube
      qwert@k8s-master-1:~$ sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
      qwert@k8s-master-1:~$ sudo chown $(id -u):$(id -g) $HOME/.kube/config
      ```

### DaemonSetの設定

* 執筆時点ではCalico用の設定例は[このIssue](https://github.com/intel/multus-cni/issues/227)で
  議論されている段階のため, その中で提案されている例を借用することにする
    * [Multus関連のManifest](https://raw.githubusercontent.com/project-azorian/poc-multus-calico/c5beb3139fdde2182a5c14554a719144311c05b2/manifests/calico/multus.yaml)を用いて
    Multusに必要なCRD, RBAC, DaemonSetを設定

        ```
        qwert@k8s-master-1:~$ kubectl apply -f https://raw.githubusercontent.com/project-azorian/poc-multus-calico/c5beb3139fdde2182a5c14554a719144311c05b2/manifests/calico/multus.yaml
        customresourcedefinition.apiextensions.k8s.io/network-attachment-definitions.k8s.cni.cncf.io created
        clusterrole.rbac.authorization.k8s.io/multus created
        clusterrolebinding.rbac.authorization.k8s.io/multus created
        serviceaccount/multus created
        configmap/multus-cni-config created
        daemonset.extensions/kube-multus-ds-amd64 created
        ```
    * [Calico関連のManifest](https://raw.githubusercontent.com/project-azorian/poc-multus-calico/c5beb3139fdde2182a5c14554a719144311c05b2/manifests/calico/calico.yaml)を用いて
    Calicoに必要なRBAC, DaemonSetを設定

        ```
        qwert@k8s-master-1:~$ kubectl apply -f https://raw.githubusercontent.com/project-azorian/poc-multus-calico/c5beb3139fdde2182a5c14554a719144311c05b2/manifests/calico/calico.yaml
        configmap/calico-config created
        service/calico-typha created
        deployment.apps/calico-typha created
        poddisruptionbudget.policy/calico-typha created
        daemonset.extensions/calico-node created
        serviceaccount/calico-node created
        customresourcedefinition.apiextensions.k8s.io/felixconfigurations.crd.projectcalico.org created
        customresourcedefinition.apiextensions.k8s.io/bgppeers.crd.projectcalico.org created
        customresourcedefinition.apiextensions.k8s.io/bgpconfigurations.crd.projectcalico.org created
        customresourcedefinition.apiextensions.k8s.io/ippools.crd.projectcalico.org created
        customresourcedefinition.apiextensions.k8s.io/hostendpoints.crd.projectcalico.org created
        customresourcedefinition.apiextensions.k8s.io/clusterinformations.crd.projectcalico.org created
        customresourcedefinition.apiextensions.k8s.io/globalnetworkpolicies.crd.projectcalico.org created
        customresourcedefinition.apiextensions.k8s.io/globalnetworksets.crd.projectcalico.org created
        customresourcedefinition.apiextensions.k8s.io/networkpolicies.crd.projectcalico.org created
        clusterrole.rbac.authorization.k8s.io/calico-node created
        clusterrolebinding.rbac.authorization.k8s.io/calico-node created
        ```
        * この YAML の中身をみるとわかるが, この中で `172.18.0.0/16` を指定しているため,
          `kubeadm init` 時に `--pod-network-cidr=172.18.0.0/16` を指定した

### Nodeの追加

* 通常通りNodeを追加 (`k8s-node-2` も同様に実施し, workload用のnodeを2つ追加した)

    ```
    qwert@k8s-node-1:~$ sudo kubeadm join 172.31.253.102:6443 --token XXXX --discovery-token-ca-cert-hash sha256:xxxxxx
    (略)
    This node has joined the cluster:
    (略)
    ```
* Nodeの追加を確認

    ```
    qwert@k8s-master-1:~$ kubectl get node
    NAME           STATUS   ROLES    AGE    VERSION
    k8s-master-1   Ready    master   29m    v1.13.2
    k8s-node-1     Ready    <none>   2m2s   v1.13.2
    k8s-node-2     Ready    <none>   34s    v1.13.2
    ```

### Pod作成

* まず, 追加ネットワークの指定をせずに(=Multus固有の設定無しに)Podを作成してみる

    ```
    qwert@k8s-master-1:~$ cat test-pod1.yaml
    ---
    apiVersion: v1
    kind: Pod
    metadata:
      name: test-pod1
    spec:
      containers:
        - name: alpine
          image: alpine:latest
          command: ["/usr/bin/tail"]
          args: ["-f", "/dev/null"]

    qwert@k8s-master-1:~$ kubectl apply -f test-pod1.yaml
    pod/test-pod1 created

    qwert@k8s-master-1:~$ kubectl exec -i -t test-pod1 /bin/sh
    / # ip a
    1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN qlen 1000
        link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
        inet 127.0.0.1/8 scope host lo
           valid_lft forever preferred_lft forever
    2: tunl0@NONE: <NOARP> mtu 1480 qdisc noop state DOWN qlen 1000
        link/ipip 0.0.0.0 brd 0.0.0.0
    4: eth0@if5: <BROADCAST,MULTICAST,UP,LOWER_UP,M-DOWN> mtu 1440 qdisc noqueue state UP
        link/ether ae:12:ce:ec:49:0c brd ff:ff:ff:ff:ff:ff
        inet 172.18.2.2/32 scope global eth0
           valid_lft forever preferred_lft forever
    ```
    * Calicoが用意したインターフェイスとアドレスが見えている
    * 外部からも実際に疎通する

        ```
        $ ping 172.18.2.2
        PING 172.18.2.2 (172.18.2.2) 56(84) bytes of data.
        64 bytes from 172.18.2.2: icmp_seq=1 ttl=63 time=0.495 ms
        64 bytes from 172.18.2.2: icmp_seq=2 ttl=63 time=0.374 ms
        64 bytes from 172.18.2.2: icmp_seq=3 ttl=63 time=0.355 ms
        ```
        * ここでは サブネット `172.18.2.0/24` に対し, それが割り当てられたnodeのnode IPに転送するよう
      ping実行元ホストにルートを追加して試している
* Multusが使用する, CNI pluginに指定するネットワークの設定を
  CRDである `NetworkAttachmentDefinition` として作成する

    ```
    qwert@k8s-master-1:~$ cat test-network.yaml
    apiVersion: "k8s.cni.cncf.io/v1"
    kind: NetworkAttachmentDefinition
    metadata:
      name: test-network
    spec:
      config: >
        {
            "cniVersion": "0.3.0",
            "type": "bridge",
            "bridge": "test-br",
            "ipam": {
                "type": "host-local",
                "subnet": "192.168.1.0/24",
                "rangeStart": "192.168.1.100",
                "rangeEnd": "192.168.1.200"
            }
        }
    qwert@k8s-master-1:~$ kubectl apply -f test-network.yaml
    networkattachmentdefinition.k8s.cni.cncf.io/test-network created
    ```
* 作成した `NetworkAttachmentDefinition` を利用するように
  Manifestの `annotation` に指定を追加して, Podを作成

    ```
    qwert@k8s-master-1:~$ cat test-pod2.yaml
    ---
    apiVersion: v1
    kind: Pod
    metadata:
      name: test-pod2
      annotations:
        k8s.v1.cni.cncf.io/networks: test-network
    spec:
      containers:
        - name: alpine
          image: alpine:latest
          command: ["/usr/bin/tail"]
          args: ["-f", "/dev/null"]
    qwert@k8s-master-1:~$ kubectl apply -f test-pod2.yaml
    pod/test-pod2 created
    qwert@k8s-master-1:~$ kubectl exec -i -t test-pod2 /bin/sh
    / # ip a
    1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN qlen 1000
        link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
        inet 127.0.0.1/8 scope host lo
           valid_lft forever preferred_lft forever
    2: tunl0@NONE: <NOARP> mtu 1480 qdisc noop state DOWN qlen 1000
        link/ipip 0.0.0.0 brd 0.0.0.0
    4: eth0@net1: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1440 qdisc noqueue state UP
        link/ether 4a:47:7d:a9:1f:dd brd ff:ff:ff:ff:ff:ff
        inet 172.18.2.3/32 scope global eth0
           valid_lft forever preferred_lft forever
    6: net1@if8: <BROADCAST,MULTICAST,UP,LOWER_UP,M-DOWN> mtu 1500 qdisc noqueue state UP
        link/ether 0a:58:c0:a8:01:64 brd ff:ff:ff:ff:ff:ff
        inet 192.168.1.100/24 scope global net1
           valid_lft forever preferred_lft forever
    ```
    * Calicoにより作成されるインターフェイスの他に, `net1` というインターフェイスが作成されている
    * Podが配置されたnodeを見ると, `bridge` CNI pluginの期待通りの動きをしているとわかる
        * bridgeインターフェイスが作成されている

            ```
            qwert@k8s-node-2:~$ ip link | grep test-br
            7: test-br: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue state UP mode DEFAULT group default qlen 1000
            8: vethc623a4a8@if6: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue master test-br state UP mode DEFAULT group default
            ```
        * netnsid = 1 で index = 6 のインターフェイスはbridgeインターフェイスにつながっている

            ```
            qwert@k8s-node-2:~$ brctl show
            bridge name	bridge id		STP enabled	interfaces
            docker0		8000.0242175d8c92	no		
            test-br		8000.ae9d5b977edb	no		vethc623a4a8

            qwert@k8s-node-2:~$ ip link show vethc623a4a8
            8: vethc623a4a8@if6: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue master test-br state UP mode DEFAULT group default 
                link/ether ae:9d:5b:97:7e:db brd ff:ff:ff:ff:ff:ff link-netnsid 1
            ```
        * 作成したPodがまさに netnsid = 1 であり, index = 6 のインターフェイスは `net1`

            ```
            qwert@k8s-node-2:~$ sudo docker ps | grep test-pod2
            6e8321489147        alpine                 "/usr/bin/tail -f /d…"   25 minutes ago      Up 25 minutes                           k8s_alpine_test-pod2_default_c6c519e0-2cb5-11e9-85f2-0050569dc4a8_0
            fb09adadbf12        k8s.gcr.io/pause:3.1   "/pause"                 25 minutes ago      Up 25 minutes                           k8s_POD_test-pod2_default_c6c519e0-2cb5-11e9-85f2-0050569dc4a8_0

            qwert@k8s-node-2:~$ sudo docker inspect 6e8321489147 | grep -w Pid
                        "Pid": 28483,

            qwert@k8s-node-2:~$ sudo ln -s /proc/28483/ns/net /var/run/netns/test-pod2

            qwert@k8s-node-2:~$ sudo ip netns list
            test-pod2 (id: 1)

            qwert@k8s-node-2:~$ sudo ip netns exec test-pod2 ip link
            1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN mode DEFAULT group default qlen 1000
                link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
            2: tunl0@NONE: <NOARP> mtu 1480 qdisc noop state DOWN mode DEFAULT group default qlen 1000
                link/ipip 0.0.0.0 brd 0.0.0.0
            4: eth0@if6: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1440 qdisc noqueue state UP mode DEFAULT group default
                link/ether 4a:47:7d:a9:1f:dd brd ff:ff:ff:ff:ff:ff link-netnsid 0
            6: net1@if8: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue state UP mode DEFAULT group default
                link/ether 0a:58:c0:a8:01:64 brd ff:ff:ff:ff:ff:ff link-netnsid 0
            ```

## 所感

* Multusを使えば簡単に複数NICを持つPodを作れる
  * セットアップしたら, `NetworkAttachmentDefinition` としてネットワーク設定を作成して,
    `Pod` に対し利用したいネットワークを `annotation` として指定するだけ
* セットアップもだいぶ簡単になっているし, 設定例等ドキュメント関連の整備も進んできており,
  ますます手を出しやすくなりそう
* 今回はお試しで `bridge` pluginを使ったが, 使い途は色々ありそう
    * `vlan` plugin等を使って物理環境とLayer2で接続できるPodを作るとか...
    * ソフトウェアルーターをPodとして作るとか...
    * ホストの物理NICを直接Podに使わせたい場合とかにも便利なのかもしれない

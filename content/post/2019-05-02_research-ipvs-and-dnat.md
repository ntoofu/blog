+++
title = "IPVS (LVS/NAT) とiptables DNAT targetの共存について調べた"
date = "2019-05-02T08:53:00+09:00"
draft = false
+++

# TL;DR

* IPVSはデフォルトで`conntrack`(標準のconnection tracking)を利用しない
* そのせいでIPVSに処理させるパケットをNATしたりすると期待通り動かないことがある
* 必要があれば `/proc/sys/net/ipv4/vs/conntrack` (`net.ipv4.vs.conntrack`) を設定しよう

# 背景

* iptablesの `DNAT` targetと IPVS(LVS/NAT構成)を共存させると上手く行かない
    * PREROUTING chainにおいて `REDIRECT` target によりポート番号をIPVSのvirtual serviceのものに変換するように設定した場合, REDIRECTされるポートに対し接続すると, 戻り通信(SYNに対するSYN+ACK)の送信元ポートがvirtual serviceのものになる（期待されるのは, 変換前のもの)
    * (PREROUTING chainの) `REDIRECT` targetは内部的には `DNAT` と同等の動きをするようで, `DNAT` でも起きる
* 以下のようにすれば再現する
    * テスト用ホストでnetcatなどで適当にListenしてくれるポートを2つ作り, IPVSでバランシング設定

        ```
        (while true;do nc -l -p 1001; echo 1001; done)&
        (while true;do nc -l -p 1002; echo 1002; done)&
        ```
        ```
        ipvsadm -A -t 172.16.1.8:1000 -s rr
        ipvsadm -a -t 172.16.1.8:1000 -r 172.16.1.8:1001 -m
        ipvsadm -a -t 172.16.1.8:1000 -r 172.16.1.8:1002 -m
        ```
    * 別ホストから接続を確認 (期待通り2つのnetcatプロセスに分散されて動いている)

        ```
        echo hoge | nc -N 172.16.1.8 1000
        echo hoge | nc -N 172.16.1.8 1000
        ```
        * テストホストでの出力

            ```
            hoge
            1001
            hoge
            1002
            ```
    * テスト用ホストでREDIRECTターゲットを設定

        ```
        iptables -t nat -A PREROUTING -p tcp --dport 1010 -j REDIRECT --to-port 1000
        ```
    * 別ホストから接続を確認 (__接続できず, tcpdumpで見ると戻りパケットの送信元ポート番号がおかしい__)

        ```
        echo hoge | nc -N 172.16.1.8 1000
        ```
        ```
        23:01:43.223963 IP 172.16.1.2.54996 > 172.16.1.8.1010: Flags [S], seq 3333715063, win 29200, options [mss 1460,sackOK,TS val 535608746 ecr 0,nop,wscale 7], length 0
        23:01:43.224307 IP 172.16.1.8.1000 > 172.16.1.2.54996: Flags [S.], seq 3354407097, ack 3333715064, win 28960, options [mss 1460,sackOK,TS val 128663562 ecr 535608746,nop,wscale 7], length 0
        23:01:43.224330 IP 172.16.1.2.54996 > 172.16.1.8.1000: Flags [R], seq 3333715064, win 0, length 0
        ```
    * テスト用ホストでDNATターゲットを設定しても, 同様

        ```
        iptables -t nat -A PREROUTING -p tcp --dport 1020 -j DNAT --to-destination :1000
        ```
    * ( Linux kernel 4.14.83 で確認 )


# 調べたこと

netfilterの動きについて, ソースコードを中心に確認し, 前述の事象が起きる理由を調査した。

## 前提知識

* iptablesの操作やtable, chain, targetなどの概念は知っている前提
* 仕組みの概要は [A Deep Dive into Iptables and Netfilter Architecture](https://www.digitalocean.com/community/tutorials/a-deep-dive-into-iptables-and-netfilter-architecture) がiptablesとnetfilterの関係を含め説明している
    * Linux kernelのネットワーク処理の中のhookを提供する枠組み(=netfilter)を利用したfirewall softwareがiptables
    * netfilterの用意するhookは何種類かあり, それに対応するのがiptablesのchain(の中の初期から存在するもの)
    * chainに紐づくtableは複数存在しうるが, 所定の順で呼び出される
* IPVS も netfilter を利用して作られている

## ソースコードリーディング

* 取っ掛かりとして REDIRECT target について調べる
    * [`/net/netfilter/xt_REDIRECT.c`](https://elixir.bootlin.com/linux/v4.14.83/source/net/netfilter/xt_REDIRECT.c) が実装で,
      ぱっと見 [`redirect_tg_reg`](https://elixir.bootlin.com/linux/v4.14.83/source/net/netfilter/xt_REDIRECT.c#L74) で target の定義をしている
        * `redirect_tg_init` がmodule初期化時(多分)に呼び出され, `xt_register_target` を介して [AddressFamilyごとのtargetのリスト](https://elixir.bootlin.com/linux/v4.14.83/source/net/netfilter/x_tables.c#L56) に追加されていく
* targetを呼び出す箇所が気になるので, パケットに対し各ルールを評価する場所を探す
    * __[`ipt_do_table`](https://elixir.bootlin.com/linux/v4.14.83/source/net/ipv4/netfilter/ip_tables.c#L227) がそれ__
        * packet(`skb`) に対し, 現在評価対象のhookに関するtable内のルール(`ipt_entry`がルールを表現)を順番にチェックしている様子
        * [ここ](https://elixir.bootlin.com/linux/v4.14.83/source/net/ipv4/netfilter/ip_tables.c#L291)で match (プロトコルを解釈した上でのマッチ条件) によらないIPヘッダだけでのマッチ
        * [ここ](https://elixir.bootlin.com/linux/v4.14.83/source/net/ipv4/netfilter/ip_tables.c#L298)で match ごとの評価
        * [ここ](https://elixir.bootlin.com/linux/v4.14.83/source/net/ipv4/netfilter/ip_tables.c#L308)からtargetの実行
    * `ipt_do_table` は最終的には [`nf_hook_ops`構造体](https://elixir.bootlin.com/linux/v4.14.83/source/include/linux/netfilter.h#L64) としてhookに登録される
        * `ipt_do_table` は [`iptable_filter_hook` などに呼び出され](https://elixir.bootlin.com/linux/v4.14.83/source/net/ipv4/netfilter/iptable_filter.c#L47)ている
        * `iptable_filter_hook` などは [ここ](https://elixir.bootlin.com/linux/v4.14.83/source/net/ipv4/netfilter/iptable_filter.c#L102) のように
          初期化時に [`xt_hook_ops_alloc`](https://elixir.bootlin.com/linux/v4.14.83/source/net/netfilter/x_tables.c#L1614) により
          [`nf_hook_ops`構造体](https://elixir.bootlin.com/linux/v4.14.83/source/include/linux/netfilter.h#L64) に詰められ
          [`ipt_register_table` によって登録される](https://elixir.bootlin.com/linux/v4.14.83/source/net/ipv4/netfilter/iptable_filter.c#L71)
        * [`ipt_regsiter_table`](https://elixir.bootlin.com/linux/v4.14.83/source/net/ipv4/netfilter/ip_tables.c#L1787)
          -> [`nf_register_net_hooks`](https://elixir.bootlin.com/linux/v4.14.83/source/net/netfilter/core.c#L382)
          -> [`nf_register_net_hook`](https://elixir.bootlin.com/linux/v4.14.83/source/net/netfilter/core.c#L277)
          とたどり, [`nf_hook_entries_grow`](https://elixir.bootlin.com/linux/v4.14.83/source/net/netfilter/core.c#L101) でhook関数のリストに追加される
        * 登録時には[priorityを見て](https://elixir.bootlin.com/linux/v4.14.83/source/net/netfilter/core.c#L138)hook関数の順序を決定しており,
          元は `xt_table` として設定された `.priority` が `nf_hook_ops` の `.priority` に引き継がれている [`xt_proto_init`](https://elixir.bootlin.com/linux/v4.14.83/source/net/netfilter/x_tables.c#L1635)
            * 大元になる priority は [このへん](https://elixir.bootlin.com/linux/v4.14.83/source/include/uapi/linux/netfilter_ipv4.h#L58) であり,
              [A Deep Dive into Iptables and Netfilter Architecture](https://www.digitalocean.com/community/tutorials/a-deep-dive-into-iptables-and-netfilter-architecture) の表に合致する
* IPVSについては, [ここ](https://elixir.bootlin.com/linux/v4.14.83/source/net/netfilter/ipvs/ip_vs_core.c)を中心に実装がある
    * 受け付ける通信は[SNATより1つ高い優先度](https://elixir.bootlin.com/linux/v4.14.83/source/net/netfilter/ipvs/ip_vs_core.c#L2126)で,
      戻りの通信は[DNATより1つ低い優先度](https://elixir.bootlin.com/linux/v4.14.83/source/net/netfilter/ipvs/ip_vs_core.c#L2133)で処理される
* 気になっている事象は, 戻りの通信自体はあるがsource portがおかしく, 戻り通信の処理がおかしいように感じる
* `DNAT` や `REDIRECT` については PREROUTING chain にしかルールを書かなくても通常は戻りの通信も上手く行くが, これは`conntrack`によるconnection trackingのおかげ
    * `conntrack` も [netfilterを用いて実装されている](https://elixir.bootlin.com/linux/v4.14.83/source/net/ipv4/netfilter/nf_conntrack_l3proto_ipv4.c#L181)
    * `conntrack` の情報は [skbに紐づけて管理される](https://elixir.bootlin.com/linux/v4.14.83/source/include/linux/skbuff.h#L701)

## 仮説: conntrack と IPVS の評価順序の問題

* 仮説: 評価順序からすると, ingress traffic も egress traffic も `conntrack` -> IPVS の順序であり,
  IPVSによるpacket書き換えの影響で egress traffic の connection tracking がマッチしないのではないか
* IPVSによるoutput通信の書き換えの優先度を上げて実験
    * [ここ](https://elixir.bootlin.com/linux/v4.14.83/source/net/netfilter/ipvs/ip_vs_core.c#L2133)について,
      `NF_IP_PRI_CONNTRACK - 1` としてkernelをビルドし直す
* 結果: 変化なし
    * 事象は全く変わりなく発生した
    * `conntrack -E` コマンドで `conntrack` について確認してみると,
      そもそも IPVSのvirtual serviceへの通信は `conntrack` 情報が乗ってこない事がわかる

## さらに調査

* そもそも __IPVSは独自にconnection trackingを実装していて, 標準の`conntrack`を利用しない__
    * [ここ](http://kb.linuxvirtualserver.org/wiki/Performance_and_Tuning#Netfilter_Connection_Track)などで触れられている
    * __`conntrack`の情報は削除している__
        * [ここ](https://elixir.bootlin.com/linux/v4.14.83/source/net/netfilter/ipvs/ip_vs_conn.c#L514) で設定された `ip_vs_nat_xmit` が
          [ここ](https://elixir.bootlin.com/linux/v4.14.83/source/net/netfilter/ipvs/ip_vs_core.c#L1983) で呼び出され,
          [`ip_vs_nat_send_or_cont` を経由](https://elixir.bootlin.com/linux/v4.14.83/source/net/netfilter/ipvs/ip_vs_xmit.c#L810)して
          `IP_VS_CONN_F_NFCT` フラグが立っていなければ [`ip_vs_notrack`](https://elixir.bootlin.com/linux/v4.14.83/source/net/netfilter/ipvs/ip_vs_xmit.c#L615) により
          `conntrack` の情報を削除される
* 先に触れていたように, __`conntrack` の情報は `skb` に紐付けられており__, `DNAT` `REDIRECT` がどう動いていても
  情報が消されてしまっては戻りの通信を正しく処理できるはずがない
    * IPVS独自実装のconnection trackingだけが働くため, IPVSのvirtual serviceのportが送信元になる
* しかし, よく見ると [ここ](https://elixir.bootlin.com/linux/v4.14.83/source/net/netfilter/ipvs/ip_vs_conn.c#L989) が
  真であれば `IP_VS_CONN_F_NFCT`フラグが立ち, `conntrack`の情報削除を免れる
    * 直前のコメント文にある, "`conntrack` を保持することが有用なケース" がまさに今回のはず
    * `ip_vs_conntrack_enabled`が真になるのはたどっていくとsysctlで [`net/ipv4/vs`配下](https://elixir.bootlin.com/linux/v4.14.83/source/net/netfilter/ipvs/ip_vs_ctl.c#L3978) の [`conntrack`](https://elixir.bootlin.com/linux/v4.14.83/source/net/netfilter/ipvs/ip_vs_ctl.c#L1752)

## 再実験

* テストホストで見つけた設定を試す

    ```
    sysctl net.ipv4.vs.conntrack=1
    ```
* 別のホストから `REDIRECT` tagetが働くようにアクセス

    ```
    echo hoge | nc -N 172.16.1.8 1000
    echo hoge | nc -N 172.16.1.8 1000
    ```
    * テストホストでの出力

        ```
        hoge
        1001
        hoge
        1002
        ```

無事解決。副作用が無いかについてはまだ調べられていない。

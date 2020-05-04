+++
title = "VMware ESXi 仮想スイッチのreverse teamingについて"
+++

VMware ESXi の仮想スイッチの挙動に関して同僚に質問され調べたことがあったので、軽くまとめておく。

# TL;DR

* VMware ESXi の仮想スイッチではreverse teamingと呼ばれるフィルタが機能することがある
* たとえpromiscuous modeを有効にしていても機能するため、ミラーポート代わりにしたい時は注意が必要

# 前提

## 注意

この記事の内容はESXi 6.7に対して確認しており、それ以外については確認していない。

## NICチーミングについて

詳細は[ドキュメント](https://docs.vmware.com/jp/VMware-vSphere/6.7/com.vmware.vsphere.networking.doc/GUID-4D97C749-1FFD-403D-B2AE-0CD0F1C70E2B.html)にあるので、概要についてのみ記述する。

仮想マシンがESXi外部と通信するために、ESXiの仮想スイッチに対してESXiホストが持つNICを割り当てる必要がある。（仮想スイッチのアップリンクと呼ばれる。）
冗長化や負荷分散のために複数のNICをこのアップリンクとして指定することができ、これをNICチーミングという。

イメージとしてはリンクアグリゲーションが近いし、実際に対向となるスイッチでLAGの設定をしなければならない構成もあるが、LAGを設定しないでも、NICチーミングを利用できる方法も用意されている。
チーミングでは複数NICにどのように通信を分散させるかを規定したロードバランシングポリシーを選択でき、送信元の仮想マシン（正確には仮想スイッチのポート）ごとに利用する物理NICを決定したり、送信元MACアドレスから決定する方法などを選択した場合は、対向スイッチでLAGを設定する必要がない。
（むしろ、設定してはいけない。）

# reverse teaming

以前トラブルシューティング時に存在に気づいたが、あまり言及されるのをみないreverse teamingと呼ばれるらしい機能が仮想スイッチには存在している。これはどうやら前述のNICチーミングにおいてLAGを必要としないケースで動作するもののようだ。

スイッチにLAGが設定されなければ、BUMトラフィックについてはフラッディングするため、仮想スイッチの複数のアップリンクに複製された同一のフレームが到達することとなる。したがって、宛先MACアドレスから転送先の仮想マシンを単純に決定するだけでは、複製されたフレームを仮想マシンが受け取ることとなる。それを防ぐため、上述のロードバランシングポリシーが選択されている場合は、仮想マシンがフレーム送信時に利用しているアップリンクで受け取ったパケットしか仮想マシンに転送しないようフィルタリングがなされる。これがreverse teamingと呼ばれるものである。

送信時に利用しているアップリンクの対向ポートにて、仮想マシンのMACアドレスが学習されているはずなので、仮想マシン宛のユニキャスト通信は送信時に利用しているアップリンクで受け取ることになるという想定で、そのような作りになっているのだと思われる。

この機能自体はよく考えられているが、細かい部分でトラブルシューティング時などに影響してくるので、気をつける必要もある。


## ESXiでのパケットキャプチャ時の注意

ESXiでは `pktcap-uw` コマンドによりパケットキャプチャが可能で、アップリンクや仮想マシンが接続される仮想スイッチのポートなどを指定してキャプチャを取得できる。
この時、キャプチャポイントを `--capture` オプションにより指定できるが、この指定によってreverse teamingが作用した後の結果が取れるかそうでないかが変わってくるようだ。

具体的には、 `--capture PortOutput` では、reverse teamingが働く前の結果が取得される。
仮想スイッチのポートを指定してキャプチャした際、特にキャプチャポイントを `--capture` で指定しなかった場合は、デフォルトでこの状態になると思われる。
明示的に `--capture VnicRx` を指定した場合は、reverse teamingを反映した結果となる。

## promiscuous mode と reverse teaming

### promiscuous mode

通常仮想スイッチはMACアドレスの学習はしないものの、仮想マシンに割り当てているMACアドレスを元に仮想マシンを特定し、仮想スイッチが受け取ったパケットの転送を行う。（イメージとしては、MACテーブルに静的にエントリが指定されており、動的な変更がないような状態での動きに近いかもしれない。）そのため、同一ポートグループを利用していたとしても他の仮想マシンへのユニキャスト通信を受け取ることは基本的に無い。

しかし、promiscuous mode(無差別モード)が有効な場合は、そのポートグループ内の全仮想マシンにパケットが転送される。（イメージとしては、ハブの動作に近いかもしれない。）仮想マシンがL2VPNの終端になっている場合や、再仮想化などの都合で、仮想NICに割り当てられているアドレス以外のアドレスを受け取る必要がある場合などに、よく利用される。

ただし、ESXi6.7からは分散仮想スイッチにてMACアドレス学習させるオプションが出来ているので、必要性は薄れているかもしれない。

### reverse teaming の影響

検証したところでは、promiscuous modeであるかどうかに関わらず、reverse teamingは同じように動作するらしい。
しかしこの事がかえって複雑な挙動につながることがある。

promiscuous modeでは、そのポートグループ上で仮想マシンがやり取りした通信は、同一ポートグループ上の他の仮想マシンにも見えてしまうことがある。
同一ESXiホスト上であればポートグループ内の全仮想マシンに転送するため、それ自体は自然なことである。

しかしもう少し詳しく見ると、他の仮想マシンの通信が見えたり見えなかったりの条件が非常に複雑であることがわかる。

ESXiホストを跨ぐ場合はそれらを結ぶ物理スイッチの挙動にも依存するが、通常は物理スイッチ側はpromiscuous modeのような動作はしないだろうから、送信元と宛先のどちらも自身と同じホスト上の仮想マシンではないユニキャスト通信は、フラッディングされない限り基本的に見ることが出来ない。
これは、仮想スイッチの挙動以前に物理スイッチがESXiまで転送してこないので当然の話ではある。
同一ホスト上の仮想マシンが送信元である通信については、（promiscuous有効の同一ポートグループを利用している時）自身が宛先でなくとも転送されてくるため見ることが出来る。

ややこしいのは、以上に当てはまらない、同一ホスト上の仮想マシンが宛先で、送信元がホスト外であるような通信についてである。
基本的にはpromiscuous modeの動作にしたがい、通信を見ることが出来るはずなのだが、この場合もreverse teamingは健在であるため、この通信を受け取るアップリンクが自身が送信時に利用するものでないと、破棄されてしまい見ることが出来ない。
例えばロードバランシングポリシーが送信元ポートによるものであれば、アップリンクの割り当ては仮想マシンの再起動などでも変化することがあるぐらいなので、同じホスト上の別仮想マシンへの通信は見えたり見えなかったりする…というのが実情になる。

|送信元|送信先|見えるかどうか|
|---|---|---|
|同じホスト上の仮想マシン |同じホスト上の仮想マシン|見える|
|同じホスト上の仮想マシン |ホスト外|見える|
|ホスト外 |同じホスト上の仮想マシン|アップリンクが同じ時だけ見える|
|ホスト外 |ホスト外|見えない|

例えば、ある仮想マシンへの通信を別の仮想マシンでキャプチャするために、promiscuous modeが利用できるのではないかと考えた場合は、このような事情も加味しないといけないので気をつけなければならない。

# 検証の記録

## 構成

* ESXi 6.7 (2台)
* 標準仮想スイッチを利用
    * アップリンクは2つ用意
    * ロードバランシングポリシーは送信元ポートを利用
* それぞれのESXiホストに1台ずつテスト用仮想マシンを配置し、pingによりICMPパケットをやり取りさせておく
* 仮想マシンに割り当てられなかった方のアップリンクについて、対向の物理スイッチのポートをミラーポートとして設定し、両アップリンクに同じフレームが流れ込むように仕向ける

## 手順と結果

* アップリンクに同じようにICMPパケットが来ているか確認（同一のタイミングで取得していないので完全には一致しないが、同じ送信元と宛先で1秒間隔に受信している）

    ```
    [root@esxi:~] pktcap-uw --uplink vmnic2 -c 3 --proto 0x01
    The name of the uplink is vmnic2
    To capture 3 packets
    The session filter IP protocol is 0x01
    No server port specifed, select 65440 as the port
    Output the packet info to console.
    Local CID 2
    Listen on port 65440
    Accept...Vsock connection from port 1062 cid 2
    16:13:16.471308[1] Captured at EtherswitchDispath point, TSO not enabled, Checksum not offloaded and not verified, VLAN tag 100, length 98.
    Segment[0] ---- 98 bytes:
    0x0000:  0050 569d fdb1 0050 569d 4342 0800 4500 
    0x0010:  0054 15e5 0000 4001 1b5c c0a8 640b c0a8 
    0x0020:  640c 0000 b9f4 0d1d 019f 9ced ae5e 0000 
    0x0030:  0000 2630 0700 0000 0000 1011 1213 1415 
    0x0040:  1617 1819 1a1b 1c1d 1e1f 2021 2223 2425 
    0x0050:  2627 2829 2a2b 2c2d 2e2f 3031 3233 3435 
    0x0060:  3637 
    16:13:17.495309[2] Captured at EtherswitchDispath point, TSO not enabled, Checksum not offloaded and not verified, VLAN tag 100, length 98.
    Segment[0] ---- 98 bytes:
    0x0000:  0050 569d fdb1 0050 569d 4342 0800 4500 
    0x0010:  0054 1632 0000 4001 1b0f c0a8 640b c0a8 
    0x0020:  640c 0000 f995 0d1d 01a0 9ded ae5e 0000 
    0x0030:  0000 e58d 0700 0000 0000 1011 1213 1415 
    0x0040:  1617 1819 1a1b 1c1d 1e1f 2021 2223 2425 
    0x0050:  2627 2829 2a2b 2c2d 2e2f 3031 3233 3435 
    0x0060:  3637 
    Receive thread exiting... 
    16:13:18.519323[3] Captured at EtherswitchDispath point, TSO not enabled, Checksum not offloaded and not verified, VLAN tag 100, length 98.
    Segment[0] ---- 98 bytes:
    0x0000:  0050 569d fdb1 0050 569d 4342 0800 4500 
    0x0010:  0054 170b 0000 4001 1a36 c0a8 640b c0a8 
    0x0020:  640c 0000 2c37 0d1d 01a1 9eed ae5e 0000 
    0x0030:  0000 b1eb 0700 0000 0000 1011 1213 1415 
    0x0040:  1617 1819 1a1b 1c1d 1e1f 2021 2223 2425 
    0x0050:  2627 2829 2a2b 2c2d 2e2f 3031 3233 3435 
    0x0060:  3637 
    Dump thread exiting... 
    Destroying session 38 

    Dumped 3 packet to console, dropped 0 packets.
    Done.
    ```

    ```
    [root@esxi:~] pktcap-uw --uplink vmnic3 -c 3 --proto 0x01
    The name of the uplink is vmnic3
    To capture 3 packets
    The session filter IP protocol is 0x01
    No server port specifed, select 65444 as the port
    Output the packet info to console.
    Local CID 2
    Listen on port 65444
    Accept...Vsock connection from port 1063 cid 2
    16:13:26.711366[1] Captured at EtherswitchDispath point, TSO not enabled, Checksum not offloaded and not verified, VLAN tag 100, length 98.
    Segment[0] ---- 98 bytes:
    0x0000:  0050 569d fdb1 0050 569d 4342 0800 4500 
    0x0010:  0054 19df 0000 4001 1762 c0a8 640b c0a8 
    0x0020:  640c 0000 2241 0d1d 01a9 a6ed ae5e 0000 
    0x0030:  0000 b0d9 0a00 0000 0000 1011 1213 1415 
    0x0040:  1617 1819 1a1b 1c1d 1e1f 2021 2223 2425 
    0x0050:  2627 2829 2a2b 2c2d 2e2f 3031 3233 3435 
    0x0060:  3637 
    16:13:27.735507[2] Captured at EtherswitchDispath point, TSO not enabled, Checksum not offloaded and not verified, VLAN tag 100, length 98.
    Segment[0] ---- 98 bytes:
    0x0000:  0050 569d fdb1 0050 569d 4342 0800 4500 
    0x0010:  0054 1a35 0000 4001 170c c0a8 640b c0a8 
    0x0020:  640c 0000 4ce2 0d1d 01aa a7ed ae5e 0000 
    0x0030:  0000 8437 0b00 0000 0000 1011 1213 1415 
    0x0040:  1617 1819 1a1b 1c1d 1e1f 2021 2223 2425 
    0x0050:  2627 2829 2a2b 2c2d 2e2f 3031 3233 3435 
    0x0060:  3637 
    Receive thread exiting... 
    16:13:28.759426[3] Captured at EtherswitchDispath point, TSO not enabled, Checksum not offloaded and not verified, VLAN tag 100, length 98.
    Segment[0] ---- 98 bytes:
    0x0000:  0050 569d fdb1 0050 569d 4342 0800 4500 
    0x0010:  0054 1b25 0000 4001 161c c0a8 640b c0a8 
    0x0020:  640c 0000 9f83 0d1d 01ab a8ed ae5e 0000 
    0x0030:  0000 3095 0b00 0000 0000 1011 1213 1415 
    0x0040:  1617 1819 1a1b 1c1d 1e1f 2021 2223 2425 
    0x0050:  2627 2829 2a2b 2c2d 2e2f 3031 3233 3435 
    0x0060:  3637 
    Dump thread exiting... 
    Destroying session 39 

    Dumped 3 packet to console, dropped 0 packets.
    Done.
    ```

* 仮想マシンの接続するポートのPortOutputを確認し、複製を確認（1秒の間をあけず同じパケットが来ている）

    ```
    [root@esxi:~] pktcap-uw --switchport 50331657 --dir 1 -c 3 --proto 0x01 --capture PortOutput
    The switch port id is 0x03000009
    The dir is Output
    To capture 3 packets
    The session filter IP protocol is 0x01
    The session capture point is PortOutput
    No server port specifed, select 65434 as the port
    Output the packet info to console.
    Local CID 2
    Listen on port 65434
    Accept...Vsock connection from port 1060 cid 2
    16:12:40.631403[1] Captured at PortOutput point, TSO not enabled, Checksum not offloaded and not verified, VLAN tag 100, length 98.
    Segment[0] ---- 98 bytes:
    0x0000:  0050 569d fdb1 0050 569d 4342 0800 4500 
    0x0010:  0054 06e1 0000 4001 2a60 c0a8 640b c0a8 
    0x0020:  640c 0000 d1a6 0d1d 017c 78ed ae5e 0000 
    0x0030:  0000 30a1 0900 0000 0000 1011 1213 1415 
    0x0040:  1617 1819 1a1b 1c1d 1e1f 2021 2223 2425 
    0x0050:  2627 2829 2a2b 2c2d 2e2f 3031 3233 3435 
    0x0060:  3637 
    16:12:40.631419[2] Captured at PortOutput point, TSO not enabled, Checksum not offloaded and not verified, VLAN tag 100, length 98.
    Segment[0] ---- 98 bytes:
    0x0000:  0050 569d fdb1 0050 569d 4342 0800 4500 
    0x0010:  0054 06e1 0000 4001 2a60 c0a8 640b c0a8 
    0x0020:  640c 0000 d1a6 0d1d 017c 78ed ae5e 0000 
    0x0030:  0000 30a1 0900 0000 0000 1011 1213 1415 
    0x0040:  1617 1819 1a1b 1c1d 1e1f 2021 2223 2425 
    0x0050:  2627 2829 2a2b 2c2d 2e2f 3031 3233 3435 
    0x0060:  3637 
    Receive thread exiting... 
    16:12:41.655419[3] Captured at PortOutput point, TSO not enabled, Checksum not offloaded and not verified, VLAN tag 100, length 98.
    Segment[0] ---- 98 bytes:
    0x0000:  0050 569d fdb1 0050 569d 4342 0800 4500 
    0x0010:  0054 06f5 0000 4001 2a4c c0a8 640b c0a8 
    0x0020:  640c 0000 0348 0d1d 017d 79ed ae5e 0000 
    0x0030:  0000 fdfe 0900 0000 0000 1011 1213 1415 
    0x0040:  1617 1819 1a1b 1c1d 1e1f 2021 2223 2425 
    0x0050:  2627 2829 2a2b 2c2d 2e2f 3031 3233 3435 
    0x0060:  3637 
    Dump thread exiting... 
    Destroying session 36 

    Dumped 3 packet to console, dropped 0 packets.
    Done.
    ```

* 仮想マシンの接続するポートのVnicRxを確認し、複製がないことを確認

    ```
    [root@esxi:~] pktcap-uw --switchport 50331657 --dir 1 -c 3 --proto 0x01 --capture VnicRx
    The switch port id is 0x03000009
    The dir is Output
    To capture 3 packets
    The session filter IP protocol is 0x01
    The session capture point is VnicRx
    No server port specifed, select 65437 as the port
    Output the packet info to console.
    Local CID 2
    Listen on port 65437
    Accept...Vsock connection from port 1061 cid 2
    16:12:53.943341[1] Captured at VnicRx point, TSO not enabled, Checksum not offloaded and not verified, length 98.
    Segment[0] ---- 98 bytes:
    0x0000:  0050 569d fdb1 0050 569d 4342 0800 4500 
    0x0010:  0054 0b67 0000 4001 25da c0a8 640b c0a8 
    0x0020:  640c 0000 f4d6 0d1d 0189 85ed ae5e 0000 
    0x0030:  0000 fb63 0e00 0000 0000 1011 1213 1415 
    0x0040:  1617 1819 1a1b 1c1d 1e1f 2021 2223 2425 
    0x0050:  2627 2829 2a2b 2c2d 2e2f 3031 3233 3435 
    0x0060:  3637 
    16:12:54.967365[2] Captured at VnicRx point, TSO not enabled, Checksum not offloaded and not verified, length 98.
    Segment[0] ---- 98 bytes:
    0x0000:  0050 569d fdb1 0050 569d 4342 0800 4500 
    0x0010:  0054 0c2e 0000 4001 2513 c0a8 640b c0a8 
    0x0020:  640c 0000 3d78 0d1d 018a 86ed ae5e 0000 
    0x0030:  0000 b1c1 0e00 0000 0000 1011 1213 1415 
    0x0040:  1617 1819 1a1b 1c1d 1e1f 2021 2223 2425 
    0x0050:  2627 2829 2a2b 2c2d 2e2f 3031 3233 3435 
    0x0060:  3637 
    Receive thread exiting... 
    16:12:55.991379[3] Captured at VnicRx point, TSO not enabled, Checksum not offloaded and not verified, length 98.
    Segment[0] ---- 98 bytes:
    0x0000:  0050 569d fdb1 0050 569d 4342 0800 4500 
    0x0010:  0054 0c60 0000 4001 24e1 c0a8 640b c0a8 
    0x0020:  640c 0000 7519 0d1d 018b 87ed ae5e 0000 
    0x0030:  0000 781f 0f00 0000 0000 1011 1213 1415 
    0x0040:  1617 1819 1a1b 1c1d 1e1f 2021 2223 2425 
    0x0050:  2627 2829 2a2b 2c2d 2e2f 3031 3233 3435 
    0x0060:  3637 
    Dump thread exiting... 
    Destroying session 37 

    Dumped 3 packet to console, dropped 0 packets.
    Done.
    ```


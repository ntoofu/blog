+++
title = "Gentoo Linux で (Portageで) パッケージビルドサーバーを使う"
date = "2022-05-01T17:12:17+09:00"
+++

Gentoo Linux でPortageを利用してソフトウェアをインストールする場合、一般的にはソースコードを取得しビルドすることになるが、消費リソースや依存関係の問題で苦労することも多い。その問題を緩和する方法として、Portageがサポートしている Binary Package のビルドサーバー構築を行った。その大まかな手順を書き残しておく。

# 背景

## Portageの弱点

Gentoo Linuxでは、標準でPortageというパッケージマネージャを利用する。
その大きな特徴として、パッケージインストール時に、事前にコンパイル・ビルドされたバイナリではなくソースコードを取得してきて、手元の環境でビルドすることが挙げられる。
これにより、細かな粒度で機能の有効化・無効化を調整したり、コンパイラによる最適化の恩恵を最大限に受けることができる。

一方で、手元でビルドする以上避けられない問題もある。
まず、パッケージインストール時に消費するリソースが大きいことが挙げられる。
ビルドする以上当然のことではあるが、Webブラウザのような巨大なソフトウェアのビルドには多くのCPU,メモリ,ストレージのリソースを消費する。
例えばFirefoxをビルドしようと考えると、Rustがビルド時に必要となるが、このインストールに際しては7GiB以上のディスクの空きスペースが要求される。

また、依存関係の解決にも苦労することがある。
目的のパッケージをインストールするにあたって、そのビルドに必要となるパッケージもインストールしなければならないため、通常より依存関係が複雑になりやすい。
パッケージによっては同時に環境内にインストールされることを禁じられている組み合わせもあり、これによって思うようにパッケージ更新が進まないことも多い。
特に、しばらくシステムの更新を怠っていると、同時に多くのパッケージの更新を行う必要が出てきてしまい、こうした問題に当たりやすくなる。

個人的な問題として、長い間放置していたラップトップにインストールしていたGentooのシステムを更新しようとした際に、まさにこれらの問題が大きな妨げとなった。
数世代前のマシンのスペックで大量のパッケージをソースコードからビルドすることは大きな負担であったし、依存関係を紐解いてすべてのパッケージのバージョンを新しくしていくことは現実的でないように思えた。

## Binary Package

こうした問題に対する解決策の一つとして、[Binary Package](https://wiki.gentoo.org/wiki/Binary_package_guide)が挙げられる。

Binary Packageは、パッケージをビルドした後のファイルをまとめたもので、これを利用してパッケージインストールをする場合は、ソースコードをコンパイルする工程が不要となる。
多くのLinuxディスリビューションにおいて、パッケージのインストール時には配布されているコンパイル済みのバイナリを取得してくるのによく似ているが、Portageの場合Binary Packageが配布されているわけではないため、自身でBinary Packageのビルドを行わねばならない。
したがって、実際の活用方法は、あるホストでビルドしたものを同じ構成の別ホストに流用したり、バックアップとしてビルド済みのものを退避しておく、といったことが想定されているようだ。

個人的に抱えていた古いラップトップ上のシステム更新に際しても、この方法は活用できそうだ。ラップトップ上のシステムと同等の環境を、より性能の高い別のホスト上に準備して、そちらでパッケージのビルドをすべて行ってしまい、その成果物をBinary Packageとしてラップトップに対して配布すれば良い。

## distccとの比較

また、別の案としてdistccによる分散コンパイルを行う方法も考えられる。

Portageからはdistccという分散コンパイラを呼び出すことができ、コンパイルの処理の一部をリモートホストに任せることができる。ローカルホストとリモートホストでgccのバージョンを揃える必要があるなど、いくつかの注意点はあるが、比較的手軽にローカルホストの負荷軽減を期待できる方法である。

ただし、ビルドそのものはローカルホストで実行することになるため、ビルド時に依存するパッケージのインストールは必要になるし、オブジェクトファイルのリンク処理などはローカルホストで実施することになる。そのため、今回はdistccを利用することはしていない。

## SYSROOT

Binary Packageをビルドする際には、バイナリをビルドする環境と実行する環境が異なる点に注意する必要がある。
CPUアーキテクチャのレベルで異なればクロスコンパイルをすることになるが、例えば同じx86-64系でマイクロアーキテクチャが異なる程度であれば、同じツールチェインを用いつつコンパイルオプションを変えることになる。
今回はマイクロアーキテクチャは異なるがアーキテクチャはx86-64で同一というケースに絞って考える。

基本的には実行環境に最適化してコンパイルができれば良いのだが、ビルド時に必要なパッケージについてはビルド環境で動作することもあるため、Portageのビルド時の設定を指定する `make.conf` で一律に実行環境向けの設定を書くわけにもいかない。
ビルド環境と実行環境で共通して動作するレベルまで最適化を控えるようにするのも手だが、せっかくGentooを使っているのだから、最適化したバイナリが得られるように工夫することを考える。

実は、まさにこうした事態をPortageは想定しており、 `ROOT` `SYSROOT` `PORTAGE_CONFIGROOT` 環境変数の指定によって挙動を細かく制御できるようになっている。
詳細は[manにも記載がある](https://dev.gentoo.org/~zmedico/portage/doc/man/emerge.1.html)。
これを利用することで、別のホストに最適化したBinary Packageをビルドすることが可能となる。

# 手順

## ビルドサーバーの準備

試行錯誤することが予想されたこともあって、環境の再作成が容易なようにDockerコンテナを使った。
適当に用意したそこそこのスペックのサーバー上で `gentoo/portage` イメージとvolumeを共有する形で `gentoo/stage3` のイメージを動かし、これをビルド環境として `docker exec` にて作業することにした。
（イメージの詳細については https://github.com/gentoo/gentoo-docker-images を参照）

ビルド環境内には `/alt` というディレクトリを作って、Binary Package向けの各種設定や成果物の配置先として使う。
これは、前述の `ROOT` `SYSROOT` `PORTAGE_CONFIGROOT` で指定する。
例えば `ROOT=/alt SYSROOT=/ PORTAGE_CONFIGROOT=/alt` のようにすることで、`/alt/etc/portage/make.conf` などが `make.conf` として参照されることになり、 `/alt/usr` 以下にライブラリや実行ファイルがインストールされるようになるが、ビルド時の依存の都合でビルド環境用にインストールされるものについては、通常通り `/etc/portage/make.conf` を参照した上で `/usr` 以下にライブラリや実行ファイルはインストールされることになる。

この `/alt` 以下にBinary Packageも保持されるので、volumeを介してsshd用のコンテナにもマウントさせ、SSH経由でアクセス可能な状態を作る。（sshd用コンテナにはrsyncも入れておく。）

細かい部分は後ほど補足するが、これらを実現するために以下のようなdocker-compose.ymlを利用した。

```
version: "3.9"
services:
  gentoo:
    image: "gentoo/stage3:latest"
    volumes:
      - "repos:/var/db/repos"
      - "gentoo:/var/db/repos/gentoo:ro"
      - "altroot:/alt/"
      - "./base/make.conf:/etc/portage/make.conf:ro"
      - "./base/package.use:/etc/portage/package.use/manual:ro"
      - "./alt/make.conf:/alt/etc/portage/make.conf:ro"
      - "./alt/package.use:/alt/etc/portage/package.use/manual:ro"
    command: ["tail", "-f"]
    cap_add:
      - ALL
    environment:
      ROOT: /alt/
      SYSROOT: /
      PORTAGE_CONFIGROOT: /alt/
      SYMLINK_LIB: "no"
  portage:
    image: "gentoo/portage:latest"
    volumes:
      - "gentoo:/var/db/repos/gentoo"
    command: ["/bin/true"]
  ssh:
    image: lscr.io/linuxserver/openssh-server
    environment:
      - PUID=1000
      - PGID=1000
      - TZ=Asia/Tokyo
      - PUBLIC_KEY=ecdsa-sha2-nistp256 AAA...
      - SUDO_ACCESS=true
      - PASSWORD_ACCESS=false
      - USER_NAME=gentoo
    volumes:
      - "altroot:/alt/:ro"
      - "repos:/var/db/repos:ro"
      - "gentoo:/var/db/repos/gentoo:ro"
    ports:
      - 22:2222
    restart: unless-stopped
volumes:
  repos:
  gentoo:
  altroot:
```

## リポジトリの設定

portageによるパッケージインストールで選択されるパッケージバージョンと、Binary Packageのバージョンは一致している必要があるので、クライアント（Binary Packageのインストール先になるホスト）側はBinary Packageのビルドサーバーがローカルに同期したディレクトリをリポジトリとして参照するのが望ましい。
そのため、クライアント側の `/etc/portage/repos.conf` は、

```
[gentoo]
location = /var/db/repos/portage
sync-type = rsync
sync-uri = ssh://gentoo@build-server/var/db/repos/gentoo
```

のような内容にする。

（このとき、SSHのポートとして22以外を使う場合、URIの中で指定しても正しく認識されないため、22を使うようにしたい。その場しのぎではあるが、emergeを呼び出すときに `PORTAGE_SSH_OPTS='-o Port=10022'` のようにすれば、一応指定はできる。）

また、overlayを利用する場合は、overlayの設定をビルドサーバーで行った上で、同様にクライアント側はビルドサーバーのディレクトリを参照する。
overlayの追加（例えば `haskell` overlayを追加する場合）は以下のような流れになる。

```
ROOT=/ SYSROOT=/ PORTAGE_CONFIGROOT=/ emerge eselect-repository
eselect repository enable haskell
ROOT=/ SYSROOT=/ PORTAGE_CONFIGROOT=/ emerge dev-vcs/git
ROOT=/ SYSROOT=/ PORTAGE_CONFIGROOT=/ emerge --sync haskell
```

## profileの変更

`sys-apps/systemd` と `sys-fs/udev` はconflictを起こすため同時にシステムには入れられず、かつ他のパッケージからの依存によりビルド環境にインストールされる可能性が高い。
ビルド環境として利用する前述のコンテナイメージにはudevがインストールされているが、Binary Packageとしてsystemdやsystemd依存のパッケージをビルドしたい場合は、先にビルド環境内をudevからsystemdに切り替えておくほうが良い。
そのため、systemd用のプロファイルに `eselect profile` などで切り替えた上で、
```
ROOT=/ SYSROOT=/ PORTAGE_CONFIGROOT=/ emerge -ND world
```
のようにしてudevからsystemdに切り替える。

## 設定のコピー

パッケージビルド時の細かな粒度での設定を表す "USEフラグ" について、クライアント側でemergeコマンド実行時に選択されるものと、Binary Packageのビルド時に選択されているものは、一致している必要がある。
そのため、USEフラグの決定に関わる設定について、クライアント側とビルドサーバー側で統一しておかねばならない。
具体的には、 `/etc/portage/package.use` 配下のファイルと、 `/etc/portage/make.conf` 内の一部の設定（ `USE` 変数など）である。

これらのファイルをビルドサーバーの `/alt` 以下に保存する。（例えば `/etc/portage/make.conf` であれば、 `/alt/etc/portage/make.conf` のようなパスで保存する。）

ビルドサーバーの `/alt/etc/portage/make.conf` は以下のような内容になる。

```
COMMON_FLAGS="-O2 -pipe -march=haswell -mmmx -msse -msse2 -msse3 -mssse3 -mcx16 -msahf -mmovbe -maes -mpclmul -mpopcnt -mabm -mfma -mbmi -mbmi2 -mavx -mavx2 -msse4.2 -msse4.1 -mlzcnt -mrdrnd -mf16c -mfsgsbase -mfxsr -mxsave -mxsaveopt --param l1-cache-size=32 --param l1-cache-line-size=64 --param l2-cache-size=3072 -mtune=haswell -fstack-protector-strong"
CFLAGS="${COMMON_FLAGS}"
CXXFLAGS="${COMMON_FLAGS}"
FCFLAGS="${COMMON_FLAGS}"
FFLAGS="${COMMON_FLAGS}"

CHOST="x86_64-pc-linux-gnu"
MAKEOPTS="-j8"
FEATURES="parallel-fetch buildpkg -ipc-sandbox -mount-sandbox -network-sandbox -pid-sandbox"

PORTDIR="/var/db/repos/gentoo"
DISTDIR="/alt/var/cache/distfiles"
PKGDIR="/alt/var/cache/binpkgs"

LC_MESSAGES=C

USE="acpi alsa -bindist ..."
LINGUAS="ja"
INPUT_DEVICES="evdev"
VIDEO_CARDS="intel i965"
```

`COMMON_FLAGS` は、実際にBinary Packageのインストール先として想定しているホストに最適化するように調整している。
`FEATURES` について、 `buildpkg` はBinary Packageをビルドするオプションで、CLIで `-b` 指定するのと同等である。 `-*-sandbox` は環境によっては不要なものだが、今回のようにコンテナ内でemergeを動かす場合は必要になる。
`USE` 以下はBinary Packageのインストール先で利用するものを取ってきたものになる。

なお、ビルドサーバーの `/etc/portage/make.conf` は以下のような内容にする。

```
COMMON_FLAGS="-O2 -pipe -march=native"
CFLAGS="${COMMON_FLAGS}"
CXXFLAGS="${COMMON_FLAGS}"
FCFLAGS="${COMMON_FLAGS}"
FFLAGS="${COMMON_FLAGS}"

CHOST="x86_64-pc-linux-gnu"
MAKEOPTS="-j8"
FEATURES="-ipc-sandbox -mount-sandbox -network-sandbox -pid-sandbox"

PORTDIR="/var/db/repos/gentoo"
DISTDIR="/var/cache/distfiles"
PKGDIR="/var/cache/binpkgs"

LC_MESSAGES=C
```

`COMMON_FLAGS` は、この設定でコンパイルされるバイナリはビルドサーバー内で動作するものであるため、 `-march=native` となっている。

また、これらに加えて `/var/lib/portage/world` をコピーしておくことで、利用しているパッケージをビルドサーバーで再構築できる。

## Binary Packageのビルド

```
ROOT=/alt SYSROOT=/ PORTAGE_CONFIGROOT=/alt emerge -abvND world
```

を実行する。 `-b` は Binary Packageをビルドするオプションである。

多くの場合、依存関係の都合でpackage.useを更新することになると思うが、 `etc-update` を呼ぶ場合、 `/alt/etc/portage/package.use` 配下の更新であれば
```
ROOT=/alt etc-update
```
とし、 `/etc/portage/package.use` 配下の更新であれば
```
ROOT=/ etc-update
```
とする。

### 細かい話

`acct-group/root-0` について、事前に `root` グループがないとインストールに失敗する。
今回 `ROOT` `SYSROOT` など分けている都合で、 `/alt` に chroot した環境では `root` グループがないような状態になってしまっており、エラーになっていた。

```
echo 'root:x:0:root' >> /alt/etc/group
```

のようにして回避した。

## Binary Packageのインストール

ここまでくれば、後はBinary Packageを利用するだけとなる。
クライアントにて、 `PORTAGE_BINHOST` を指定してemergeを呼び出す。 `-G` オプションによってBinary Packageを使用するようになる。

```
PORTAGE_BINHOST='ssh://gentoo@build-server/alt/var/cache/binpkgs' emerge -aGuvND world
```

# 問題点

いくつかの課題もある。

* USEフラグに関わる設定を共有する必要があるが、その部分が洗練されていない
  * 現状は手でコピーするような手順であり、繰り返しの実行には向かない
  * NFSで共有したり、gitで共有したり、色々やり方はあると思うが、いずれも泥臭くやらねばならなさそう
  * そもそもクライアント側ではBinary Packageをインストールするだけなら、ビルドサーバーで設定をいじってクライアントはemerge時にそれらを参照さえできれば良いかもしれないし、そうしたワークフローの変更も視野に入れて良い方法を考えたい
* ROOT, SYSROOT を分けてのビルド・インストールが安定していない
  * `acct-group/root-0` の例のように、ニッチなことをしているせいでうまくいかないケースを踏みがち
* ビルドサーバー構築手順の簡略化
  * docker-compose.ymlに完結するとかして、極力execで叩くものを減らしたい

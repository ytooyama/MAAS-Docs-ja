#Ubuntu MAAS 1.7(ESXiベース)のインストール

最終更新日: 2016/01/21 15:00


このドキュメントはVMware ESXiにUbuntu MAAS環境を作成して、仮想サーバーをMAASで管理する手順を示します。
本例では標準リポジトリーで提供されるMAAS 1.7.6+bzr3376とVMware ESXi 5.5 Update 3の組み合わせでインストールします。


##手順概要

これから、次の手順でそれぞれのセットアップを行います。

- ESXi 5.5のインストール 
- ESXi 5.5のパッチを適用
- (オプション)vCenter Serverのインストール
- (オプション)vCenter Serverのセットアップ
- 仮想マシンを作成
  - イメージサーバー用
  - MAASサーバー用
  - 仮想ノード x 5
- 仮想マシンにUbuntu Serverのインストール
  - イメージサーバー用
  - MAASサーバー用
- イメージサーバーのミラーを構築
- MAASのインストールとセットアップ
- MAASにノードを登録
  - 仮想ノード or 物理ノード or Other

※本例ではvCenter Serverのセットアップについては触れません。


##ネットワークスイッチについて

何を使っても構いませんが、STPの設定...Ciscoスイッチで言うところのPortFastはenableにしてください。[詳細はこちら](https://maas.ubuntu.com/docs/install.html#configure-switches-on-the-network)。


##イメージサーバーの作成

MAASでデプロイするOSのイメージは通常「maas.ubnntu.com」サーバーから取得しますが、次の手順に従ってイメージサーバーのミラーを作成できます。この手順にしたがってミラーを作成します。

- [イメージサーバーのミラーを作成する方法(英語)](https://maas.ubuntu.com/docs/sstreams-mirror.html)

マスターイメージサーバーは次の通りです。

- [リリース版](http://maas.ubuntu.com/images/ephemeral-v2/releases/)
- [デイリー版](http://maas.ubuntu.com/images/ephemeral-v2/daily/)


次のようにコマンドを実行すると、ミラー作成に必要なパッケージを全てインストールできます。

````
$ sudo apt-get install simplestreams ubuntu-cloudimage-keyring apache2
$ sudo mkdir -p /var/www/html/mirror/
````

マスターからミラーリングするには次のように実行します。

````
$ sudo sstream-mirror --keyring=/usr/share/keyrings/ubuntu-cloudimage-keyring.gpg http://maas.ubuntu.com/images/ephemeral-v2/releases/ /var/www/html/mirror/images/ephemeral-v2/releases 'arch=amd64' 'subarch~(generic|hwe-t)' 'release~(trusty|precise)' --max=1
````

`http://maas.ubuntu.com/images/ephemeral-v2/releases/` が対象とするイメージのバージョンで、リリースかデイリーのURLのうちどちらかを設定します。

`/var/www/html/mirror/ephemeral-v2/releases` はローカルのファイルの置き場です。Apache2を使ってイメージファイルを配信するので、`/var/www/html/`の下にマスターと同じディレクトリー構成のファイル群を配置します。

`arch=amd64やsubarch~(generic|hwe-t)` は、イメージのアーキテクチャーを指定しています。

`release~(trusty|precise)` はインポートしたいイメージを指定します。指定できるイメージはリリース、デイリーのどちらかを指定したかによって異なります(執筆日時点)。

`--max=1`の指定は、インポートするイメージの世代数を設定します。1の場合は最新イメージのみ、2以降は最新のイメージからその数の世代までインポートします。

コマンドを実行してミラーが完了したら、ブラウザでアクセスして参照可能なこと、ダウンロード可能なことを確認します。MAASをインストール後にMAASの設定を開き、「Boot Images」のSync URLをミラーサーバーに書き換えます。


##MAASのインストール

MAAS 1.7をインストールする前にUbuntu Server 14.04をインストールします。インストール後はアップデートを行い、最新の状態にします。

まず、リポジトリー情報を更新します。

````
maas$ sudo apt-get update
````

MAASパッケージを確認します。

````
maas$ apt-cache policy maas 
````

MAASの実行に必要なパッケージをすべてインストールします。

````
maas$ sudo apt-get install maas maas-dhcp maas-dns
````

[Web UIにアクセス](https://maas.ubuntu.com/docs/install.html#post-install-tasks)して、管理ユーザーのセットアップを実行します。


##MAASでKVM仮想マシンを使う場合の追加作業

###Virshコマンドのインストール

````
maas$ sudo apt-get install libvirt-bin
maas$ logout
````

###KVMホストのセットアップ

####セットアップ

````
kvm$ sudo apt-get install kvm virt-manager ssh-askpass-gnome bridge-utils
````

####ブリッジNIC(br0)の作成

ブリッジNICを作成します。

````
kvm$ sudo vi /etc/network/interfaces
...
auto br0iface br0 inet staticaddress 192.168.10.11
netmask 255.255.255.0
gateway 192.168.10.254
dns-nameservers 8.8.4.4bridge_ports eth0 
...
auto eth0iface eth0 inet static
address 0.0.0.0
````

####仮想マシンの作成

次のような構成で仮想マシンをKVMに作成します。

- CPU,Memoryは適切に
- NICはbr0に設定
- Boot OptionをNIC、HDDの順に設定


###MAASユーザーと作業環境のセットアップ

####ユーザーと作業環境のセットアップ

````
maas$ sudo mkdir -p /home/maas
maas$ sudo chown maas:maas /home/maas
maas$ sudo chsh -s /bin/bash maas
maas$ cat /etc/passwd |grep maas
maas:x:106:113::/var/lib/maas:/bin/bash
````

####maasユーザーのキーペアの作成

パスワードなしで認証するために、パスワード入力せずにEnterキーを数回押してキーペアを作成します。

````
maas$ sudo -u maas ssh-keygen
````

####maasユーザーの公開鍵をKVMホストに転送

以下はytooyamaユーザーでIPアドレス(192.168.10.11)が設定されたKVMホストに転送する例です。

````
maas$ sudo -u maas -i ssh-copy-id ytooyama@192.168.10.11
````

####maasユーザーで仮想マシン一覧を取得

````
maas$ sudo -u maas virsh -c qemu+ssh://ytooyama@192.168.10.11/system list --all
````

virshコマンドのインストール後、virshコマンドでエラーが出る場合は一旦再起動してください。


##MAASのセットアップ

MAASをデプロイツールとして利用するため、残りの作業を実施します。必要なセットアップ項目は次の通りです。

- MAASダッシュボード(http://maas-server-ip/MAAS)にアクセス
- [Cluster Interfaceの設定](https://maas.ubuntu.com/docs/install.html#configure-dhcp)
  - アドレス配布に使うNICを指定
  - 管理するサービスを指定(ex. "DHCP and DNS")
  - ゲートウェイのIPアドレス
  - DHCP dynamic IP range(Enlist、Commissioningなどで利用)
  - Static IP range(ノードに割り当てる範囲)
- ユーザー設定
  - ユーザーに割り当てたノードにデプロイしたOSに設定するSSH公開鍵を登録
- 全般設定
  - Upstream DNSの設定(スペース追加で複数DNSを指定可能)
  - DNSSECの設定(Upstream DNSの対応状況に応じて変更)
  - イメージのURLをミラーサーバーに設定変更
- [イメージのインポート](https://maas.ubuntu.com/docs/install.html#import-the-boot-images)
  - UbuntuイメージはCommitioningに必要なのでインポート必須
  - その他OSをダッシュボードやコマンドなどで追加してインポートを実行する 
- ノードを追加
  - 物理ノード、仮想ノードなどをMAASに登録


##MAASに仮想ノードを登録する

MAASにノードを追加するには次のように行います。

- 仮想マシンを作成
- 仮想マシンにMAASのネットワークを設定
- 仮想マシンをPXEブートするように設定(ESXiのBIOSで設定する)
- 仮想マシンを起動するとEnlistが行われる
- MAASダッシュボードにアクセスして、仮想マシンの情報を入力


項目          | 設定例
------------- | -------------
Power type    | Virsh
Power address | qemu+ssh://ytooyama@192.168.10.11/system
Power ID      | VM名
Power Password| (空)

Power IDには「maasユーザーで仮想マシン一覧を取得」で指定したものと同じものを指定します。

物理ノードを追加するには適切なものをPower typeに指定します(ex. IPMI)。KVMの仮想マシンを追加するには[Virshを指定](http://askubuntu.com/questions/292061/how-to-configure-maas-to-be-able-to-boot-virtual-machines)します。

- 設定が正しければ、次回パワーオンするとOSのプロビジョニングが始まります。
  - 例えば、MAASダッシュボードで追加したユーザーに対してSSHキーの登録がされていないと「Start Node」がうまく始まりません。
- 起動後はログなどを確認します。

````
$ sudo tail -f /var/log/maas/maas.log
````

##メモ

###VMにNested Virtualizationを設定する

`/vmfs/volumes/[your-datastore]/[your-vm]/[your-vm].vmx`に「vhv.enable = "TRUE"」を追記してVMを起動します。
OS起動後、Linuxなら`lsmod |grep kvm`や`kvm-ok`などで確認します。
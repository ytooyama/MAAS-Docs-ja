#Ubuntu MAAS 1.9(ESXiベース)のインストール

最終更新日: 2016/01/21 10:30


このドキュメントはVMware ESXiにUbuntu MAAS環境を作成して、仮想サーバーをMAASで管理する手順を示します。
本例ではMAAS 1.9.0+bzr4533、VMware ESXi 5.5 Update 3を使っています。


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


##ESXiについて

MAAS 1.8以降はESXi 5.5、Linux KVMの管理ができる様になっており、ハイパーバイザー上の仮想マシンをMAASの仮想ノードの一つとして管理可能です。ESXiのマイナーバージョンはなんでも構わないため、本例ではESXi 5.5 Update 3を利用します。MAASがESXiと連携するにはvSphere APIが利用できる必要があります。そのため、有償(もしくは評価版)ライセンスが適用されたESXiである必要があります。


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

MAAS 1.9をインストールする前にUbuntu Server 14.04をインストールします。インストール後はアップデートを行い、最新の状態にします。インストールしたUbuntu ServerにMAAS 1.9をインストールするため、MAASのPPAを追加します。

まず、PPAの追加に必要なパッケージをインストールします。

````
$ sudo apt-get install software-properties-common python-software-properties
````

安定版のパッケージを使う場合は次の通りコマンドを実行します。

````
$ sudo add-apt-repository ppa:maas/stable
````


リポジトリーを追加したので、リポジトリー情報を更新します。

````
$ sudo apt-get update
````

MAASパッケージを確認します。

````
$ apt-cache policy maas 
````

MAASとESXi VMをマネージメントするために必要なパッケージをインストールします。

````
$ sudo apt-get install maas python-pyvmomi
````

[Web UIにアクセス](https://maas.ubuntu.com/docs/install.html#post-install-tasks)して、管理ユーザーのセットアップを実行します。


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
Power type    | VMware
Name          | VM名
UUID          | VMのUUID
hostname      | ESXiのIPアドレスもしくはFQDNで
user          | ユーザー
password      | パスワード
API Port      | 443
API protocol  | https

物理ノードを追加するには適切なものをPower typeに指定します(ex. IPMI)。ESXiの仮想マシンを追加するにはVMwareを指定、KVMの仮想マシンを追加するには[Virshを指定](http://askubuntu.com/questions/292061/how-to-configure-maas-to-be-able-to-boot-virtual-machines)します。

- 設定が正しければ、次回パワーオンするとOSのプロビジョニングが始まります
- 起動後はログなどを確認します

````
$ sudo tail -f /var/log/maas/maas.log
````

##メモ

###VMにNested Virtualizationを設定する

`/vmfs/volumes/[your-datastore]/[your-vm]/[your-vm].vmx`に「vhv.enable = "TRUE"」を追記してVMを起動します。
OS起動後、Linuxなら`lsmod |grep kvm`や`kvm-ok`などで確認します。

###ESXi VMのUUIDを確認する

UUIDはESXiコンソール上で次のようにコマンドを実行すると確認できます。
VMのUUIDはvmxファイルに書かれているので、grepすると目的の項目を見つけられます。

````
esxi-host # cat /vmfs/volumes/datastore/esx-vm1/esx-vm1.vmx |egrep "displayName|uuid.bios"
uuid.bios = "56 4d 8f cd e3 38 19 71-c3 93 81 29 64 eb 9a cc"
````

###MAASにUUIDを設定する

VMのUUIDがわかったら、UUIDを「8-4-4-4-12」表記に直してMAASに設定します。

(例) 564d8fcd-e338-1971-c393-812964eb9acc
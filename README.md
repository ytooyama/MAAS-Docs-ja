# MAAS-Docs-ja
Canonical MAASの日本語版ドキュメント（非公式）


## インストールの流れ
インストール方法は以下やこのリポジトリーで公開されているドキュメントを参照。9/28/2015現在、Ubuntu Trusty (14.04.3)ではtrusty-updatesリポジトリーでMAAS 1.7系のバージョンのパッケージが標準で用意されているようです。

- <http://maas.ubuntu.com/docs1.7/install.html>

MAAS 1.8以降の新しいバージョンをインストールしたい場合は、MAAS Stable PPAを追加する必要があります。PPAのパッケージは新しい安定版が公開されるとなくなるので注意が必要です。

- <https://launchpad.net/~maas-maintainers/+archive/ubuntu/stable>


## MAAS 1.8の変更点（概要）

- Web UIのデザインが変わった
- MAAS Region ControllerがApache非依存になった
- Node & Storage タグの管理をWeb UIでできるようになった
- VMware ESXi 5.5を仮想ノードのホストとして使えるようになった

詳細は[changelog](http://maas.ubuntu.com/docs/changelog.html#id12)を参照

## MAAS 1.7の変更点（概要）

- イメージの管理UIが追加
- ノードイベントログがWeb UIに追加された
- maas-proxyが使われるようになった
- Windows Server,CentOS,Suse Linuxのサポート
- カスタムイメージをサポート[^1]

詳細は[changelog](http://maas.ubuntu.com/docs/changelog.html#id31)を参照

[^1]: [コマンド](https://maas.ubuntu.com/docs/bootsources.html)でカスタムイメージの登録とインポートを実行する必要がある。
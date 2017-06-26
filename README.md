# MAAS-Docs-ja

Last Update: 2017/6/26

Canonical MAASの日本語版ドキュメント（非公式）


## インストールの流れ

インストール方法は以下やこのリポジトリーで公開されているドキュメントを参照。2017/6/26/現在、Ubuntu Trusty (14.04.5)では標準リポジトリーにMAAS 1.9系のバージョンのパッケージが用意されているようです。

- <https://docs.ubuntu.com/maas/1.9/en/install>

2017/6/26/現在、Ubuntu Xenial (16.04.x)では標準リポジトリーにMAAS 2.1系のバージョンのパッケージが用意されているようです。

- <https://docs.ubuntu.com/maas/2.1/en/installconfig-install>

以降の新しいバージョンを16.04LTSで構築したい場合は、MAAS Stable PPAを追加する必要があります。PPAのパッケージは新しい安定版が公開されるとなくなるので注意が必要です。

- <https://launchpad.net/~maas/+archive/ubuntu/stable>

なお、Ubuntu Standard版にはLTS版よりも新しいバージョンのMAASがバンドルされます。


## MAAS 2.2の変更点

- <https://docs.ubuntu.com/maas/2.2/en/release-notes>

## MAAS 2.1の変更点

- <https://docs.ubuntu.com/maas/2.1/en/release-notes>

## MAAS 1.9の変更点 (概要)

- Software RAIDサポート
- LVMサポート
- Bcacheサポート
- VLAN,Bondingなどのサポート
- SubnetのVLAN IDや使用率の表示が可能に

### 変更履歴

- <https://docs.ubuntu.com/maas/1.9/en/changelog>


## MAAS 1.8の変更点 (概要)

- Web UIのデザインが変わった
- MAAS Region ControllerがApache非依存になった
- Node & Storage タグの管理をWeb UIでできるようになった
- VMware ESXi 5.5を仮想ノードのホストとして使えるようになった


## MAAS 1.7の変更点 (概要)

- イメージの管理UIが追加
- ノードイベントログがWeb UIに追加された
- maas-proxyが使われるようになった
- Windows Server,CentOS,Suse Linuxのサポート
- カスタムイメージをサポート(注1)


----
(注1) [コマンド](https://maas.ubuntu.com/docs/bootsources.html)でカスタムイメージの登録とインポートを実行する必要がある。
# Web Hosting Service Construction System

Webホスティングサービス構築システム

## 利用方法

ホストファイル、group_varsやhost_varsを適当に設定しておく。各変数は palybooks/roles/common/default/main.yml を参考にする。

ctlのセットアップを行った後に、セットアップ、アップデート、全体設定、ユーザー同期を行う。

1. `ansible-playbook -l ctl setup.yml`
2. `ansible-playbook setup.yml`
3. `ansible-playbook update_reboot.yml -e update_autoremove=yes`
4. `ansible-playbook conf_all.yml`
5. `ansible-playbook user_sync.yml`

## ユーザー管理

### 登録

ユーザーの登録とTLSの作成後にユーザー同期を行う。ユーザー名には `/^[a-z][a-z0-9_]*$/` で、ユーザー名の"_"はサイト名(FQDN)では"-"に変換される。

1. `ansible-playbook create_webuser.yml -e user={ユーザー名}`
2. `ansible-playbook create_tls.yml -e user={ユーザー名}`
3. `vim ../data/webuser/{ユーザー名}.yml`
4. `ansible-playbook user_present.yml -e user={ユーザー名}`

### 無効化

サイトを無効化するにはdisabledをtrueにしてuser_presentを実行する。

1. `vim ../data/webuser/{ユーザー名}.yml`
    `disabled`を`true`に設定する。
2. `ansible-playbook user_present.yml -e user={ユーザー名}`

### 削除

ファイルがつかんである可能性があるため、削除前に無効にすることを推奨する。

`{ユーザー名}.yml`のファイルを退避させてから`user_absent.yml`を実行する。

1. `mv ../data/webuser/{ユーザー名}.yml ../data/backup/.`
2. `ansible-playbook user_absent.yml -e user={ユーザー名}`

ログも含めて、ファイルサーバー上のファイルはすべて削除される。`../data/tls`にある証明書類は自動で退避や削除はされないため、必要に応じて手動で退避しておくこと。

## ファイルシステム

glusterを使用した分散型ファイルシステムになっている。nfs-ganeshaでpNFSも提供している。

### スナップショットについて

下記コマンドでファイルサーバーのスナップショットを取得できる。

```shell
ansible-playbook gluster_ss.yml
```

下記のことにも注意する。

#### スナップショット数

snapshotの保持数は`gluster_snapshot_hard`で調整する。(デフォルトは256)

```shell
gluster snapshot config web snap-max-hard-limit 100
```

#### スナップショット頻度

起動後に取得されたスナップショットが多いと再起動時にタイムアウトが発生する場合がある。タイムアウトが頻発する場合は、`gluster_device_timeout` でタイムアウト時間を調整すること。

#### メタデータ容量

スナップショットが多くなるとメタデータの容量が圧迫される。`lvs -a` で容量を確認すること。メタデータの領域が少なくなっている場合は

```shell
sudo lvextend --poolmetadatasize 1G vg/pool
```

等を実行する。(1Gはサイズに応じて要調整)

#### 自動拡張をオンにすべき？

(これは未実施)
/etc/lvm/lvm.confでactivation/thin_pool_autoextend_thresholdを100未満にして、poolを自動拡張すべき？

#### スナップショットタイミング

ログローテートの時刻(デフォルトは午前0時)と被る場合は、ログローテートの実行時刻をずらす。group_varsやhost_varsで`logrotate_time`変数を設定しsetup.ymlを適用することで、実行時刻を調整できる。

### EOLに伴う注意

GlusterおよびGaneshaはSIGのレポジトリであり、CentOS Stream向けであるため5年でEOLになる。EOLを迎えた場合はpNFS経由でのアクセスとなり、冗長性はなくなる。

pNFSがちゃんと動いているかは、ちょっと怪しい。

### キャッシュが圧迫

TODO: 自動的な対応は未実施

EL9ではdnfの古いキャッシュが溜まりすぎている場合がある。ひとまず、下記で減らせる。

```
sudo dnf clean packages
# または
sudo dnf clean all
```

定期的なパッケージキャッシュのクリーンを検討した方がいいかもしれない？

## 新サーバーの追加

1. ansibleが見に行くhostsファイルに新サーバーを追加する。
2. コントロールサーバーから新サーバーにSSH接続をする。`ssh-copy-id`で鍵ファイルをコピーしておく。
3. 新サーバーに対して、setup.yml、update_reboot.yml、conf_all.yml、user_sync.ymlを実行する。
4. 全体に対して、上記を実行する。

## オプション

### update_reboot.yml

- autoremove: 不要パッケージの自動削除を実行する。
- id: 特定の番号のサーバーのみ実行する。

### create_webuser.yml

- user: 必須。ユーザー名。英子文字、数字、アンダーバー"_"のみ使用可能。
- site: サイトのFQDN。未指定の場合は、ユーザー名の"_"を"-"にかえ、デフォルトのドメイン名を付ける。
- uid: ユーザーのuid。未指定の場合は、他のユーザーで指定されている番号に+1を割り当てる。

### create_tls.yml

see playbooks/roles/creat_tls/REDME.md

## 注意事項と制限事項

### vsftpdのデフォルト変更

ホームページビルダー22への対応として、デフォルトを下記のように変更している。

- TLSv1.3は無効
- データ接続のタイムアウトを10秒
- データアップロード時の終了の厳格化を無効

ホームページビルダー22等の一部のFTPクライアントでTLSv1.3ではデータ転送が正常に行えない不具合が発生することがあるため、デフォルトではTLSv1.3は無効にし、TLSv1.2のみ有効としている。

<https://access.redhat.com/solutions/7042423>

### selinux-policyとselinux-policy-targetedのアップデート除外

Rocky9でglusterfsをマウントしている環境でselinux-policy-targeted適用時にauditdが暴走する。回避には `sudo setenforce 0` とパーミッションモードに切り替えてから `sudo dnf update` を実施し、再起動すればよい。

selinux-policyおよびselinux-policy-targetedはアップデートから除外し、必要な場合は、他のアップデートが終わった後にfailになるようにしている。failが検出された場合は、update_in_permissive.ymlのプレイブックで対応すること。

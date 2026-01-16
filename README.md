# Web Hosting Service Construction System

Webホスティングサービス構築システム

## 利用方法

ansibleが見に行くインベントリ、group_vars、host_varsを適当に設定しておく。各変数は palybooks/roles/common/default/main.yml を参考にする。

### インベントリのグループ

各グループは次のようになっている。

- ctl: コントロールサーバー、このansibleを実行する。
- mgt: 管理サーバー、ユーザーがログインしてコンテンツの管理を行う。
- fs: ファイルサーバー、コンテンツがおいてある。
- app: アプリケーションサーバー、PHPやCGIなどの動的コンテンツを実行する。
- web: ウェブサーバー、HTMLやCSS等の静的コンテンツを返す。
- db: データベースサーバー、MariaDBやPostgreSQL等のデータベースを実行する。
- bk: バックアップサーバー、コンテンツやデータベースのバックを保管する。
- lb: 負荷分散サーバー、ウェブサーバーへのアクセスを分散する。
- rp: リバースプロキシサーバー、ウェブサーバーへのリバースプロキシになる。
- test: テストサーバー、サイトの動作を確認する。
- tpl: テンプレートサーバー、他サーバーのテンプレートになる。

このうち、fs,db,bkは設定時に空のディスクが必要になる。

### 初回セットアップ手順

ctlのセットアップを行った後に、セットアップ、アップデート、全体設定、ユーザー同期を行う。

1. `ansible-playbook -l ctl setup.yml`
2. `ansible-playbook setup.yml`
3. `ansible-playbook update_reboot.yml -e update_autoremove=yes`
4. `ansible-playbook conf_all.yml`
5. `ansible-playbook user_sync.yml`

### サーバーを追加する手順

あらかじめ、追加するサーバーにansible実行ユーザーのの公開鍵が登録しておく。

1. インベントリにサーバーを追加する。
2. `ansible-playbook -l ctl setup.yml`
3. `ansible-playbook setup.yml`
4. `ansible-playbook -l {{サーバー名}} update_reboot.yml -e update_autoremove=yes`
5. `ansible-playbook conf_all.yml`
6. `ansible-playbook user_sync.yml`
7. `ansible-playbook health_check.yml`

update_reboot.ymlのみサーバー名を指定する。setup.yml、conf_all.yml、user_sync.yml、health_check.ymlはサーバーを指定せずに実行する。ctl,fs,db,lbの場合は、user_sync.ymlを省略してもよい。

ファイルサーバーの追加は、レプリカ数と同じ数のサーバーが必要になる。追加と同時にrebalanceも実行されるため、ファイルサーバー全体が少し重くなる。なお、各サーバーが

### サーバーを削除する手順

追加とは順番が逆になることに注意する。

1. インベントリからサーバーを削除する。
2. `ansible-playbook user_sync.yml`
3. `ansible-playbook conf_all.yml`
4. `ansible-playbook setup.yml`
5. サーバーをシャットダウンする。
6. `ansible-playbook health_check.yml`

setup.ymlを実施する前にuser_sync.ymlとconf_all.ymlをサーバーを指定せずに実施する。ctl,fs,db,lbの場合は、user_sync.ymlを省略してもよい。

ファイルサーバーの削除はansibleではできない。「Glusterのbrick構成変更」を参照すること。

## ユーザー/サイト管理

ユーザーとサイトは一対一で紐づけられている。ユーザー名は `/^[a-z][a-z0-9_]*$/` でなければならず、ユーザー名の`_`はサイト名(FQDN)では`-`に変換される。そのため、作成したいサイト名の`-`を`_`にしたユーザー名で登録する必要がある。

ACMEによる証明書自動更新には対応していない。

### 登録

ユーザーの登録とTLSの作成後にユーザー同期を行う。

1. `ansible-playbook create_webuser.yml -e user={{ユーザー名}}`
2. `ansible-playbook create_tls.yml -e user={{ユーザー名}}`
3. `../data/csrs/{{fqdn}}.csr`から証明書を作成し、`../data/certs/{{fqdn}}.cer`に置く。
4. `vim ../data/webuser/{{ユーザー名}}.yml`
5. `ansible-playbook user_present.yml -e user={{ユーザー名}}`

`data/webuser/{{ユーザー名}}.yml`ファイルのパラメーターの詳細は[ユーザー設定](doc/webuser_yml.md)に記載している。

TLSの種類を変えたい場合はcreate_tls.ymlで下記を追加する。

- `-e tls_type=ECC -e tls_curve=secp384r1 -e tls_digest=sha384`
- `-e tls_type=RSA -e tls_size=4096 -e tls_digest=sha384`

自己署名の場合はcreate_tls.ymlで下記を追加する。

- `-e selfsigned=yes`

### 無効化

サイトを無効化するにはdisabledをtrueにしてuser_presentを実行する。

1. `vim ../data/webuser/{{ユーザー名}}.yml`
    `disabled`を`true`に設定する。
2. `ansible-playbook user_present.yml -e user={{ユーザー名}}`

### 削除

ファイルがつかんである可能性があるため、削除前に無効にすることを推奨する。

`{{ユーザー名}}.yml`のファイルを退避させてから`user_absent.yml`を実行する。

1. `mv ../data/webuser/{{ユーザー名}}.yml ../data/backup/.`
2. `ansible-playbook user_absent.yml -e user={{ユーザー名}}`

ログも含めて、ファイルサーバー上のファイルはすべて削除される。`../data/tls`にある証明書類は自動で退避や削除はされないため、必要に応じて手動で退避しておくこと。

### 証明書更新

1. `ansible-playbook create_tls.yml -e user={{ユーザー名}} -e backup=yes`
2. `../data/csrs/{{fqdn}}.csr`から証明書を作成し、`../data/certs/{{fqdn}}.cer`に置く。
3. `ansible-playbook user_present.yml -e user={{ユーザー名}}`
4. `ansible-playbook restart_web.yml`
5. `ansible-playbook restart_app.yml -e user={{ユーザー名}}`

証明書を書き換えてもnginxやhttpdは再起動しないため、新しい証明書反映されない。webとappのリスタートが必要になる。

TLSの種類を変えたい場合はcreate_tls.ymlで下記を追加する。

- ECC ECDSA(NIST P-384) + SHA384`-e tls_type=ECC -e tls_curve=secp384r1 -e tls_digest=sha384`
- RSA 4096ビット + SHA384 `-e tls_type=RSA -e tls_size=4096 -e tls_digest=sha384`

自己署名の場合はcreate_tls.ymlで下記を追加する。

- `-e selfsigned=yes`

## ファイルシステム

glusterを使用した分散型ファイルシステムになっている。nfs-ganeshaでpNFSも提供している。

### スナップショットについて

下記コマンドでファイルサーバーのスナップショットを取得できる。

```shell
ansible-playbook gluster_ss.yml
```

下記のことにも注意する。

#### スナップショット数

snapshotの保持数は`gluster_snapshot_hard`で調整する。(デフォルトは10)

#### スナップショット頻度

起動後に取得されたスナップショットが多いと再起動時にタイムアウトが発生する場合がある。タイムアウトが頻発する場合は、`gluster_device_timeout` でタイムアウト時間を調整すること。

#### メタデータ容量

スナップショットが多くなるとメタデータの容量が圧迫される。`lvs -a` で容量を確認すること。メタデータの領域が少なくなっている場合は下記コマンドで設定する。

```shell
sudo lvextend --poolmetadatasize 1G vg/pool
```

デフォルトでは1GiBで作成するようになっている。状況に応じて増やすこと。なお、ansible-playbookでは作成時のみ指定可能で、作成後の変更はバグのため対応していない。

#### 自動拡張をオンにすべき？

(これは未実施)
/etc/lvm/lvm.confでactivation/thin_pool_autoextend_thresholdを100未満にして、poolを自動拡張すべき？

#### スナップショットタイミング

ログローテートの時刻(デフォルトは午前0時)と被る場合は、ログローテートの実行時刻をずらす。group_varsやhost_varsで`logrotate_time`変数を設定しsetup.ymlを適用することで、実行時刻を調整できる。

### EOLに伴う注意

GlusterおよびGaneshaはSIGのレポジトリであり、CentOS Stream向けであるため5年でEOLになる。EOLを迎えた場合はpNFS経由でのアクセスとなり、冗長性はなくなる。

pNFSがちゃんと動いているかは、ちょっと怪しい。

### キャッシュが圧迫

EL9ではdnfの古いキャッシュが溜まりすぎている場合がある。clean.ymlでキャシュ削除できる。または、下記コマンドをサーバーで実行する。

```shell
sudo dnf clean packages
# または
sudo dnf clean all
```

TODO: 定期的なパッケージキャッシュのクリーンを検討した方がいいかもしれない？

### 容量増加(ディスク追加)

LVMになっている領域はディスクを追加することで容量を増やすことができる。

1. ディスクを追加し、再起動でOSに認識させる。
2. そのサーバーを対象としてconf_all.ymlを実行する。
    `ansible-playbook -l 【対象サーバー】conf_all.yml`
3. 追加されたディスクにPVが作られ、VGに追加し、LVを拡張し、ボリュームをリサイズする。

### Glusterのbrick構成変更

ansibleで追加以外の構成変更はできないため、削除や置換は手動で変更する必要がある。

`gluster_cluster_servers`が未設定の場合、fsサーバーすべてを見に行くため、追加したfsが追加されないように、まずは、現在のfsサーバーのみをリストとし記載しておく。この状態であれば、メインサーバーに対してconf_fs.ymlを実行しても、ボリュームに追加されることはなく、fsサーバーのpeerのみ設定される。追加や置換をする場合は、peerの設定までは完了しておくこと。

サーバーを追加する場合はansibleで可能であり、次のサーバー追加の項目を参照すること。ただし、レプリカ数(デフォルトは3)と同じ数を一度に追加する必要がある。add-brickを実行することで手動で追加することもできる。

```shell
gluster volume add-brick web fs1:/data/web fs2:/data/web fs3:/data/web
```

削除も同様に、レプリカ数と同じ数を減らす必要がある。手動でremove-brickを実行する。(絶対にforceはつかないこと、ファイルが見えなくなる)

```shell
gluster volume remove-brick web fs1:/data/web fs2:/data/web fs3:/data/web start
```

削除は他のサーバーへデータの移動が必要になる場合があるため、すぐに完了しない。下記コマンドで状況を確認する必要がある。

```shell
gluster volume remove-brick web fs1:/data/web fs2:/data/web fs3:/data/web status
```

サーバー故障等により1台だけ置き換えるという場合は、手動でreplica-brickを実行する。下記はfs1をfs4に置き換える場合である。(replica-brickは故障時のみのに対応すること)

```shell
gluster volume replace-brick web fs1:/data/web fs4:/data/web commit force
```

サーバーの追加、削除、置換後は`gluster_cluster_servers`の構成を変えてメインサーバーでconf_fs.ymlをすることで、構成に問題ないかのチェックを行うことができる。/etc/fstabを書き換えるために、全サーバーに対してconf_all.ymlを実行し、ファイルサーバーを見に行くサーバー(web,app,mgt,bk)は再起動しておくこと。

クライアントを含めて構成変更が完了した後に、リバランスを実行する。

```shell
gluster volume rebalance web start
gluster volume rebalance web status
```

リバラスが完了しても、スナップショットが残る。削除予定のサーバーが管理しているスナップショットをすべて削除されて、サーバーが削除可能になる。conf_fs.ymlを実行してもpeerは削除されないので、手動でpeerから削除する。

```shell
gluster peer detach fs1
```

これで、サーバーを廃棄可能になる。

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

### DNFのキャッシュ増加

EL8やEL9を長時間運用しているとDNFのキャッシュが溜まって、ディスクを圧迫する。今のところ、clean.ymlを定期実行すうることで回避するが、ディスク容量に応じて自動実行も考えたい。

### 中間証明書

`tls_im_certs`で証明書の名前(CN)とファイル名のマップを作ることで、中間証明書がインストールされる。中間証明書から先は見に行かないため、2つ以上の証明書をチェインされたい場合は、チェインされた中間証明書のファイルを用意する必要がある。

### Streamのサポート終了後のGlusterFS

デフォルトのGlusterFSはCentOS Stream用のレポジトリのため、SteramのEOLとともにレポジトリが消される。クライアント向けはStreamようではないRHEL向けのレポジトリを見に行くカスタムレポジトリで対応している。サーバーはRHEL向けではglusterfs-serverが入らないため、Stream版を使い続けるしかない。

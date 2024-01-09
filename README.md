# Web Hosting Service Construction System

Webホスティングサービス構築システム

## 利用方法

ホストファイル、group_varsやhost_varsを適当に設定しておく。各変数は palybooks/roles/common/default/main.yml を参考にする。

ctlのセットアップを行った後に、アップデート、セットアップ、全体設定、ユーザー同期を行う。

1. `ansible-playbook -l ctl setup.yml`
2. `ansible-playbook update_reboot.yml -e update_autoremove=yes`
3. `ansible-playbook setup.yml`
4. `ansible-playbook conf_all.yml`
5. `ansible-playbook user_sync.yml`

## 登録

ユーザーの登録とTLSの作成後にユーザー同期を行う。ユーザー名には `/^[a-z][a-z0-9_]*$/` で、ユーザー名の"_"はサイト名(FQDN)では"-"に変換される。

1. `ansible-playbook create_webuser.yml -e user={ユーザー名}`
2. `ansible-playbook create_tls.yml -e user={ユーザー名}`
3. `vim ../data/webuser/{ユーザー名}.yml`
4. `ansible-playbook user_sync.yml -e user={ユーザー名}`

## スナップショットについて

下記コマンドでファイルサーバーのスナップショットを取得できる。

```
ansible-playbook gluster_ss.yml
```

下記のことにも注意する。

### スナップショット数

snapshotの保持数は`gluster_snapshot_hard`で調整する。(デフォルトは256)

```
gluster snapshot config web snap-max-hard-limit 100
```

### スナップショット頻度

起動後に取得されたスナップショットが多いと再起動時にタイムアウトが発生する場合がある。タイムアウトが頻発する場合は、`gluster_device_timeout` でタイムアウト時間を調整すること。

### メタデータ容量

スナップショットが多くなるとメタデータの容量が圧迫される。`lvs -a` で容量を確認すること。メタデータの領域が少なくなっている場合は

```
sudo lvextend --poolmetadatasize 1G vg/pool
```

等を実行する。(1Gはサイズに応じて要調整)

### スナップショットタイミング

ログローテートの時刻(デフォルトは午前0時)と被る場合は、ログローテートの実行時刻をずらす。group_varsやhost_varsで`logrotate_time`変数を設定しsetup.ymlを適用することで、実行時刻を調整できる。

## 新サーバーの追加

1. ansibleが見に行くhostsファイルに新サーバーを追加する。
2. コントロールサーバーから新サーバーにSSH接続をする。`ssh-copy-id`で鍵ファイルをコピーしておく。
3. 新サーバーに対して、setup.yml、conf_all.yml、update_reboot.yml、user_sync.ymlを実行する。
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

# Web Hosting Service Construction System

Webホスティングサービス構築システム

## 利用方法

1. setup.yml
2. update_reboot.yml -e autoremove=1
3. conf_all.yml
4. user_sync.yml

## 登録

- create_webuser.yml
- create_tls.yml

## スナップショットについて

ファイルサーバーでスナップショットを取る際に、下記のことに注意する。

スナップショットが多くなるとメタデータの容量が圧迫される。`lvs -a` で容量を確認すること。メタデータの領域が少なくなっている場合は

```
sudo lvextend --poolmetadatasize 1G vg/pool
```

等を実行する。(1Gはサイズに応じて要調整)

snapshotの保持数は次のコマンドで調整する。(デフォルトはhard256のsoft90%)

```
gluster snapshot config web snap-max-hard-limit 100
```

起動後に取得されたスナップショットが多いと再起動時にタイムアウトが発生する場合がある。毎日スナップショットを取る場合は、週一回の再起動を推奨する。

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

# Web Hosting Service Construction System

Webホスティングサービス構築システム

## 利用方法

1. setup.yml
2. conf_all.yml
3. user_sync.yml

## 登録

- create_webuser.yml
- create_tls.yml

## スナップショットについて

`lvs -a` で容量を確認する。メタデータの領域が少なくなっている場合は

```
sudo lvextend --poolmetadatasize 1G vg/pool
```

を実行する。(1Gは微調整)

snapshotの容量自体は次のコマンドで調整する。(デフォルトはhard256のsoft90%)

```
gluster snapshot config web snap-max-hard-limit 100
```

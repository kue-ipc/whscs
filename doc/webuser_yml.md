# ユーザー設定

`{{ユーザー名}}.yml`はユーザーに紐づくサイトの設定になっている。パラメーター変更後に`ansible-playbook user_present.yml -e user={{ユーザー名}}`を実行するうことで、設定を反映できる。

## パラメーター一覧

- name: (string) [変更不可] ユーザー名
- site: (string) サイトのFQDN
- enabled: (boolean) サイトが有効無効
- host: (string) 使用するサーバー
- uid: (integer) [変更不可] ユーザーID番号
- db_use: (boolean) MariaDB使用の有無
- db_pass: (string) MariaDBのパスワード
- pg_use: (boolean) PostgreSQL使用のの有無
- pg_pass: (string) PostgreSQLのパスワード
- app_use: (boolean) アプリケーション使用の有無
- app_port: (intger) [変更不可] アプリケーションが使用するポート
- app_path: (string) アプリケーション使用になるパス
- app_one: (boolean) アプリケーション使用を1台のサーバーに限定
- public: (boolean) 外部公開の有無
- http: (boolean) HTTP接続の有無、falseの場合はHTTPSにリダイレクト

[変更不可]となっている項目は変更してはいけない。変更した場合の動作は保証できない。

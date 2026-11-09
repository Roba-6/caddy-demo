# caddy-demo

[One Coder の構築記事](https://github.com/Roba-6/one-coder-articles/blob/content/19-compose-caddy-static-https/articles/ja/20261109-docker-compose-caddy-static-https.md)で紹介している，**Docker Compose と Caddy で静的 Web サイトを配信するサンプル**です．まず `localhost` で HTTP 接続を確認し，ドメインと DNS を設定したら HTTPS で公開できます．

## ファイル構成

~~~text
caddy-demo/
├── .env.example     # .env のひな型（Git 管理）
├── .env             # 自分の設定（自分で作成・Git 管理外）
├── .gitignore
├── compose.yaml     # Docker Compose
├── Caddyfile        # Caddy の配信設定
└── site/
    └── index.html   # テスト用ページ
~~~

`site/` 配下のファイルが公開対象です．`site/index.html` はトップページになり，例えば `site/css/style.css` を作成すれば `/css/style.css` から参照できます．**公開してはいけない秘密情報は `site/` に置かないでください**．

`./site:/srv:ro` というバインドマウントにより，ホストの `site/` の内容をコンテナ内の `/srv` から読み取ります．HTML・CSS・画像の差し替えに Docker イメージの再ビルドや Caddy の再起動は不要です．これは**同じホスト上のディレクトリ共有**であり，作業用 PC とサーバー間の自動同期ではありません．

## 事前準備

Docker Engine と Docker Compose V2 をインストールした Linux サーバー等を使用します．最初の確認は TCP 80 番ポートを使うので，他の Web サーバーが同じポートを占有していないことを確かめてください．

~~~bash
docker --version
docker compose version
~~~

## 1．ローカルで HTTP 表示を確認する

リポジトリのディレクトリに移動したら，まず `.env.example` を `.env` にコピーします．初期値の `SITE_DOMAIN=http://localhost` は HTTPS 証明書を必要としない動作確認用の設定です．

~~~bash
cp .env.example .env
docker compose config -q
docker compose config --environment
docker compose up -d
docker compose ps
curl -I http://localhost/
curl -fsS http://localhost/ | grep 'CADDY_DEMO_OK'
~~~

**期待結果**：Caddy コンテナが起動し，`http://localhost/` が HTTP 200 を返し，`CADDY_DEMO_OK` が表示されます．エラー時は `docker compose logs --tail=100 caddy` で確認してください．

## 2．独自ドメインで HTTPS 公開する

公開するドメインの DNS A レコードをサーバーの公開 IPv4 アドレスへ向け，TCP 80・443 番ポートを外部から到達可能にします．IPv6 を使う場合は AAAA レコードも正しく設定します．

`.env` の内容を，**自分が管理する実際のドメイン**へ変更します．以下は説明用の値です．

~~~dotenv
SITE_DOMAIN=example.com
~~~

設定を変更した後，コンテナを再作成して新しいドメインを反映します．

~~~bash
docker compose config -q
docker compose up -d --force-recreate caddy
docker compose exec caddy printenv SITE_DOMAIN
docker compose logs --tail=100 caddy
~~~

次に `example.com` を実際のドメインへ置き換えて確認します．

~~~bash
curl -I http://example.com/
curl -I https://example.com/
curl -fsS https://example.com/ | grep 'CADDY_DEMO_OK'
~~~

**期待結果**：HTTP から HTTPS へリダイレクトされ，HTTPS では HTTP 200 とテストページの内容が返ります．Caddy が証明書を自動取得するには，ドメインの DNS 設定と外部ポートの到達性が必要です．`example.com` は自分のドメインではないため，そのまま公開設定に使用しないでください．

## 3．Web サイトのファイルを更新する

HTML の更新例です．新しい HTML をサーバー上の `~/new-index.html` に置いてある場合は，次のように差し替えます．

~~~bash
cp ~/new-index.html site/index.html
docker compose exec caddy ls -l /srv/index.html
curl -fsS https://example.com/
~~~

`example.com` は実際のドメインに変更してください．**サイトのファイル変更だけでは，Caddy の再起動は必要ありません**．

## 4．完成後の運用コマンド

以下は必要な操作だけを個別に実行します．上から順番にすべて実行する手順ではありません．

| 操作 | コマンド |
| --- | --- |
| 作成・起動 | `docker compose up -d caddy` |
| 停止 | `docker compose stop caddy` |
| 停止済みコンテナの起動 | `docker compose start caddy` |
| 再起動 | `docker compose restart caddy` |
| 稼働状態 | `docker compose ps` |
| ログ | `docker compose logs --tail=100 caddy` |
| Caddyfile の設定検証 | `docker compose exec caddy caddy validate --config /etc/caddy/Caddyfile --adapter caddyfile` |
| Caddyfile の再読み込み | `docker compose exec -w /etc/caddy caddy caddy reload` |
| .env・Compose の変更反映 | `docker compose up -d --force-recreate caddy` |

`.env` を変更しても `docker compose restart` だけでは既存コンテナの環境変数は更新されません．`--force-recreate` を使ってください．なお，`Caddyfile` をエディタでファイルごと置換保存した場合は，単一ファイルのバインドマウントが旧ファイルを参照し続けることがあります．その場合はコンテナを再作成して反映してください．

## TLS 証明書の運用

Caddy が TLS 証明書を自動取得・更新するため，通常，手動更新コマンドや定期再起動は不要です．証明書は Docker の名前付きボリューム `caddy_data:/data` に保存され，コンテナを再作成しても保持されます．**`docker compose down -v` はボリュームを削除するため，通常の停止操作には使わないでください**．

現在配信している証明書の有効期間を確認する例です（ドメインは置き換えてください）．

~~~bash
openssl s_client -connect example.com:443 -servername example.com \
  </dev/null 2>/dev/null | openssl x509 -noout -dates
~~~

自動更新に失敗する場合は，DNS，TCP 80・443 番の外部到達性，`docker compose logs caddy`，`caddy_data` ボリュームを確認します．

## 参考資料

- [One Coder の日本語記事](https://github.com/Roba-6/one-coder-articles/blob/content/19-compose-caddy-static-https/articles/ja/20261109-docker-compose-caddy-static-https.md)
- [Docker Compose と環境変数](https://docs.docker.com/compose/how-tos/environment-variables/variable-interpolation/)
- [Docker のバインドマウント](https://docs.docker.com/engine/storage/bind-mounts/)
- [Caddy の Docker Compose 実行例](https://caddyserver.com/docs/running)
- [Caddy の自動 HTTPS](https://caddyserver.com/docs/automatic-https)

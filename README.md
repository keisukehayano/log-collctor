# READEME

## Diesel_cliのインストール

1. postgresqlしか使わないので、オプション付き

- `cargo install diesel_cli --no-default-features --features postgres`

<br>

## DBセットアップ
1. PostgreSQLのサーバを立ち上げ
る

- `docker-compose up -d`

2. Dieselのために環境変数を設定する

### Linux,macOS

- `export DATABASE_URL=postgresql://postgres:password@localhost:5432/log-collector`

### Windows Powershell

- `set &env:DATABASE_URL='postgresql://postgres:password@localhost:5432/log-collector'`

<br>

## スキーマの作成

1. serverディレクトリに移動

- `cd server`

2. 下記コマンドで、logcollerDBが作られmigrationディレクトリが作られる

- `diesel setup`

3. スキーマ作成

- `diesel migration generate create_logs`

4. 作られたマイグレーションファイルの[up.sql][down.sql]に処理を記述する。

```sql:up.sql
-- Your SQL goes here
CREATE TABLE logs (
    id BIGSERIAL NOT NULL,
    user_agent VARCHAR NOT NULL,
    response_time INT NOT NULL,
    timestamp TIMESTAMP DEFAULT CURRENT_TIMESTAMP NOT NULL,
    PRIMARY KEY (id)
);
```

<br>

```sql:down.sql
DROP TABLE IF EXISTS logs;
```

<br>

5. マイグレーションを走らせる

- `diesel migration run`

上記コマンドでテーブルが作られ、自動で[schema.rs]に対応するRUSTのコードが生成される
また、[diesel.toml]というDieselのコンフィギレーションファイルも生成される

6. [down.sql]がちゃんと書けているかの確認。redoを行う

- `diesel migration redo`

上記コマンドは`diesel migration revert` -> `diesel migration run`の流れと一緒

<br>

## モデル定義

1. serverの[cargo.toml]に追記する

```toml:server/Cargo.toml
[dependencies.diesel]
features = ["postgres", "chrono", "r2d2"]
version = "1.4"
```

2. [.env]ファイルに環境変数を書く

- `DATABASE_URL=postgresql://postgres:password@localhost:5432/log-collector`

3. Dieselが内部で使っているマクロをまとめてインポートするために下記を[server.main.rs]に追加

```rust:server/main.rs
#[macro_use]
extern crate diesel;
```

```rust:server/main.rs
mod schema;
```

4. ビルドする

<br>

# serverのテスト

## POST

env_logerはRUST_LOG環境変数に設定された値でログ出力を変える

1. 下記コマンドで、デバッグ出力をonにしてサーバを起動する

- `RUST_LOG=server=debug cargo run`

curlコマンドで確認

```
curl -v -H 'Content-Type: application/json' \
-d '{"user_agent": "Mozilla", "response_time": 200}' \
localhost:3000/logs
```

レスポンス

```
*   Trying ::1...
* TCP_NODELAY set
* Connected to localhost (::1) port 3000 (#0)
> POST /logs HTTP/1.1
> Host: localhost:3000
> User-Agent: curl/7.64.1
> Accept: */*
> Content-Type: application/json
> Content-Length: 47
> 
* upload completely sent off: 47 out of 47 bytes
< HTTP/1.1 202 Accepted
< content-length: 0
< date: Sat, 13 Nov 2021 03:10:37 GMT
< 
* Connection #0 to host localhost left intact
* Closing connection 0
```

レスポンスが202ならOK

## DBの内容確認

下記のコマンドで、Dockerのcliクライアントを呼び出す

- `docker-compose exec postgresql psql -U postgres log-collector`

確認結果

```
 id | user_agent | response_time |         timestamp          
----+------------+---------------+----------------------------
  1 | Mozilla    |           200 | 2021-11-13 03:10:37.562896
(1 row)
```

POSTは成功

<br>

## GET

1. テストcsvファイルを用意する

```csv:test.csv
user_agent,response_time,timestamp
hogehoge,10,"2021-11-13T12:00:00.931320Z"
```

2. フィールド名'file', MIMEタイプ'text/csv'でtest.csvの内容をhttp://localhost:3000/csv にポストする

- `curl -F 'file=@test.csv;type=text/csv' http://localhost:3000/csv`

3. http://localhost:3000/csv にGETリクエストする

- `curl http://localhost:3000/csv`

4. 結果が帰ってきたらOK!
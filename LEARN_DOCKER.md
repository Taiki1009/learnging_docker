# Rails on Docker Compose (Rails + PostgreSQL)

このプロジェクトは、**ローカルに Ruby を入れずに** Docker Compose 上で Rails アプリを動かす最小構成のサンプルです。  
Rails の Welcome ページが表示されるところまでをゴールにしています。

---

## セットアップ手順

### 1. 必要ファイルの作成

プロジェクト直下に以下を作成してください。

#### Dockerfile
```Dockerfile
FROM ruby:3.3

RUN apt-get update -y && apt-get install -y --no-install-recommends \
  build-essential libpq-dev nodejs curl && \
  rm -rf /var/lib/apt/lists/*

WORKDIR /app

COPY Gemfile Gemfile.lock ./
RUN bundle install

COPY . .

ENTRYPOINT ["./bin/docker-entrypoint"]

CMD ["bash", "-lc", "bin/rails server -b 0.0.0.0 -p 3000"]
```

compose.yml
```yaml
services:
  web:
    build: .
    command: bash -lc "bin/rails server -b 0.0.0.0 -p 3000"
    volumes:
      - .:/app
      - bundle_cache:/usr/local/bundle
    ports:
      - "3000:3000"
    environment:
      DATABASE_HOST: db
      DATABASE_USER: postgres
      DATABASE_PASSWORD: postgres
      DATABASE_NAME: app_development
    depends_on:
      - db

  db:
    image: postgres:16
    environment:
      POSTGRES_USER: postgres
      POSTGRES_PASSWORD: postgres
    ports:
      - "5432:5432"
    volumes:
      - db_data:/var/lib/postgresql/data

volumes:
  db_data:
  bundle_cache:
```

.dockerignore
```bash
.git
log/*
tmp/*
node_modules
vendor/bundle
.bundle
```

Gemfile
```ruby
source "https://rubygems.org"
gem "rails", "~> 7.1"
Gemfile.lock
```
空ファイルでOK

bin/docker-entrypoint
```bash
#!/usr/bin/env bash
set -e

rm -f tmp/pids/server.pid 2>/dev/null || true

exec "$@"
```

実行権限を付与：
```bash
chmod +x bin/docker-entrypoint
```

2. Rails アプリを生成
```bash
# イメージのビルド
docker compose build

# Gem をインストール
docker compose run --rm --no-deps web bundle install

# Rails アプリ雛形を生成（Postgres用）
docker compose run --rm --no-deps web rails new . --force --database=postgresql

# 生成後に再度 Gem をインストール
docker compose run --rm --no-deps web bundle install
```

3. database.yml の修正
config/database.yml を以下に置き換え：
```yaml
default: &default
  adapter: postgresql
  encoding: unicode
  host: <%= ENV.fetch("DATABASE_HOST", "db") %>
  username: <%= ENV.fetch("DATABASE_USER", "postgres") %>
  password: <%= ENV.fetch("DATABASE_PASSWORD", "postgres") %>
  pool: <%= ENV.fetch("RAILS_MAX_THREADS", 5) %>

development:
  <<: *default
  database: <%= ENV.fetch("DATABASE_NAME", "app_development") %>

test:
  <<: *default
  database: app_test
```

4. サービス起動 & DB準備
```bash
# サービス起動（バックグラウンド）
docker compose up -d

# DB作成 & マイグレーション
docker compose run --rm web bin/rails db:prepare
```

5. 動作確認
ブラウザで`localhost:3000`を開くと Rails Welcome ページ が表示されます。

### よく使うコマンド

サービス起動：
```bash
docker compose up -d
```

サービス停止：
```bash
docker compose down
```

コンテナに一時ログイン：
```bash
docker compose run --rm web bash
```

Rails コマンド実行例：
```bash
docker compose run --rm web bin/rails console
docker compose run --rm web bin/rails db:migrate
```

ログ表示：
```bash
docker compose logs -f web
```

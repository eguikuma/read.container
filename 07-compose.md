---
layout: default
title: Compose
---

# [07-compose：Compose](#compose) {#compose}

## [はじめに](#introduction) {#introduction}

[06-security](../06-security/) まで、コンテナの主要な仕組みをすべて学びました

namespace と cgroup がコンテナを作り、ランタイムがそれを管理し、イメージがコンテナの中身を提供し、ネットワークとストレージがコンテナの通信とデータ永続化を実現し、セキュリティ機構がコンテナを保護します

しかし、ここまでの内容はすべて<strong>単一のコンテナ</strong>に関するものでした

実際のアプリケーションでは、複数のコンテナを組み合わせて使うことが一般的です

たとえば、Web アプリケーションを作るとき

- Web サーバー（nginx）のコンテナ
- アプリケーション（Python / Node.js 等）のコンテナ
- データベース（PostgreSQL / MySQL 等）のコンテナ

これらを1つずつ `docker run` で起動し、ネットワークやボリュームを手動で設定するのは面倒で間違いも起きやすくなります

<strong>Docker Compose</strong> は、複数のコンテナの構成を1つのファイルで定義し、まとめて管理するツールです

このトピックでは、Compose の仕組みと使い方を学びます

---

## [日常の例え](#everyday-analogy) {#everyday-analogy}

Docker Compose を「レストランのレシピ」に例えてみましょう

<strong>docker run（1つずつ指定）</strong>

料理を作るたびに、材料の分量、調理手順、火加減をすべて口頭で伝えるようなものです

正確に伝えるのが大変で、毎回同じ結果を再現するのが難しくなります

<strong>compose.yaml（定義ファイル）</strong>

すべてのメニューの<strong>レシピブック</strong>です

前菜、メインディッシュ、デザートの作り方がすべて書かれていて、「今日のコースを作って」と言うだけで全品が用意されます

レシピを見れば、誰が作っても同じ料理が再現できます

---

## [このページで学ぶこと](#what-you-will-learn) {#what-you-will-learn}

このページでは、以下の概念を学びます

<strong>複数コンテナの管理</strong>

- <strong>なぜ複数コンテナが必要か</strong>
  - マイクロサービスと関心の分離
- <strong>Docker Compose とは何か</strong>
  - 複数コンテナの定義・管理ツール

<strong>compose.yaml の構造</strong>

- <strong>サービス定義</strong>
  - image、build、ports、environment、volumes、depends_on
- <strong>ネットワーク定義</strong>
  - Compose が自動作成するブリッジネットワーク
- <strong>ボリューム定義</strong>
  - 名前付きボリュームの管理

<strong>ライフサイクル管理</strong>

- <strong>Compose コマンド</strong>
  - up / down / ps / logs による統合管理

---

## [目次](#table-of-contents) {#table-of-contents}

1. [なぜ複数コンテナが必要か](#why-multiple-containers)
2. [Docker Compose とは何か](#what-is-docker-compose)
3. [compose.yaml の構造](#compose-yaml-structure)
4. [サービス定義](#service-definition)
5. [Compose のネットワーク](#compose-network)
6. [Compose のボリューム](#compose-volume)
7. [Compose のライフサイクル管理](#compose-lifecycle)
8. [まとめ：このリポジトリで学んだこと](#final-summary)
9. [用語集](#glossary)
10. [参考資料](#references)

---

## [なぜ複数コンテナが必要か](#why-multiple-containers) {#why-multiple-containers}

### [関心の分離](#separation-of-concerns) {#separation-of-concerns}

コンテナは「1つのコンテナで1つの役割」という設計思想に基づいています

1つのコンテナに Web サーバー、アプリケーション、データベースをすべて入れることもできますが、以下の問題が生じます

{: .labeled}
| 問題 | 説明 |
| ---------------- | ---------------------------------------------------------------------- |
| 更新が困難 | アプリケーションだけを更新したいのに、コンテナ全体を作り直す必要がある |
| スケールが困難 | Web サーバーだけを増やしたいのに、データベースまで複製される |
| 障害の影響 | 1つのプロセスの障害が他のプロセスに影響する |
| イメージの肥大化 | すべてのソフトウェアを含むため、イメージが大きくなる |

各役割を別のコンテナに分けることで、それぞれを独立して更新、スケール、管理できます

### [複数コンテナの例](#multiple-containers-example) {#multiple-containers-example}

典型的な Web アプリケーションの構成例です

```
┌─────────┐     ┌─────────┐     ┌─────────┐
│  nginx  │────→│  app    │────→│  db     │
│ （Web） │     │（API）  │     │（DB）   │
└─────────┘     └─────────┘     └─────────┘
   :80             :3000           :5432
```

各コンテナは専用のネットワークで接続され、それぞれが独立して動作します

---

## [Docker Compose とは何か](#what-is-docker-compose) {#what-is-docker-compose}

<strong>Docker Compose</strong> は、複数のコンテナの構成を <strong>compose.yaml</strong>（または docker-compose.yml）ファイルで定義し、1つのコマンドでまとめて管理するツールです

### [Compose がない場合](#without-compose) {#without-compose}

複数のコンテナを手動で管理する必要があります

```
docker network create my-app
docker volume create db-data
docker run -d --name db --network my-app -v db-data:/var/lib/postgresql/data postgres:16
docker run -d --name app --network my-app -e DATABASE_URL=postgres://db:5432/mydb myapp
docker run -d --name web --network my-app -p 80:80 nginx
```

### [Compose がある場合](#with-compose) {#with-compose}

同じ構成を compose.yaml に記述し、1つのコマンドで管理できます

```
docker compose up -d
```

起動も停止も、すべてのコンテナがまとめて管理されます

### [Compose Specification](#compose-specification) {#compose-specification}

Docker Compose の設定ファイル形式は、<strong>Compose Specification</strong> として標準化されています

この仕様は Docker 社が主導していますが、Podman など他のツールでも互換性があります

---

## [compose.yaml の構造](#compose-yaml-structure) {#compose-yaml-structure}

compose.yaml は、以下の3つのセクションで構成されます

```yaml
services:
  # コンテナの定義

networks:
  # ネットワークの定義

volumes:
  # ボリュームの定義
```

### [実際の例](#example) {#example}

Web アプリケーション + データベースの構成例です

```yaml
services:
  web:
    image: nginx:1.27
    ports:
      - "80:80"
    volumes:
      - ./nginx.conf:/etc/nginx/nginx.conf:ro
    depends_on:
      - app

  app:
    build: ./app
    environment:
      - DATABASE_URL=postgres://db:5432/mydb
    depends_on:
      - db

  db:
    image: postgres:16
    volumes:
      - db-data:/var/lib/postgresql/data
    environment:
      - POSTGRES_DB=mydb
      - POSTGRES_PASSWORD=secret

volumes:
  db-data:
```

この compose.yaml は、3つのサービス（web、app、db）と1つのボリューム（db-data）を定義しています

---

## [サービス定義](#service-definition) {#service-definition}

services セクションでは、各コンテナの設定を<strong>サービス</strong>として定義します

### [主な設定項目](#main-config-options) {#main-config-options}

{: .labeled}
| 設定 | 説明 | 例 |
| ----------- | ------------------------------- | ---------------------------------- |
| image | 使用するコンテナイメージ | `image: nginx:1.27` |
| build | Dockerfile からイメージをビルド | `build: ./app` |
| ports | ポートマッピング | `ports: ["80:80"]` |
| environment | 環境変数 | `environment: [DATABASE_URL=...]` |
| volumes | ボリュームやバインドマウント | `volumes: [db-data:/var/lib/data]` |
| depends_on | 依存関係（起動順序） | `depends_on: [db]` |
| command | デフォルトコマンドの上書き | `command: ["python", "app.py"]` |
| restart | 再起動ポリシー | `restart: unless-stopped` |

### [image と build](#image-and-build) {#image-and-build}

<strong>image</strong> は既存のイメージを使用する場合に指定します

```yaml
services:
  web:
    image: nginx:1.27
```

<strong>build</strong> は Dockerfile からイメージをビルドする場合に指定します

```yaml
services:
  app:
    build: ./app
```

build と image の両方を指定すると、ビルドしたイメージに指定した名前が付けられます

### [depends_on](#depends-on) {#depends-on}

depends_on は、サービス間の<strong>起動順序</strong>を定義します

```yaml
services:
  app:
    depends_on:
      - db
  db:
    image: postgres:16
```

この場合、db が起動してから app が起動します

デフォルトでは、depends_on は「コンテナが起動したか」だけを確認します

「データベースが接続を受け付ける準備ができたか」は確認しません

`condition: service_healthy` を指定すればヘルスチェックの成功を待つこともできますが、アプリケーション側でもデータベースへの接続をリトライする仕組みを持つことが推奨されます

```yaml
services:
  app:
    depends_on:
      db:
        condition: service_healthy
  db:
    image: postgres:16
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U postgres"]
      interval: 5s
      timeout: 5s
      retries: 5
```

### [environment](#environment) {#environment}

environment は、コンテナに<strong>環境変数</strong>を設定します

```yaml
services:
  db:
    image: postgres:16
    environment:
      - POSTGRES_DB=mydb
      - POSTGRES_PASSWORD=secret
```

環境変数は、コンテナ内のアプリケーションの設定に使われます

データベースの接続先、API キー、動作モードなど、環境によって変わる設定を環境変数で管理するのが一般的です

### [restart](#restart) {#restart}

restart は、コンテナの<strong>再起動ポリシー</strong>を定義します

{: .labeled}
| ポリシー | 説明 |
| -------------- | ------------------------------ |
| no | 再起動しない（デフォルト） |
| always | 常に再起動する |
| on-failure | 異常終了時のみ再起動する |
| unless-stopped | 手動で停止しない限り再起動する |

---

## [Compose のネットワーク](#compose-network) {#compose-network}

### [デフォルトネットワーク](#default-network) {#default-network}

Docker Compose は、プロジェクトごとに<strong>デフォルトのブリッジネットワーク</strong>を自動作成します

compose.yaml が置かれたディレクトリ名がプロジェクト名として使われ、ネットワーク名は `<プロジェクト名>_default` になります

```
プロジェクトディレクトリ: my-app/

作成されるネットワーク: my-app_default

  my-app_default（bridge）
      │
      ├── web（nginx）
      ├── app（myapp）
      └── db（postgres）
```

### [サービス名による名前解決](#service-name-resolution) {#service-name-resolution}

[04-network](../04-network/) で学んだユーザー定義 bridge の DNS 機能が、Compose でも使われます

各サービスは、サービス名（compose.yaml で定義した名前）で他のサービスにアクセスできます

```yaml
services:
  app:
    environment:
      - DATABASE_URL=postgres://db:5432/mydb
```

この例では、app サービスから `db` というホスト名でデータベースにアクセスしています

Docker の内部 DNS サーバーが `db` を db コンテナの IP アドレスに解決します

### [カスタムネットワーク](#custom-network) {#custom-network}

必要に応じて、カスタムネットワークを定義してサービス間の通信を制御できます

```yaml
services:
  web:
    networks:
      - frontend

  app:
    networks:
      - frontend
      - backend

  db:
    networks:
      - backend

networks:
  frontend:
  backend:
```

この例では、web と app は frontend ネットワークで通信し、app と db は backend ネットワークで通信します

web と db は異なるネットワークに属するため、直接通信できません

これにより、外部に公開する web から直接 db にアクセスされることを防ぐ、セキュリティ上の効果があります

---

## [Compose のボリューム](#compose-volume) {#compose-volume}

### [名前付きボリューム](#named-volume) {#named-volume}

compose.yaml でボリュームを定義すると、Docker がボリュームを管理します

```yaml
services:
  db:
    image: postgres:16
    volumes:
      - db-data:/var/lib/postgresql/data

volumes:
  db-data:
```

`volumes:` セクションでボリュームを宣言し、サービスの `volumes:` でマウント先を指定します

### [バインドマウント](#bind-mount) {#bind-mount}

compose.yaml でバインドマウントも指定できます

```yaml
services:
  web:
    image: nginx:1.27
    volumes:
      - ./nginx.conf:/etc/nginx/nginx.conf:ro
```

`./` で始まるパスはバインドマウントとして扱われます

`:ro` は読み取り専用（read-only）を意味し、コンテナからの変更を防ぎます

### [ボリュームのライフサイクル](#volume-lifecycle) {#volume-lifecycle}

`docker compose down` でコンテナを停止・削除しても、<strong>ボリュームは保持されます</strong>

```
docker compose down       ← コンテナとネットワークを削除。ボリュームは残る
docker compose down -v    ← コンテナ、ネットワーク、ボリュームをすべて削除
```

データベースのデータなど永続化したいデータは名前付きボリュームを使います

---

## [Compose のライフサイクル管理](#compose-lifecycle) {#compose-lifecycle}

Docker Compose は、複数のコンテナを1つのコマンドでまとめて管理します

### [主要なコマンド](#main-commands) {#main-commands}

{: .labeled}
| コマンド | 説明 |
| ------------------------ | ------------------------------ |
| `docker compose up` | サービスを作成して起動する |
| `docker compose up -d` | バックグラウンドで起動する |
| `docker compose down` | サービスを停止して削除する |
| `docker compose ps` | 実行中のサービス一覧を表示する |
| `docker compose logs` | サービスのログを表示する |
| `docker compose logs -f` | ログをリアルタイムで表示する |
| `docker compose build` | サービスのイメージをビルドする |
| `docker compose restart` | サービスを再起動する |

### [up の動作](#up-command-behavior) {#up-command-behavior}

`docker compose up` を実行すると、以下の処理が順番に行われます

{: .labeled}
| 順番 | 処理 |
| ---- | ----------------------------------------------- |
| 1 | compose.yaml を読み込む |
| 2 | デフォルトネットワークを作成する |
| 3 | 定義されたボリュームを作成する |
| 4 | depends_on に従ってサービスの起動順序を決定する |
| 5 | 各サービスのイメージを pull またはビルドする |
| 6 | 各サービスのコンテナを作成・起動する |

### [down の動作](#down-command-behavior) {#down-command-behavior}

`docker compose down` を実行すると、以下の処理が行われます

{: .labeled}
| 順番 | 処理 |
| ---- | --------------------------------------------- |
| 1 | すべてのコンテナを停止する |
| 2 | すべてのコンテナを削除する |
| 3 | デフォルトネットワークを削除する |
| 4 | ボリュームは削除しない（-v を指定しない限り） |

---

## [まとめ：このリポジトリで学んだこと](#final-summary) {#final-summary}

このリポジトリでは、「コンテナはどう動き、どう使うか」を学びました

### [学んだトピック](#learned-topics) {#learned-topics}

{: .labeled}
| トピック | 内容 |
| ------------------ | ---------------------------------------------------------------------------- |
| 01-container | コンテナの正体は namespace で隔離され cgroup で制限されたプロセス |
| 02-oci-and-runtime | OCI 仕様がコンテナを標準化し、runc / containerd / Docker / Podman が管理する |
| 03-image | レイヤ構造と overlay filesystem でイメージが効率的に構成される |
| 04-network | bridge、veth ペア、ポートマッピングでコンテナが通信する |
| 05-storage | ボリュームとバインドマウントでデータを永続化する |
| 06-security | capabilities、seccomp、rootless の多層防御でコンテナを保護する |
| 07-compose | compose.yaml で複数コンテナを定義し、まとめて管理する |

`docker compose up` を実行したとき、何が起きているかを最初から最後まで説明できるようになりました

1. compose.yaml を読み込み、サービス構成を解析する（07-compose）
2. ネットワーク（bridge）を作成する（04-network）
3. ボリュームを作成する（05-storage）
4. 各サービスのイメージを pull / ビルドする（03-image）
5. Docker Engine が containerd に指示し、runc がコンテナを作成する（02-oci-and-runtime）
6. namespace を作成し、cgroup を設定し、プロセスを起動する（01-container）
7. capabilities と seccomp でセキュリティを適用する（06-security）

---

## [用語集](#glossary) {#glossary}

{: .labeled}
| 用語 | 説明 |
| ---------------------- | ------------------------------------------------------------------------------------------- |
| Docker Compose | 複数のコンテナの構成を定義し、まとめて管理するツール |
| compose.yaml | Docker Compose の設定ファイル。サービス、ネットワーク、ボリュームを定義する |
| Compose Specification | compose.yaml の形式を標準化する仕様 |
| サービス | compose.yaml で定義されるコンテナの設定単位 |
| プロジェクト | Docker Compose が管理する一連のサービス、ネットワーク、ボリュームの集合 |
| depends_on | サービス間の起動順序の依存関係を定義する設定 |
| デフォルトネットワーク | Docker Compose がプロジェクトごとに自動作成するブリッジネットワーク |
| 名前付きボリューム | compose.yaml の volumes セクションで定義される、Docker が管理するボリューム |
| docker compose up | サービスを作成して起動するコマンド |
| docker compose down | サービスを停止して削除するコマンド |
| 再起動ポリシー | コンテナが終了した際の再起動動作を定義する設定（no / always / on-failure / unless-stopped） |
| 環境変数 | コンテナ内のアプリケーションに渡す設定値 |
| 関心の分離 | 各コンテナが1つの役割を担い、独立して管理できるようにする設計原則 |

---

## [参考資料](#references) {#references}

このページの内容は、以下のソースに基づいています

<strong>Compose 仕様</strong>

- [Compose Specification](https://github.com/compose-spec/compose-spec/blob/main/spec.md){:target="\_blank"}
  - Docker Compose の設定ファイル形式の標準仕様

<strong>Docker Compose</strong>

- [Docker Compose overview](https://docs.docker.com/compose/){:target="\_blank"}
  - Docker Compose の概要と基本的な使い方
- [Compose file reference](https://docs.docker.com/reference/compose-file/){:target="\_blank"}
  - compose.yaml の全設定項目のリファレンス

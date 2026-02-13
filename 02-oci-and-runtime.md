---
layout: default
title: OCI とランタイム
---

# [02-oci-and-runtime：OCI とランタイム](#oci-and-runtime) {#oci-and-runtime}

## [はじめに](#introduction) {#introduction}

[01-container](../01-container/) では、コンテナの正体が「namespace で隔離され、cgroup で制限されたプロセス」であることを学びました

そして、namespace と cgroup だけではコンテナにならないことも学びました

- namespace の作成
- cgroup の設定
- ファイルシステムのマウント
- ネットワークの接続

これらすべてを<strong>まとめて管理する</strong>ソフトウェアが必要です

それが<strong>コンテナランタイム</strong>です

しかし、コンテナランタイムは1つだけではありません

Docker、Podman、containerd、runc など、さまざまなソフトウェアが「コンテナランタイム」と呼ばれています

これらはどう違い、どう連携しているのでしょうか？

そして、異なるランタイム間でコンテナの互換性を保つために、<strong>OCI</strong>（Open Container Initiative）という標準仕様が存在します

このトピックでは、OCI 仕様とコンテナランタイムの全体像を学びます

---

## [日常の例え](#everyday-analogy) {#everyday-analogy}

コンテナランタイムの階層構造を「建設業界」に例えてみましょう

<strong>OCI 仕様</strong>

<strong>建築基準法</strong>のようなものです

「建物はこういう構造でなければならない」「こういう手順で建てなければならない」という標準規格を定めています

どの建設会社が建てても、この基準を満たしていれば安全です

<strong>runc（低レベルランタイム）</strong>

実際に建物を建てる<strong>職人</strong>です

設計図（config.json）を受け取り、基礎工事（namespace の作成）、配管工事（cgroup の設定）など、実際の作業を行います

<strong>containerd（高レベルランタイム）</strong>

<strong>現場監督</strong>のようなものです

設計図の準備、資材（イメージ）の調達、職人（runc）への指示、工事の進捗管理など、プロジェクト全体を管理します

<strong>Docker / Podman</strong>

<strong>建設会社</strong>のようなものです

お客さん（ユーザー）の要望を聞き、現場監督に指示を出します

Docker は大きな会社（デーモン常駐型）、Podman は小さな工務店（必要な時だけ動く）のような違いがあります

---

## [このページで学ぶこと](#what-you-will-learn) {#what-you-will-learn}

このページでは、以下の概念を学びます

<strong>OCI 仕様</strong>

- <strong>OCI（Open Container Initiative）</strong>
  - コンテナの標準仕様を策定する組織と、その仕様の概要
- <strong>OCI Runtime Specification</strong>
  - コンテナのライフサイクルと設定ファイル（config.json）の構造
- <strong>OCI Image Specification / Distribution Specification</strong>
  - イメージの構造と配布方法の標準（概要のみ、詳細は 03-image で学ぶ）

<strong>コンテナランタイム</strong>

- <strong>runc</strong>
  - namespace と cgroup を実際に操作する低レベルランタイム
- <strong>containerd</strong>
  - イメージ管理やコンテナ管理を担う高レベルランタイム
- <strong>Docker のアーキテクチャ</strong>
  - Docker CLI → dockerd → containerd → runc の連携
- <strong>Podman のアーキテクチャ</strong>
  - デーモンレスで動作する代替ランタイム

---

## [目次](#table-of-contents) {#table-of-contents}

1. [OCI とは何か](#what-is-oci)
2. [OCI Runtime Specification](#oci-runtime-spec)
3. [OCI Image Specification と Distribution Specification](#oci-image-and-distribution-spec)
4. [低レベルランタイム：runc](#low-level-runtime-runc)
5. [高レベルランタイム：containerd](#high-level-runtime-containerd)
6. [Docker のアーキテクチャ](#docker-architecture)
7. [Podman のアーキテクチャ](#podman-architecture)
8. [Docker と Podman の比較](#docker-vs-podman)
9. [次のトピックへ](#next-topic)
10. [用語集](#glossary)
11. [参考資料](#references)

---

## [OCI とは何か](#what-is-oci) {#what-is-oci}

<strong>OCI</strong>（Open Container Initiative）は、コンテナの標準仕様を策定する組織です

2015 年に Docker 社と CoreOS 社を中心に設立されました

### [なぜ標準仕様が必要なのか](#why-standard-spec) {#why-standard-spec}

コンテナ技術の初期には、Docker が事実上の標準でした

しかし、Docker 以外のランタイム（rkt、LXC など）も登場し、互換性の問題が生じました

「Docker で作ったコンテナが別のランタイムで動かない」という状況を防ぐために、コンテナの<strong>標準仕様</strong>が必要になりました

### [OCI の3つの仕様](#oci-three-specs) {#oci-three-specs}

OCI は以下の3つの仕様を策定しています

{: .labeled}
| 仕様 | 内容 | 定義するもの |
| -------------------------- | ---------------------- | -------------------------------------------- |
| Runtime Specification | コンテナの実行方法 | ライフサイクル、設定ファイルの形式、実行環境 |
| Image Specification | コンテナイメージの構造 | レイヤ構造、メタデータ、マニフェスト |
| Distribution Specification | イメージの配布方法 | レジストリとの通信プロトコル |

これらの仕様に準拠していれば、どのランタイムで作ったコンテナでも、どのランタイムで実行できます

---

## [OCI Runtime Specification](#oci-runtime-spec) {#oci-runtime-spec}

OCI Runtime Specification は、コンテナの<strong>実行方法</strong>を標準化する仕様です

この仕様は、大きく分けて2つのことを定義しています

### [config.json：コンテナの設定ファイル](#config-json) {#config-json}

コンテナをどのように構成するかを記述する JSON ファイルです

runc などの低レベルランタイムは、この config.json を読み取ってコンテナを作成します

```json
{
  "ociVersion": "1.0.0",
  "process": {
    "args": ["/bin/sh"],
    "cwd": "/",
    "env": ["PATH=/usr/bin:/bin"]
  },
  "root": {
    "path": "rootfs",
    "readonly": true
  },
  "linux": {
    "namespaces": [
      { "type": "pid" },
      { "type": "network" },
      { "type": "mount" },
      { "type": "uts" },
      { "type": "ipc" }
    ]
  }
}
```

<strong>process</strong> セクション

コンテナ内で実行するプロセスの設定です

実行するコマンド（args）、作業ディレクトリ（cwd）、環境変数（env）などを指定します

<strong>root</strong> セクション

コンテナのルートファイルシステムのパスを指定します

rootfs ディレクトリに、コンテナのファイルシステムが展開されます

<strong>linux.namespaces</strong> セクション

作成する namespace の種類を指定します

この例では、PID、Network、Mount、UTS、IPC の 5 つの namespace を作成します

### [コンテナのライフサイクル](#container-lifecycle-spec) {#container-lifecycle-spec}

OCI Runtime Specification は、コンテナの状態遷移も定義しています

[01-container](../01-container/) で学んだライフサイクル（Creating → Created → Running → Stopped）は、この仕様に基づいています

{: .labeled}
| 状態 | 説明 |
| -------- | ---------------------------------------------------- |
| Creating | コンテナの作成中（namespace と cgroup の設定中） |
| Created | コンテナが作成された（プロセスはまだ開始していない） |
| Running | コンテナ内のプロセスが実行中 |
| Stopped | コンテナ内のプロセスが終了した |

この標準的なライフサイクルにより、異なるランタイム間で一貫した動作が保証されます

---

## [OCI Image Specification と Distribution Specification](#oci-image-and-distribution-spec) {#oci-image-and-distribution-spec}

### [OCI Image Specification](#oci-image-spec) {#oci-image-spec}

コンテナイメージの構造を標準化する仕様です

イメージがどのようなレイヤで構成され、どのようなメタデータを持つかを定義しています

この仕様の詳細は [03-image](../03-image/) で学びます

### [OCI Distribution Specification](#oci-distribution-spec) {#oci-distribution-spec}

コンテナイメージの配布方法を標準化する仕様です

コンテナレジストリ（Docker Hub 等）との通信プロトコルを定義しています

この仕様により、Docker で push したイメージを Podman で pull する、といった相互運用が可能になります

---

## [低レベルランタイム：runc](#low-level-runtime-runc) {#low-level-runtime-runc}

<strong>runc</strong> は、OCI Runtime Specification に準拠した<strong>低レベルコンテナランタイム</strong>です

もともと Docker の内部コンポーネントだったものが、OCI プロジェクトとして独立しました

### [runc の役割](#runc-role) {#runc-role}

runc は config.json を受け取り、以下の操作を実行します

1. config.json に従って namespace を作成する
2. cgroup を設定してリソースを制限する
3. ルートファイルシステムをマウントする
4. セキュリティ設定を適用する（capabilities、seccomp 等）
5. コンテナ内のプロセスを起動する

### [なぜ「低レベル」と呼ばれるのか](#why-low-level) {#why-low-level}

runc は<strong>コンテナの実行だけ</strong>を担当します

イメージの取得（pull）、イメージの展開、ネットワークの設定、コンテナの管理（一覧表示、ログ取得等）は runc の責任範囲外です

そのため、通常は runc を直接使うことはなく、containerd のような高レベルランタイムから呼び出されます

### [runc の動作の流れ](#runc-operation-flow) {#runc-operation-flow}

```
config.json + rootfs ディレクトリ
        │
        ↓
      runc
        │
        ├── namespace を作成
        ├── cgroup を設定
        ├── rootfs をマウント
        ├── セキュリティを設定
        └── プロセスを起動
        │
        ↓
  コンテナ内のプロセスが実行される
```

---

## [高レベルランタイム：containerd](#high-level-runtime-containerd) {#high-level-runtime-containerd}

<strong>containerd</strong> は、コンテナのライフサイクル全体を管理する<strong>高レベルコンテナランタイム</strong>です

もともと Docker の内部コンポーネントだったものが、独立したプロジェクトとして分離されました

### [containerd の役割](#containerd-role) {#containerd-role}

containerd は、runc が担当しない部分をカバーします

{: .labeled}
| 機能 | 説明 |
| ------------------ | ---------------------------------------------------- |
| イメージ管理 | イメージの pull / push、レイヤの展開、ストレージ管理 |
| コンテナ管理 | コンテナの作成、起動、停止、削除 |
| タスク管理 | コンテナ内のプロセスの監視と管理 |
| スナップショット | イメージレイヤからルートファイルシステムを準備 |
| ランタイム呼び出し | runc を呼び出してコンテナを実際に起動 |

### [containerd と runc の関係](#containerd-and-runc-relationship) {#containerd-and-runc-relationship}

```
containerd
    │
    ├── イメージを pull
    ├── レイヤを展開して rootfs を準備
    ├── config.json を生成
    │
    └── runc を呼び出し
            │
            ├── namespace を作成
            ├── cgroup を設定
            └── プロセスを起動
```

containerd はイメージの取得からルートファイルシステムの準備までを行い、実際のコンテナ作成は runc に委譲します

### [containerd のデーモン](#containerd-daemon) {#containerd-daemon}

containerd は<strong>デーモン</strong>（常駐プロセス）として動作します

gRPC API を提供し、Docker や Kubernetes などの上位ツールからの要求を受け付けます

---

## [Docker のアーキテクチャ](#docker-architecture) {#docker-architecture}

Docker は、最も広く使われているコンテナプラットフォームです

ユーザーが `docker run` と入力してからコンテナが起動するまで、複数のコンポーネントが連携します

### [コンポーネントの階層](#docker-component-hierarchy) {#docker-component-hierarchy}

```
ユーザー
  │
  │ docker run nginx
  │
  ↓
Docker CLI（docker コマンド）
  │
  │ REST API
  │
  ↓
Docker Engine（dockerd）
  │
  │ gRPC API
  │
  ↓
containerd
  │
  │ OCI Runtime 呼び出し
  │
  ↓
runc
  │
  ↓
コンテナ（namespace + cgroup + rootfs）
```

### [各コンポーネントの役割](#docker-component-roles) {#docker-component-roles}

<strong>Docker CLI</strong>

ユーザーが操作するコマンドラインツールです

`docker run`、`docker build`、`docker ps` などのコマンドを提供します

Docker CLI は REST API を使って Docker Engine にリクエストを送ります

<strong>Docker Engine（dockerd）</strong>

Docker のメインデーモンです

イメージのビルド、ネットワークの管理、ボリュームの管理など、Docker 固有の機能を提供します

コンテナの実行は containerd に委譲します

<strong>containerd</strong>

イメージの管理とコンテナのライフサイクル管理を担当します

実際のコンテナ作成は runc に委譲します

<strong>runc</strong>

OCI 仕様に従ってコンテナを作成・実行します

### [docker run の流れ](#docker-run-flow) {#docker-run-flow}

`docker run nginx` を実行したとき、内部では以下のことが起きます

{: .labeled}
| 順番 | コンポーネント | 処理 |
| ---- | -------------- | ------------------------------------------------------------ |
| 1 | Docker CLI | コマンドを解析し、Docker Engine に REST API リクエストを送信 |
| 2 | Docker Engine | イメージのチェック（ローカルになければ pull を指示） |
| 3 | containerd | イメージの pull とレイヤの展開 |
| 4 | containerd | ルートファイルシステムの準備と config.json の生成 |
| 5 | runc | namespace の作成、cgroup の設定（コンテナは created 状態） |
| 6 | Docker Engine | ネットワークの設定（bridge、veth ペアの作成） |
| 7 | runc | コンテナ内のプロセスを起動（コンテナは running 状態） |

---

## [Podman のアーキテクチャ](#podman-architecture) {#podman-architecture}

<strong>Podman</strong> は、Docker と CLI 互換のコンテナランタイムです

Docker との最大の違いは、<strong>デーモンが不要</strong>（デーモンレス）であることです

### [コンポーネントの階層](#podman-component-hierarchy) {#podman-component-hierarchy}

```
ユーザー
  │
  │ podman run nginx
  │
  ↓
Podman
  │
  ↓
conmon（コンテナモニター）
  │
  │ OCI Runtime 呼び出し
  │
  ↓
runc
  │
  ↓
コンテナ（namespace + cgroup + rootfs）
```

### [Docker との構造の違い](#difference-from-docker) {#difference-from-docker}

<strong>デーモンレス</strong>

Docker は dockerd というデーモンが常に動いている必要があります

Podman はデーモンを持たず、コマンド実行時に直接コンテナを管理します

<strong>conmon（コンテナモニター）</strong>

Podman はコンテナごとに <strong>conmon</strong> という小さなプロセスを起動します

conmon は以下の役割を持ちます

- コンテナプロセスの監視（終了の検知）
- ログの収集
- ターミナルの管理（attach 時）

Podman 自体はコンテナの起動後に終了してもよく、conmon がコンテナを見守り続けます

<strong>fork/exec モデル</strong>

Docker では、すべてのコンテナが dockerd を頂点とする階層構造（dockerd → containerd → containerd-shim → コンテナ）で管理されます

containerd-shim は containerd とコンテナプロセスの間に位置する小さなプロセスで、コンテナプロセスの親として動作し、containerd が再起動してもコンテナを動かし続ける役割を持ちます

Podman では、各コンテナは Podman コマンドから fork/exec されます

これにより、デーモンの単一障害点がなくなります

### [rootless がデフォルト](#rootless-by-default) {#rootless-by-default}

Podman は<strong>rootless モード</strong>（root 権限なしでのコンテナ実行）をデフォルトでサポートしています

User namespace を使い、一般ユーザーがコンテナを実行できます

Docker も rootless モードをサポートしていますが、追加の設定が必要です

---

## [Docker と Podman の比較](#docker-vs-podman) {#docker-vs-podman}

{: .labeled}
| 項目 | Docker | Podman |
| ------------------ | ------------------------------------------- | ----------------------------------------------------------------------- |
| デーモン | 必要（dockerd が常駐） | 不要（デーモンレス） |
| CLI | docker コマンド | podman コマンド（Docker 互換） |
| rootless | 追加設定が必要 | デフォルトでサポート |
| コンテナ監視 | dockerd が全コンテナを管理 | conmon がコンテナごとに監視 |
| 低レベルランタイム | runc（containerd 経由） | runc（直接呼び出し） |
| 高レベルランタイム | containerd | Podman 自体が担当 |
| Pod サポート | なし（複数コンテナの管理は Compose で行う） | Kubernetes 互換の Pod（関連するコンテナをグループ化する単位）をサポート |
| OCI 準拠 | はい | はい |

### [デーモンの有無がもたらす違い](#daemon-presence-difference) {#daemon-presence-difference}

<strong>Docker（デーモンあり）</strong>

- dockerd が停止すると、新しいコンテナの起動や既存コンテナの操作ができなくなる（ただし、live-restore 機能を有効にしている場合、実行中のコンテナは containerd-shim の下で動作し続ける）
- dockerd は root 権限で動作するため、Docker ソケットへのアクセスは root 権限の取得と同等になりうる
- API サーバーとして動作するため、リモートからの操作が容易

<strong>Podman（デーモンなし）</strong>

- デーモンがないため、単一障害点がない
- 一般ユーザー権限で動作するため、セキュリティリスクが低い
- systemd との統合により、コンテナをシステムサービスとして管理できる

### [CLI の互換性](#cli-compatibility) {#cli-compatibility}

Podman は Docker との CLI 互換性を重視しています

ほとんどの Docker コマンドは、`docker` を `podman` に置き換えるだけで動作します

```
docker run nginx    →    podman run nginx
docker ps           →    podman ps
docker build .      →    podman build .
```

---

## [次のトピックへ](#next-topic) {#next-topic}

このトピックでは、以下のことを学びました

- OCI がコンテナの標準仕様（Runtime / Image / Distribution）を定義している
- runc が低レベルランタイムとして namespace / cgroup を直接操作する
- containerd が高レベルランタイムとしてイメージ管理やコンテナ管理を担当する
- Docker は dockerd → containerd → runc の階層構造で動作する
- Podman はデーモンレスで、rootless をデフォルトサポートする

これで「コンテナを<strong>誰が管理するか</strong>」が分かりました

しかし、まだ大きな疑問が残っています

ランタイムが実行する<strong>コンテナの中身</strong>はどこから来るのでしょうか？

`docker run nginx` の `nginx` とは何でしょうか？

次のトピック [03-image](../03-image/) では、<strong>コンテナイメージ</strong>の仕組みを学びます

レイヤ構造、overlay filesystem、レジストリ、ビルドプロセスを理解することで、コンテナの中身がどう作られ、どう配布されるかが分かります

---

## [用語集](#glossary) {#glossary}

{: .labeled}
| 用語 | 説明 |
| -------------------------------- | ----------------------------------------------------------------------------------------- |
| OCI（Open Container Initiative） | コンテナの標準仕様を策定する組織。2015 年に設立 |
| OCI Runtime Specification | コンテナの実行方法（ライフサイクル、config.json）を定義する仕様 |
| OCI Image Specification | コンテナイメージの構造（レイヤ、メタデータ）を定義する仕様 |
| OCI Distribution Specification | コンテナイメージの配布方法（レジストリとの通信）を定義する仕様 |
| config.json | OCI Runtime Specification で定義される、コンテナの設定ファイル |
| コンテナランタイム | コンテナのライフサイクルを管理するソフトウェアの総称 |
| 低レベルランタイム | namespace と cgroup を直接操作してコンテナを作成・実行するランタイム（runc 等） |
| 高レベルランタイム | イメージ管理やコンテナ管理を含む、より広い機能を持つランタイム（containerd 等） |
| runc | OCI 準拠の低レベルコンテナランタイム。Docker から独立したプロジェクト |
| containerd | コンテナのライフサイクル全体を管理する高レベルランタイム。Docker から独立したプロジェクト |
| Docker CLI | Docker のコマンドラインツール。REST API で Docker Engine と通信する |
| Docker Engine（dockerd） | Docker のメインデーモン。コンテナ、イメージ、ネットワーク、ボリュームを管理する |
| Podman | Docker 互換のデーモンレスコンテナランタイム |
| conmon | Podman が使用するコンテナモニター。コンテナプロセスの監視とログ収集を行う |
| デーモン | バックグラウンドで常時動作するプロセス |
| デーモンレス | デーモンを必要としないアーキテクチャ |
| gRPC | Google が開発した高性能な RPC（Remote Procedure Call）フレームワーク |
| REST API | HTTP ベースの API 設計スタイル |
| rootless | root 権限なしで動作するモード |
| 単一障害点 | その機能が停止するとシステム全体に影響が及ぶ箇所 |

---

## [参考資料](#references) {#references}

このページの内容は、以下のソースに基づいています

<strong>OCI 仕様</strong>

- [OCI Runtime Specification](https://github.com/opencontainers/runtime-spec/blob/main/spec.md){:target="\_blank"}
  - コンテナランタイムの標準仕様（ライフサイクル、config.json の構造）
- [OCI Image Specification](https://github.com/opencontainers/image-spec/blob/main/spec.md){:target="\_blank"}
  - コンテナイメージの標準仕様（レイヤ構造、マニフェスト）
- [OCI Distribution Specification](https://github.com/opencontainers/distribution-spec/blob/main/spec.md){:target="\_blank"}
  - イメージ配布の標準仕様（レジストリとの通信プロトコル）

<strong>runc</strong>

- [runc - GitHub](https://github.com/opencontainers/runc){:target="\_blank"}
  - OCI 準拠の低レベルコンテナランタイム

<strong>containerd</strong>

- [containerd - Getting Started](https://containerd.io/docs/getting-started/){:target="\_blank"}
  - containerd の概要とアーキテクチャ

<strong>Docker</strong>

- [Docker overview](https://docs.docker.com/get-started/overview/){:target="\_blank"}
  - Docker のアーキテクチャ（CLI → dockerd → containerd → runc）

<strong>Podman</strong>

- [What is Podman?](https://docs.podman.io/en/latest/){:target="\_blank"}
  - Podman の概要とデーモンレスアーキテクチャ

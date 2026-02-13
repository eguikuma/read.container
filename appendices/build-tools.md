<div align="right">
<img src="https://img.shields.io/badge/AI-ASSISTED_STUDY-3b82f6?style=for-the-badge&labelColor=1e293b&logo=bookstack&logoColor=white" alt="AI Assisted Study" />
</div>

# appendix：ビルドツール

## はじめに

[03-image](../03-image.md) では、Dockerfile と `docker build` でコンテナイメージをビルドする仕組みを学びました

[alternative-runtimes](./alternative-runtimes.md) では、コンテナランタイムに Docker 以外の選択肢があることを学びました

イメージのビルドにも、同様に複数のツールが存在します

この補足資料では、主要なビルドツールの特徴と仕組みの違いを紹介します

---

## ビルドツールの分類

```
Docker デーモン連携型
├── classic builder（Docker の旧ビルダー）
└── BuildKit（Docker の標準ビルダー）

デーモンレス型
├── Buildah
└── Kaniko
```

---

## classic builder と BuildKit

Docker のビルドエンジンには<strong>classic builder</strong>（旧ビルダー）と<strong>BuildKit</strong> の 2 つがあります

BuildKit は Docker 23.0 以降でデフォルトのビルダーとなっています

### classic builder の動作

classic builder は Dockerfile の命令を<strong>上から順番に 1 つずつ</strong>処理します

```
FROM debian → RUN apt-get update → RUN apt-get install → COPY . . → RUN make
                 ↓                    ↓                    ↓          ↓
              レイヤ 1             レイヤ 2             レイヤ 3   レイヤ 4
              （順次実行）
```

各命令の完了を待ってから次の命令を実行するため、依存関係のない命令も並列化されません

### BuildKit の動作

BuildKit は Dockerfile の命令を<strong>DAG</strong>（Directed Acyclic Graph：有向非巡回グラフ）として解析します

命令間の依存関係を分析し、依存関係のない命令を<strong>並列に実行</strong>します

マルチステージビルド（[multi-stage-build](./multi-stage-build.md)）では、独立したステージが並列にビルドされます

### BuildKit の主な改善点

| 機能                   | classic builder                        | BuildKit                                |
| ---------------------- | -------------------------------------- | --------------------------------------- |
| ビルドの並列化         | なし（順次実行のみ）                   | DAG ベースの並列実行                    |
| キャッシュマウント     | なし                                   | --mount=type=cache でキャッシュを永続化 |
| シークレットの受け渡し | 環境変数で渡す（イメージに残る危険性） | --mount=type=secret で安全に渡す        |
| 未使用ステージ         | すべてのステージをビルド               | 最終イメージに不要なステージをスキップ  |

### キャッシュマウント

BuildKit の<strong>キャッシュマウント</strong>は、ビルド間で再利用できるキャッシュディレクトリを提供します

```dockerfile
RUN --mount=type=cache,target=/root/.cache/pip \
    pip install -r requirements.txt
```

この例では、pip のキャッシュディレクトリ（`/root/.cache/pip`）がビルド間で永続化されます

パッケージの再ダウンロードが不要になり、ビルドが高速化されます

キャッシュマウントの内容は最終イメージのレイヤには含まれません

### シークレットマウント

BuildKit の<strong>シークレットマウント</strong>は、ビルド時に秘密情報を安全に渡す仕組みです

```dockerfile
RUN --mount=type=secret,id=github_token \
    GITHUB_TOKEN=$(cat /run/secrets/github_token) go mod download
```

シークレットはビルド時に一時的にマウントされるだけで、イメージのレイヤには記録されません

---

## Buildah

<strong>Buildah</strong> は、Podman エコシステムの一部として開発されたビルドツールです

[alternative-runtimes](./alternative-runtimes.md) で学んだ Podman が「コンテナの実行」を担うのに対し、Buildah は「イメージのビルド」を担います

### 特徴

| 項目             | 説明                                                        |
| ---------------- | ----------------------------------------------------------- |
| デーモン不要     | Docker デーモンなしでイメージをビルドできる                 |
| Dockerfile 対応  | Dockerfile からのビルドをサポートする                       |
| スクリプティング | Dockerfile を使わず、シェルスクリプトでイメージを構築できる |
| OCI 準拠         | OCI Image Specification に準拠したイメージを生成する        |
| rootless         | 一般ユーザー権限でイメージをビルドできる                    |

### Dockerfile を使わないビルド

Buildah は、Dockerfile を使わずにシェルコマンドでイメージを構築できます

```
container=$(buildah from debian:bookworm-slim)
buildah run $container -- apt-get update
buildah run $container -- apt-get install -y nginx
buildah config --cmd '["nginx", "-g", "daemon off;"]' $container
buildah commit $container my-nginx
```

各コマンドがコンテナに対して直接操作を行い、最後に `commit` でイメージとして保存します

Dockerfile では表現しにくい複雑な条件分岐やループ処理を、シェルスクリプトの機能で実現できます

---

## Kaniko

<strong>Kaniko</strong> は、Google が開発したコンテナイメージのビルドツールです（※ Kaniko の Google 公式リポジトリは 2025 年 6 月にアーカイブされ、Chainguard 社がフォーク（chainguard-dev/kaniko）を維持しているが、新機能の追加は予定されておらずセキュリティパッチとバグ修正のみの提供）

Kubernetes クラスタ内でイメージをビルドするために設計されています

### なぜ Kaniko が必要か

Docker でイメージをビルドするには、Docker デーモンが必要です

Docker デーモンは root 権限で動作するため、Kubernetes のポッド内でイメージをビルドしようとすると、特権コンテナ（privileged container）が必要になります

特権コンテナはセキュリティリスクが高いため、多くの環境で使用が制限されています

Kaniko は Docker デーモンなしで Dockerfile からイメージをビルドできるため、特権コンテナが不要です

### 特徴

| 項目             | 説明                                                                                    |
| ---------------- | --------------------------------------------------------------------------------------- |
| デーモン不要     | Docker デーモンなしで Dockerfile からイメージをビルドする                               |
| 特権モード不要   | `--privileged` なしで動作する（コンテナ内では root ユーザーでの実行が必要な場合がある） |
| クラスタ内ビルド | Kubernetes のポッドとして実行し、ビルド結果をレジストリに push する                     |
| CI/CD 向き       | CI/CD パイプラインでの利用を想定した設計                                                |

### 動作の仕組み

Kaniko はコンテナとして実行され、以下の手順でイメージをビルドします

| 手順 | 動作                                                           |
| ---- | -------------------------------------------------------------- |
| 1    | Dockerfile を読み込む                                          |
| 2    | ベースイメージをレジストリから pull する                       |
| 3    | 各命令をユーザー空間で実行し、ファイルシステムの差分を記録する |
| 4    | 差分からレイヤを生成し、イメージを構築する                     |
| 5    | 完成したイメージをレジストリに push する                       |

Docker のビルドが「一時コンテナを起動して各命令を実行する」のに対し、Kaniko は「ファイルシステムの操作を自身のコンテナ内で直接行う」点が異なります

---

## ビルドツールの選択指針

| 要件                                     | 推奨                      |
| ---------------------------------------- | ------------------------- |
| 一般的な開発環境でのビルド               | Docker（BuildKit）        |
| デーモンなしでビルドしたい               | Buildah                   |
| Podman 環境でのビルド                    | Buildah                   |
| Kubernetes クラスタ内でビルドしたい      | Kaniko（※）または Buildah |
| CI/CD パイプラインでのビルド（特権なし） | Kaniko（※）または Buildah |
| Dockerfile を使わないビルド              | Buildah                   |

※ Kaniko の Google 公式リポジトリは 2025 年にアーカイブされており、新規導入時は Buildah や BuildKit（rootless モード）の検討を推奨

---

## 参考資料

このページの内容は、以下のソースに基づいています

<strong>BuildKit</strong>

- [BuildKit - GitHub](https://github.com/moby/buildkit)
  - Docker の標準ビルドエンジン
- [Docker Build architecture](https://docs.docker.com/build/architecture/)
  - BuildKit のアーキテクチャと機能

<strong>Buildah</strong>

- [Buildah - GitHub](https://github.com/containers/buildah)
  - Podman エコシステムのビルドツール
- [Buildah tutorials](https://github.com/containers/buildah/tree/main/docs/tutorials)
  - Buildah の使い方のチュートリアル

<strong>Kaniko</strong>

- [Kaniko - GitHub（Google オリジナル・アーカイブ済み）](https://github.com/GoogleContainerTools/kaniko)
  - Kubernetes 環境向けのコンテナイメージビルドツール
- [Kaniko - GitHub（Chainguard フォーク）](https://github.com/chainguard-dev/kaniko)
  - Chainguard 社が維持するフォーク

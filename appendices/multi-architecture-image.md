<div align="right">
<img src="https://img.shields.io/badge/AI-ASSISTED_STUDY-3b82f6?style=for-the-badge&labelColor=1e293b&logo=bookstack&logoColor=white" alt="AI Assisted Study" />
</div>

# appendix：マルチアーキテクチャイメージ

## はじめに

[03-image](../03-image.md) では、コンテナイメージがレイヤ構造で構成され、タグやダイジェストで識別されることを学びました

`docker pull nginx:1.27` と実行すると nginx のイメージが取得されます

しかし、同じ `nginx:1.27` というタグでも、x86_64 のマシンと ARM のマシンでは<strong>異なるバイナリ</strong>が含まれるイメージが必要です

この補足資料では、1 つのタグで複数の CPU アーキテクチャをサポートする仕組みを学びます

---

## CPU アーキテクチャとコンテナ

[01-container](../01-container.md) で学んだように、コンテナはホストのカーネルを共有する「隔離されたプロセス」です

プロセスが実行するバイナリは、特定の CPU アーキテクチャ向けにコンパイルされています

x86_64（amd64）向けにコンパイルされたバイナリは、ARM（arm64）の CPU では実行できません

| アーキテクチャ | 説明                                        | 代表的な環境                    |
| -------------- | ------------------------------------------- | ------------------------------- |
| amd64          | x86_64 とも呼ばれる 64 ビットアーキテクチャ | 多くのサーバー、デスクトップ PC |
| arm64          | AArch64 とも呼ばれる 64 ビット ARM          | Apple Silicon、AWS Graviton     |
| arm/v7         | 32 ビット ARM                               | Raspberry Pi（一部のモデル）    |

コンテナイメージにはバイナリが含まれるため、イメージも CPU アーキテクチャに依存します

---

## OCI Image Index

OCI Image Specification では、1 つのタグで複数のアーキテクチャをサポートするために<strong>Image Index</strong>（マニフェストリスト）という仕組みを定義しています

### 構造

Image Index は、複数の<strong>Manifest</strong>（マニフェスト）への参照を持ちます

各 Manifest が特定のアーキテクチャ向けのイメージを指します

```
nginx:1.27（タグ）
    │
    ↓
Image Index
    ├── Manifest（amd64 向け）
    │   ├── Config（実行コマンド、環境変数等）
    │   └── Layers（amd64 向けのレイヤ群）
    ├── Manifest（arm64 向け）
    │   ├── Config
    │   └── Layers（arm64 向けのレイヤ群）
    └── Manifest（arm/v7 向け）
        ├── Config
        └── Layers（arm/v7 向けのレイヤ群）
```

### Image Index の中身

Image Index には、各 Manifest のダイジェスト、対応するアーキテクチャ、OS の情報が含まれます

| フィールド                        | 説明                                         |
| --------------------------------- | -------------------------------------------- |
| mediaType                         | このドキュメントの種類（Image Index）        |
| manifests                         | 各アーキテクチャ向け Manifest への参照の配列 |
| manifests[].digest                | 各 Manifest のダイジェスト（SHA256）         |
| manifests[].platform.architecture | CPU アーキテクチャ（amd64、arm64 等）        |
| manifests[].platform.os           | OS（linux、windows 等）                      |

---

## pull 時のアーキテクチャ選択

`docker pull nginx:1.27` を実行すると、内部では以下の手順でイメージが取得されます

| 手順 | 動作                                                                            |
| ---- | ------------------------------------------------------------------------------- |
| 1    | レジストリに nginx:1.27 の情報を問い合わせる                                    |
| 2    | Image Index を受け取る                                                          |
| 3    | Image Index の中から、ホストの CPU アーキテクチャに一致する Manifest を選択する |
| 4    | 選択した Manifest が指す Config と Layers をダウンロードする                    |

この選択は<strong>自動的</strong>に行われます

ユーザーが CPU アーキテクチャを意識する必要はありません

amd64 のマシンで `docker pull nginx:1.27` を実行すれば amd64 向けのイメージが、arm64 のマシンで実行すれば arm64 向けのイメージが取得されます

### 特定のアーキテクチャを明示的に指定する

自動選択ではなく、特定のアーキテクチャのイメージを取得したい場合は `--platform` オプションを使います

```
docker pull --platform linux/arm64 nginx:1.27
```

この例では、ホストのアーキテクチャに関係なく、arm64 向けのイメージが取得されます

---

## マルチアーキテクチャイメージのビルド

マルチアーキテクチャイメージをビルドするには、各アーキテクチャ向けのイメージを個別にビルドし、それらを 1 つの Image Index にまとめます

### docker buildx

<strong>docker buildx</strong> は、マルチアーキテクチャイメージのビルドをサポートするビルドツールです

```
docker buildx build --platform linux/amd64,linux/arm64 -t myapp:1.0 .
```

このコマンドは、amd64 と arm64 の 2 つのアーキテクチャ向けにイメージをビルドし、1 つの Image Index にまとめます

### クロスビルドの仕組み

ホストと異なるアーキテクチャ向けのビルドには、主に 2 つの方法があります

| 方法                  | 説明                                                  | 特徴               |
| --------------------- | ----------------------------------------------------- | ------------------ |
| QEMU エミュレーション | ホスト上で対象アーキテクチャの CPU をエミュレートする | 設定が簡単だが低速 |
| ネイティブノード      | 対象アーキテクチャの実機でビルドする                  | 高速だが実機が必要 |

QEMU エミュレーションでは、amd64 のマシン上で arm64 のバイナリを生成できます

カーネルの binfmt_misc 機能により、異なるアーキテクチャのバイナリを実行する際に自動的に QEMU が呼び出されます

---

## 参考資料

このページの内容は、以下のソースに基づいています

<strong>OCI</strong>

- [OCI Image Index Specification](https://github.com/opencontainers/image-spec/blob/main/image-index.md)
  - Image Index（マニフェストリスト）の標準仕様

<strong>Docker</strong>

- [Multi-platform images](https://docs.docker.com/build/building/multi-platform/)
  - マルチアーキテクチャイメージのビルド方法
- [Docker buildx](https://docs.docker.com/build/builders/)
  - buildx の概要とビルダーの設定

<strong>QEMU</strong>

- [QEMU user-mode emulation](https://www.qemu.org/docs/master/user/main.html)
  - QEMU のユーザーモードエミュレーション

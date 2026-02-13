---
layout: default
title: イメージ
---

# [03-image：イメージ](#image) {#image}

## [はじめに](#introduction) {#introduction}

[02-oci-and-runtime](../02-oci-and-runtime/) では、コンテナランタイム（runc、containerd、Docker、Podman）がコンテナを管理する仕組みを学びました

runc は config.json と<strong>rootfs</strong>（ルートファイルシステム）を受け取って、コンテナを起動します

containerd はイメージを pull してルートファイルシステムを準備し、runc に渡します

では、この「イメージ」とは何でしょうか？

`docker run nginx` の `nginx` はどこから来るのでしょうか？

イメージの中身はどうなっているのでしょうか？

このトピックでは、コンテナイメージの仕組みを学びます

---

## [日常の例え](#everyday-analogy) {#everyday-analogy}

コンテナイメージのレイヤ構造を「透明フィルムの重ね合わせ」に例えてみましょう

<strong>レイヤ構造</strong>

透明フィルムに絵を描き、何枚も重ねて1つの完成画を作るアニメーションのセル画のようなものです

- 1枚目（ベースレイヤ）：背景を描く → OS の基本ファイル
- 2枚目：キャラクターを描く → アプリケーションをインストール
- 3枚目：小道具を描く → 設定ファイルを追加

完成画（コンテナのファイルシステム）は、すべてのフィルムを重ねた結果です

<strong>再利用</strong>

同じ背景（ベースレイヤ）を使い回せば、別のキャラクター（アプリケーション）を描く時に背景を描き直す必要がありません

同じベースイメージを使う複数のコンテナイメージは、ベースレイヤを共有できます

---

## [このページで学ぶこと](#what-you-will-learn) {#what-you-will-learn}

このページでは、以下の概念を学びます

<strong>イメージの構造</strong>

- <strong>コンテナイメージとは何か</strong>
  - ファイルシステムと設定情報のパッケージ
- <strong>レイヤ構造</strong>
  - 差分の積み重ねによる効率的な構造
- <strong>Copy-on-Write</strong>
  - レイヤの読み取り専用性とコンテナレイヤ

<strong>ファイルシステムの仕組み</strong>

- <strong>overlay filesystem</strong>
  - 複数のレイヤを1つのファイルシステムに見せる仕組み

<strong>イメージの作成と配布</strong>

- <strong>Dockerfile</strong>
  - イメージをビルドするための定義ファイル
- <strong>ビルドプロセス</strong>
  - Dockerfile の各命令がレイヤを生成する仕組み
- <strong>コンテナレジストリ</strong>
  - イメージの保存と配布の仕組み

---

## [目次](#table-of-contents) {#table-of-contents}

1. [コンテナイメージとは何か](#what-is-container-image)
2. [レイヤ構造](#layer-structure)
3. [Copy-on-Write](#copy-on-write)
4. [overlay filesystem](#overlay-filesystem)
5. [Dockerfile とイメージビルド](#dockerfile-and-image-build)
6. [イメージビルドの仕組み](#image-build-mechanism)
7. [コンテナレジストリ](#container-registry)
8. [イメージの識別](#image-identification)
9. [次のトピックへ](#next-topic)
10. [用語集](#glossary)
11. [参考資料](#references)

---

## [コンテナイメージとは何か](#what-is-container-image) {#what-is-container-image}

コンテナイメージは、コンテナを実行するために必要な<strong>すべてのファイルと設定情報</strong>をまとめたパッケージです

具体的には、以下のものが含まれます

{: .labeled}
| 含まれるもの | 例 |
| ----------------- | --------------------------------------------- |
| OS の基本ファイル | /bin、/lib、/etc などのディレクトリとファイル |
| アプリケーション | nginx、python、node などのバイナリ |
| 依存ライブラリ | アプリケーションが必要とするライブラリ |
| 設定ファイル | nginx.conf、package.json など |
| メタデータ | 実行コマンド、環境変数、ポート設定など |

イメージは<strong>読み取り専用</strong>です

イメージからコンテナを起動すると、イメージの上に書き込み可能なレイヤが追加されます（これについては後述します）

### [イメージと OS の違い](#image-vs-os) {#image-vs-os}

コンテナイメージには OS のファイル（/bin/sh、/lib/libc.so など）が含まれていますが、<strong>カーネルは含まれていません</strong>

コンテナはホストのカーネルを共有するため、イメージにはユーザー空間のファイルだけが含まれます

そのため、VM のイメージ（数 GB〜数十 GB）と比べて、コンテナイメージは小さくなります（数 MB〜数百 MB）

---

## [レイヤ構造](#layer-structure) {#layer-structure}

コンテナイメージは、1つの大きなファイルではなく、<strong>複数のレイヤ（層）が積み重なった構造</strong>をしています

### [レイヤの例](#layer-example) {#layer-example}

nginx のコンテナイメージを例に考えてみましょう

```
┌──────────────────────────────────────┐
│ レイヤ 4: nginx の設定ファイルを配置  │  ← 最上位レイヤ
├──────────────────────────────────────┤
│ レイヤ 3: nginx をインストール        │
├──────────────────────────────────────┤
│ レイヤ 2: パッケージを更新            │
├──────────────────────────────────────┤
│ レイヤ 1: Debian の基本ファイル       │  ← ベースレイヤ
└──────────────────────────────────────┘
```

各レイヤは、直前のレイヤからの<strong>差分</strong>（変更されたファイル）だけを含みます

### [レイヤ構造のメリット](#layer-structure-benefits) {#layer-structure-benefits}

<strong>1. ストレージの節約</strong>

同じベースレイヤ（例：Debian）を使うイメージが複数あっても、ベースレイヤは1回だけ保存されます

```
nginx イメージ ─── レイヤ 4（nginx 設定）
                ├── レイヤ 3（nginx インストール）
                ├── レイヤ 2（パッケージ更新）
                └── レイヤ 1（Debian）  ← 共有

python イメージ ─── レイヤ 3（Python インストール）
                ├── レイヤ 2（パッケージ更新）
                └── レイヤ 1（Debian）  ← 共有
```

<strong>2. ダウンロードの高速化</strong>

イメージを pull するとき、既にローカルに存在するレイヤはダウンロードをスキップできます

<strong>3. ビルドの高速化</strong>

イメージをビルドするとき、変更のないレイヤはキャッシュから再利用できます

---

## [Copy-on-Write](#copy-on-write) {#copy-on-write}

イメージのレイヤは<strong>読み取り専用</strong>です

コンテナを起動すると、イメージのレイヤの上に<strong>書き込み可能なコンテナレイヤ</strong>が追加されます

```
┌──────────────────────────────────────┐
│ コンテナレイヤ（書き込み可能）        │  ← コンテナ起動時に追加
├──────────────────────────────────────┤
│ レイヤ 4（読み取り専用）              │
├──────────────────────────────────────┤
│ レイヤ 3（読み取り専用）              │  イメージレイヤ
├──────────────────────────────────────┤
│ レイヤ 2（読み取り専用）              │
├──────────────────────────────────────┤
│ レイヤ 1（読み取り専用）              │
└──────────────────────────────────────┘
```

### [Copy-on-Write の動作](#copy-on-write-behavior) {#copy-on-write-behavior}

<strong>ファイルの読み取り</strong>

上のレイヤから順にファイルを検索し、最初に見つかったものを返します

イメージレイヤから直接読み取るため、コピーは発生しません

<strong>ファイルの変更</strong>

イメージレイヤのファイルを変更しようとすると、そのファイルがコンテナレイヤに<strong>コピー</strong>されます

コピーされたファイルに対して変更が適用されます

元のイメージレイヤのファイルは変更されません

<strong>ファイルの削除</strong>

イメージレイヤのファイルを削除しようとすると、コンテナレイヤに<strong>ホワイトアウトファイル</strong>（削除マーカー）が作成されます

ホワイトアウトファイルは「このファイルは存在しない」ことを示します

元のイメージレイヤのファイルは実際には削除されません

### [Copy-on-Write のメリット](#copy-on-write-benefits) {#copy-on-write-benefits}

- 同じイメージから複数のコンテナを起動しても、イメージレイヤを共有できる
- コンテナごとの変更はコンテナレイヤにだけ記録されるため、ストレージ効率が良い
- コンテナを削除すると、コンテナレイヤだけが破棄される（イメージは残る）

---

## [overlay filesystem](#overlay-filesystem) {#overlay-filesystem}

<strong>overlay filesystem</strong>（overlayfs）は、複数のディレクトリを重ね合わせて1つのファイルシステムに見せるカーネル機能です

コンテナのレイヤ構造を実現するために使われます

### [overlayfs の構成要素](#overlayfs-components) {#overlayfs-components}

overlayfs は以下の 4 つのディレクトリで構成されます

{: .labeled}
| ディレクトリ | 役割 | 対応するコンテナの概念 |
| ------------ | ---------------------------------------- | ---------------------------------- |
| lowerdir | 読み取り専用の下位レイヤ（複数指定可能） | イメージレイヤ |
| upperdir | 書き込み可能な上位レイヤ | コンテナレイヤ |
| workdir | overlayfs の内部作業用ディレクトリ | （内部使用） |
| merged | すべてのレイヤが統合された結果 | コンテナから見えるファイルシステム |

### [動作の仕組み](#operation-mechanism) {#operation-mechanism}

```
コンテナから見えるファイルシステム（merged）
              │
    ┌─────────┴─────────┐
    │                   │
upperdir              lowerdir
（コンテナレイヤ）    （イメージレイヤ）
書き込み可能           読み取り専用
```

コンテナからは merged ディレクトリが見えます

ファイルの読み取り時は、upperdir → lowerdir の順に検索します

ファイルの書き込み時は、lowerdir のファイルが upperdir にコピーされてから変更されます（Copy-on-Write）

### [overlayfs のマウント例](#overlayfs-mount-example) {#overlayfs-mount-example}

```
mount -t overlay overlay \
  -o lowerdir=/image/layer2:/image/layer1,\
     upperdir=/container/upper,\
     workdir=/container/work \
  /container/merged
```

この例では、layer1 と layer2 を重ね合わせ、その上に書き込み可能な upper レイヤを載せ、結果を merged に表示しています

---

## [Dockerfile とイメージビルド](#dockerfile-and-image-build) {#dockerfile-and-image-build}

<strong>Dockerfile</strong> は、コンテナイメージをビルド（作成）するための定義ファイルです

### [基本的な Dockerfile の例](#basic-dockerfile-example) {#basic-dockerfile-example}

```dockerfile
FROM debian:bookworm-slim

RUN apt-get update && apt-get install -y nginx

COPY nginx.conf /etc/nginx/nginx.conf

EXPOSE 80

CMD ["nginx", "-g", "daemon off;"]
```

### [主要な命令](#main-instructions) {#main-instructions}

{: .labeled}
| 命令 | 役割 | レイヤ生成 |
| ---------- | ----------------------------------------------------------------- | ------------------------------------ |
| FROM | ベースイメージを指定する | ベースレイヤを取得 |
| RUN | コマンドを実行する（パッケージのインストール等） | 新しいレイヤを生成 |
| COPY | ホストのファイルをイメージにコピーする | 新しいレイヤを生成 |
| ADD | COPY と同様だが、URL からのダウンロードやアーカイブの展開もできる | 新しいレイヤを生成 |
| CMD | コンテナ起動時に実行するデフォルトコマンドを指定する | レイヤを生成しない（メタデータのみ） |
| ENTRYPOINT | コンテナ起動時に必ず実行するコマンドを指定する | レイヤを生成しない（メタデータのみ） |
| ENV | 環境変数を設定する | レイヤを生成しない（メタデータのみ） |
| EXPOSE | コンテナがリッスンするポートを宣言する | レイヤを生成しない（メタデータのみ） |
| WORKDIR | 以降の命令の作業ディレクトリを設定する | レイヤを生成しない（メタデータのみ） |

### [FROM：ベースイメージ](#from-base-image) {#from-base-image}

すべての Dockerfile は FROM 命令から始まります

FROM はベースイメージ（土台となるイメージ）を指定します

多くの場合、Debian、Ubuntu、Alpine などの Linux ディストリビューションのイメージをベースにします

### [RUN：コマンドの実行](#run-command) {#run-command}

RUN 命令は、イメージのビルド時にコマンドを実行します

パッケージのインストール、ファイルの作成、設定の変更などに使います

各 RUN 命令は新しいレイヤを生成するため、関連するコマンドは `&&` でつないで1つの RUN にまとめるのが一般的です

### [CMD と ENTRYPOINT](#cmd-and-entrypoint) {#cmd-and-entrypoint}

CMD はコンテナ起動時のデフォルトコマンドを指定します

`docker run` でコマンドを指定すると、CMD は上書きされます

ENTRYPOINT はコンテナ起動時に必ず実行するコマンドを指定します

`docker run` でコマンドを指定すると、ENTRYPOINT の引数として渡されます

---

## [イメージビルドの仕組み](#image-build-mechanism) {#image-build-mechanism}

`docker build` を実行すると、Dockerfile の各命令が順番に処理されます

### [ビルドの流れ](#build-flow) {#build-flow}

```
Dockerfile
    │
    │  FROM debian:bookworm-slim
    ├──→ ベースイメージのレイヤを取得
    │
    │  RUN apt-get update && apt-get install -y nginx
    ├──→ 一時コンテナでコマンドを実行 → 差分をレイヤとして保存
    │
    │  COPY nginx.conf /etc/nginx/nginx.conf
    ├──→ ファイルをコピー → 差分をレイヤとして保存
    │
    │  CMD ["nginx", "-g", "daemon off;"]
    └──→ メタデータとして記録（レイヤは生成しない）
    │
    ↓
  完成したイメージ（3つのレイヤ + メタデータ）
```

### [ビルドキャッシュ](#build-cache) {#build-cache}

Docker は各レイヤの結果をキャッシュします

Dockerfile を変更してリビルドする際、変更がないレイヤはキャッシュから再利用されます

```
FROM debian:bookworm-slim          ← キャッシュ使用（変更なし）
RUN apt-get update && ...          ← キャッシュ使用（変更なし）
COPY nginx.conf /etc/nginx/...     ← 再実行（nginx.conf が変更された）
CMD ["nginx", ...]                 ← 再実行（上位レイヤが変更された）
```

キャッシュは「変更されたレイヤ以降」がすべて無効になります

そのため、変更頻度の低い命令を先に書き、変更頻度の高い命令を後に書くとビルドが高速になります

---

## [コンテナレジストリ](#container-registry) {#container-registry}

<strong>コンテナレジストリ</strong>は、コンテナイメージを保存・配布するためのサービスです

### [レジストリの役割](#registry-role) {#registry-role}

{: .labeled}
| 機能 | 説明 |
| -------------- | ------------------------------------------------ |
| 保存 | イメージをサーバーに保存する |
| 配布 | イメージを pull（ダウンロード）できるようにする |
| 公開 / 非公開 | パブリックイメージとプライベートイメージの管理 |
| バージョン管理 | タグやダイジェストによるイメージのバージョン管理 |

### [主なレジストリ](#main-registries) {#main-registries}

{: .labeled}
| レジストリ | 特徴 |
| ------------------------- | ------------------------------------------------------ |
| Docker Hub | Docker 公式のパブリックレジストリ<br>最大のイメージ数 |
| GitHub Container Registry | GitHub が提供するレジストリ<br>GitHub リポジトリとの統合 |
| Amazon ECR | AWS が提供するプライベートレジストリ |
| Google Artifact Registry | Google Cloud が提供するレジストリ |

### [pull と push](#pull-and-push) {#pull-and-push}

<strong>pull（イメージの取得）</strong>

```
docker pull nginx:1.27
```

レジストリからイメージのレイヤをダウンロードします

既にローカルに存在するレイヤはダウンロードをスキップします

<strong>push（イメージの公開）</strong>

```
docker push myregistry/myapp:1.0
```

ローカルのイメージをレジストリにアップロードします

レジストリに既に存在するレイヤはアップロードをスキップします

### [OCI Distribution Specification](#oci-distribution-spec) {#oci-distribution-spec}

レジストリとの通信プロトコルは、OCI Distribution Specification で標準化されています

この仕様により、Docker で push したイメージを Podman で pull する、といった相互運用が可能になっています

---

## [イメージの識別](#image-identification) {#image-identification}

コンテナイメージは<strong>タグ</strong>と<strong>ダイジェスト</strong>の2つの方法で識別できます

### [タグ](#tag) {#tag}

タグは人間が読みやすい名前です

```
nginx:1.27        ← イメージ名:タグ
nginx:latest      ← latest は最新版を示す慣例的なタグ
nginx             ← タグを省略すると latest と見なされる
```

タグは<strong>書き換え可能</strong>です

同じタグ（例：latest）が異なるイメージを指すことがあるため、本番環境では注意が必要です

### [ダイジェスト](#digest) {#digest}

ダイジェストはイメージの内容から計算されるハッシュ値です

```
nginx@sha256:abc123...    ← イメージ名@ダイジェスト
```

ダイジェストはイメージの内容が変わらない限り同じ値になります

タグと違って書き換えることができないため、<strong>常に同じイメージ</strong>を指すことが保証されます

本番環境で確実に同じイメージを使いたい場合は、ダイジェストで指定します

---

## [次のトピックへ](#next-topic) {#next-topic}

このトピックでは、以下のことを学びました

- コンテナイメージはファイルシステムと設定のパッケージである
- イメージはレイヤ構造で、差分を積み重ねることで効率的にストレージを使う
- Copy-on-Write により、イメージレイヤは読み取り専用のまま、コンテナレイヤで変更を管理する
- overlay filesystem が複数のレイヤを1つのファイルシステムに統合する
- Dockerfile で定義し、ビルドで作成し、レジストリで配布する

これで、コンテナの「中身」がどう作られるかが分かりました

しかし、コンテナは独立した環境で動作するため、そのままでは外部と通信できません

次のトピック [04-network](../04-network/) では、<strong>コンテナのネットワーク</strong>を学びます

コンテナがどうやってホストや他のコンテナと通信するのか、bridge、veth ペア、ポートマッピングの仕組みを理解します

---

## [用語集](#glossary) {#glossary}

{: .labeled}
| 用語 | 説明 |
| ------------------------------- | ------------------------------------------------------------------------------------ |
| コンテナイメージ | コンテナの実行に必要なファイルシステムと設定情報をまとめた、読み取り専用のパッケージ |
| レイヤ | イメージを構成する個々の差分<br>各レイヤは直前の状態からの変更点を含む |
| ベースレイヤ | イメージの最下層のレイヤ<br>通常は OS のファイルシステム |
| ベースイメージ | Dockerfile の FROM で指定する、土台となるイメージ |
| コンテナレイヤ | コンテナ起動時にイメージの上に追加される書き込み可能なレイヤ |
| Copy-on-Write（CoW） | 読み取り専用のデータを変更する際に、コピーを作成してから変更する方式 |
| ホワイトアウトファイル | overlayfs でファイルの削除を表現するための特殊なマーカーファイル |
| overlay filesystem（overlayfs） | 複数のディレクトリを重ね合わせて1つのファイルシステムに見せるカーネル機能 |
| lowerdir | overlayfs の読み取り専用の下位レイヤ |
| upperdir | overlayfs の書き込み可能な上位レイヤ |
| merged | overlayfs ですべてのレイヤが統合された結果のディレクトリ |
| Dockerfile | コンテナイメージをビルドするための定義ファイル |
| FROM | Dockerfile でベースイメージを指定する命令 |
| RUN | Dockerfile でビルド時にコマンドを実行する命令<br>新しいレイヤを生成する |
| COPY | Dockerfile でホストのファイルをイメージにコピーする命令 |
| CMD | Dockerfile でコンテナ起動時のデフォルトコマンドを指定する命令 |
| ENTRYPOINT | Dockerfile でコンテナ起動時に必ず実行するコマンドを指定する命令 |
| ビルドキャッシュ | 変更のないレイヤを再利用してビルドを高速化する仕組み |
| コンテナレジストリ | コンテナイメージを保存・配布するためのサービス |
| Docker Hub | Docker 公式のパブリックコンテナレジストリ |
| タグ | イメージに付ける人間が読みやすい名前（例: nginx:1.27） |
| ダイジェスト | イメージの内容から計算されるハッシュ値<br>不変で一意にイメージを識別する |
| pull | レジストリからイメージをダウンロードする操作 |
| push | ローカルのイメージをレジストリにアップロードする操作 |

---

## [参考資料](#references) {#references}

このページの内容は、以下のソースに基づいています

<strong>OCI 仕様</strong>

- [OCI Image Specification](https://github.com/opencontainers/image-spec/blob/main/spec.md){:target="\_blank"}
  - コンテナイメージの標準仕様（レイヤ構造、マニフェスト、設定）
- [OCI Distribution Specification](https://github.com/opencontainers/distribution-spec/blob/main/spec.md){:target="\_blank"}
  - イメージ配布の標準仕様（レジストリとの通信プロトコル）

<strong>overlay filesystem</strong>

- [overlayfs - Linux kernel documentation](https://docs.kernel.org/filesystems/overlayfs.html){:target="\_blank"}
  - overlay filesystem のカーネルドキュメント（lowerdir / upperdir / merged の仕組み）

<strong>Dockerfile</strong>

- [Dockerfile reference](https://docs.docker.com/reference/dockerfile/){:target="\_blank"}
  - Dockerfile の命令リファレンス（FROM、RUN、COPY、CMD 等）

<strong>Docker ストレージ</strong>

- [Docker storage drivers](https://docs.docker.com/engine/storage/drivers/){:target="\_blank"}
  - Docker のストレージドライバ（overlay2）の仕組み

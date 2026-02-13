---
layout: default
title: read.container
---

# [read.container](#read-container) {#read-container}

<strong>「コンテナはどう動き、どう使うか」</strong>を学びます

---

## [このリポジトリは何のためにあるのか](#purpose) {#purpose}

前のシリーズでは、namespace によるプロセスの隔離と、cgroup によるリソースの制限を学びました

これらはカーネルが提供する「仕組み」でしたが、ではその仕組みを使って<strong>コンテナ</strong>はどう作られるのでしょうか？

Docker や Podman は何をしているのでしょうか？

「コンテナはプロセスである」とよく言われますが、普通のプロセスと何が違うのでしょうか？

このリポジトリでは、その「コンテナ技術の仕組み」を学びます

---

## [このリポジトリの特徴](#repository-characteristics) {#repository-characteristics}

### [<strong>namespace + cgroup からコンテナを理解する</strong>](#understand-from-namespace-cgroup) {#understand-from-namespace-cgroup}

コンテナの正体は、namespace で隔離され、cgroup で制限されたプロセスです

カーネルの仕組みから出発することで、コンテナが「魔法」ではなく「Linux の機能の組み合わせ」であることを理解します

### [<strong>仕組みを理解してからツールを学ぶ</strong>](#understand-before-tools) {#understand-before-tools}

Docker や Podman の使い方（コマンドの打ち方）を覚えるのではなく、その裏側で何が起きているかを理解します

仕組みを知った上でツールを使えば、トラブル時にも自分で原因を考えられるようになります

### [<strong>各トピック独立で読める構成</strong>](#independent-topics) {#independent-topics}

7つのトピックはそれぞれ独立して読めるように書かれています

興味のあるトピックから読み始めても、必要な知識はそのトピック内で説明されています

---

## [なぜこれを学ぶのか](#why-learn-this) {#why-learn-this}

### [<strong>1. コンテナの仕組みを理解する</strong>](#understand-container-mechanism) {#understand-container-mechanism}

「docker run したら何が起きるのか」を、namespace の作成から cgroup の設定、ファイルシステムのマウントまで、仕組みから説明できるようになります

### [<strong>2. Docker と Podman の違いを理解する</strong>](#understand-docker-vs-podman) {#understand-docker-vs-podman}

Docker と Podman はどちらもコンテナを動かすツールですが、アーキテクチャが異なります

デーモンの有無、rootless 対応、セキュリティモデルの違いを仕組みから理解します

### [<strong>3. オーケストレーションへの橋渡し</strong>](#bridge-to-orchestration) {#bridge-to-orchestration}

Kubernetes などのコンテナオーケストレーションは、コンテナの知識を前提としています

コンテナの仕組みを理解することで、オーケストレーションの学習で「複数コンテナの管理」に進む準備ができます

---

## [このリポジトリで学ぶこと](#repository-contents) {#repository-contents}

{: .labeled}
| 順番 | トピック | 学ぶこと |
| ---- | ------------------------------------------ | ------------------------------------------------------- |
| 01 | [container](./01-container/) | コンテナの正体、namespace と cgroup の役割、VM との違い |
| 02 | [oci-and-runtime](./02-oci-and-runtime/) | OCI 仕様、runc / containerd / Docker / Podman の関係 |
| 03 | [image](./03-image/) | レイヤ構造、overlay filesystem、レジストリ、ビルド |
| 04 | [network](./04-network/) | bridge、veth ペア、ポートマッピング、コンテナ間通信 |
| 05 | [storage](./05-storage/) | ボリューム、バインドマウント、ストレージドライバ |
| 06 | [security](./06-security/) | capability、seccomp、rootless コンテナ、多層防御 |
| 07 | [compose](./07-compose/) | 複数コンテナの定義と管理、compose.yaml の構造 |

---

## [前提知識](#prerequisites) {#prerequisites}

このリポジトリを始める前に、以下の学習を推奨します（必須ではありません）

- <strong>カーネル空間の学習</strong>
  - namespace（06-namespace）と cgroup（07-cgroup）の知識がコンテナの理解に直接つながります
  - ただし、このリポジトリでもコンテナの文脈で改めて説明するため、未読でも学習できます
- <strong>ネットワークの学習</strong>
  - TCP/IP の基礎知識がコンテナネットワーク（04-network）の理解に役立ちます
  - ただし、ネットワークの必要な知識はそのトピック内で説明します

---

## [既存リポジトリとの関係](#relationship-with-existing-repos) {#relationship-with-existing-repos}

このリポジトリは、カーネル空間の学習の知識を「コンテナ技術」という新しい方向に拡張します

{: .labeled}
| 概念 | カーネル空間の学習での扱い | このリポジトリでの扱い |
| -------------- | -------------------------- | -------------------------------------- |
| namespace | プロセス隔離のカーネル機能 | コンテナの隔離を実現する仕組み |
| cgroup | リソース制限のカーネル機能 | コンテナのリソース制限を実現する仕組み |
| User namespace | UID/GID マッピング | rootless コンテナの基盤 |

また、このリポジトリで学ぶ知識は、後続の学習につながります

{: .labeled}
| 概念 | このリポジトリでの扱い | 後続の学習での扱い |
| -------------------- | ------------------------------- | ---------------------------------------------- |
| コンテナの仕組み | 単一コンテナの動作原理 | オーケストレーションの学習で複数コンテナの管理 |
| コンテナセキュリティ | capability / seccomp / rootless | セキュリティの学習でセキュリティの原理を深掘り |

---

## [参考資料](#references) {#references}

このリポジトリの内容は、以下のソースに基づいています

各トピックで使用する個別の仕様や man ページは、各トピックのドキュメントに記載しています

- [OCI Specifications](https://opencontainers.org/){:target="\_blank"}
  - コンテナランタイム、イメージ、配布の標準仕様
- [Linux man-pages](https://man7.org/linux/man-pages/){:target="\_blank"}
  - namespace、cgroup、capabilities 等の公式マニュアル
- [Docker Documentation](https://docs.docker.com/){:target="\_blank"}
  - Docker の公式ドキュメント
- [Podman Documentation](https://docs.podman.io/){:target="\_blank"}
  - Podman の公式ドキュメント

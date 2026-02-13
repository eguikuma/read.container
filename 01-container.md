---
layout: default
title: コンテナとは何か
---

# [01-container：コンテナとは何か](#container) {#container}

## [はじめに](#introduction) {#introduction}

前のシリーズでは、Linux カーネルが提供する2つの重要な機能を学びました

- プロセスが見える範囲を制限する<strong>namespace</strong>（名前空間）

namespace を使うと、プロセスは自分専用の PID 空間、ネットワーク空間、ファイルシステム空間を持つことができます

- プロセスが使えるリソースを制限する<strong>cgroup</strong>（コントロールグループ）

cgroup を使うと、CPU 時間やメモリ使用量に上限を設けることができます

では、この2つの仕組みを<strong>組み合わせる</strong>と何が起きるでしょうか？

「自分専用の空間を持ち、使えるリソースも制限されたプロセス」── それが<strong>コンテナ</strong>です

このトピックでは、コンテナとは何か、そしてなぜ「軽量な仮想マシン」と呼ばれるのかを学びます

---

## [日常の例え](#everyday-analogy) {#everyday-analogy}

コンテナと仮想マシン（VM）の違いを「住居」に例えてみましょう

<strong>仮想マシン（VM）</strong>

<strong>一戸建ての家</strong>のようなものです

それぞれの家に、独自の基礎（OS カーネル）、壁、屋根、水道、電気の配線があります

完全に独立していますが、建てるのに時間がかかり、土地（リソース）もたくさん必要です

<strong>コンテナ</strong>

<strong>マンションの部屋</strong>のようなものです

建物の基盤（カーネル）は全部屋で共有していますが、部屋の中は完全にプライベートです

各部屋には独自の鍵（namespace）があり、電気や水道の使用量に上限（cgroup）が設定されています

建物を新しく建てる必要がないので、部屋の準備はすぐに終わります

<strong>共通点</strong>

どちらも「独立した環境でアプリケーションを動かす」という目的は同じです

違いは、OS カーネルを<strong>専有するか共有するか</strong>です

---

## [このページで学ぶこと](#what-you-will-learn) {#what-you-will-learn}

このページでは、以下の概念を学びます

<strong>コンテナの基礎</strong>

- <strong>コンテナとは何か</strong>
  - プロセスの隔離環境としてのコンテナの本質
- <strong>VM との違い</strong>
  - ハイパーバイザ型 vs カーネル共有型の比較

<strong>コンテナを構成するカーネル機能</strong>

- <strong>namespace</strong>
  - 8種類の namespace がコンテナの隔離をどう実現するか
- <strong>cgroup</strong>
  - CPU、メモリ、I/O の制限がコンテナの資源管理をどう実現するか

<strong>コンテナの全体像</strong>

- <strong>コンテナに必要なもの</strong>
  - namespace + cgroup だけでは足りない要素
- <strong>コンテナのライフサイクル</strong>
  - 作成から削除までの状態遷移

---

## [目次](#table-of-contents) {#table-of-contents}

1. [コンテナとは何か](#what-is-container)
2. [仮想マシンとの違い](#vs-virtual-machine)
3. [namespace がコンテナの隔離を作る](#namespace-for-isolation)
4. [cgroup がコンテナのリソースを制限する](#cgroup-for-resource-limits)
5. [namespace + cgroup だけではコンテナにならない](#beyond-namespace-and-cgroup)
6. [コンテナのライフサイクル](#container-lifecycle)
7. [次のトピックへ](#next-topic)
8. [用語集](#glossary)
9. [参考資料](#references)

---

## [コンテナとは何か](#what-is-container) {#what-is-container}

コンテナとは、<strong>隔離されたプロセス</strong>です

普通のプロセスと同じように Linux カーネル上で動作しますが、以下の2つの点で異なります

- <strong>namespace</strong> によって、他のプロセスから隔離されている
- <strong>cgroup</strong> によって、使えるリソースが制限されている

つまり、コンテナは「特別な仮想マシン」ではなく、「カーネルの機能で隔離・制限された普通のプロセス」です

この理解が非常に重要です

コンテナの中で `ps` コマンドを実行すると、コンテナ内のプロセスしか見えません

しかし、ホスト側から見ると、コンテナのプロセスは他のプロセスと同じようにプロセス一覧に表示されます

```
ホスト側の視点：

  PID 1     ─── systemd（ホストの init プロセス）
  PID 1234  ─── nginx（コンテナ内のプロセス）
  PID 5678  ─── postgres（別のコンテナ内のプロセス）
  PID 9012  ─── bash（ホストのシェル）

コンテナ内の視点（nginx コンテナ）：

  PID 1     ─── nginx（自分だけが見える）
```

ホストから見ると PID 1234 の nginx は、コンテナの中から見ると PID 1 です

これは <strong>PID namespace</strong> が、コンテナ内のプロセスに独自の PID 空間を提供しているためです

---

## [仮想マシンとの違い](#vs-virtual-machine) {#vs-virtual-machine}

コンテナと仮想マシン（VM）は、どちらも「アプリケーションを独立した環境で動かす」という目的を持っていますが、実現方法が大きく異なります

### [アーキテクチャの比較](#architecture-comparison) {#architecture-comparison}

<strong>仮想マシン（VM）</strong>

```
┌────────────────────────────────────────────┐
│              アプリケーション              │
├────────────────────────────────────────────┤
│              ゲスト OS カーネル             │
├────────────────────────────────────────────┤
│              ハイパーバイザ                 │
├────────────────────────────────────────────┤
│              ホスト OS カーネル             │
├────────────────────────────────────────────┤
│              ハードウェア                   │
└────────────────────────────────────────────┘
```

VM は<strong>ハイパーバイザ</strong>がハードウェアを仮想化し、その上で<strong>ゲスト OS</strong>（カーネルを含む完全な OS）を動かします

各 VM は独自のカーネルを持つため、完全に独立していますが、その分リソースを多く消費します

<strong>コンテナ</strong>

```
┌────────────────────────────────────────────┐
│              アプリケーション              │
├────────────────────────────────────────────┤
│              コンテナランタイム             │
├────────────────────────────────────────────┤
│              ホスト OS カーネル             │
├────────────────────────────────────────────┤
│              ハードウェア                   │
└────────────────────────────────────────────┘
```

コンテナは<strong>ホストのカーネルを共有</strong>し、namespace と cgroup でプロセスを隔離・制限します

ゲスト OS が不要なため、起動が速く、リソース消費も少なくなります

### [比較表](#comparison-table) {#comparison-table}

{: .labeled}
| 項目 | 仮想マシン（VM） | コンテナ |
| -------------- | -------------------------------------- | -------------------------------------- |
| 隔離の仕組み | ハイパーバイザによるハードウェア仮想化 | namespace と cgroup によるプロセス隔離 |
| カーネル | ゲスト OS 専用のカーネル | ホストのカーネルを共有 |
| 起動時間 | 数十秒〜数分（OS の起動が必要） | 数百ミリ秒〜数秒（プロセスの起動のみ） |
| リソース消費 | 大きい（OS 全体が必要） | 小さい（アプリケーションのみ） |
| イメージサイズ | 数 GB〜数十 GB | 数 MB〜数百 MB |
| 隔離レベル | 高い（カーネルが独立） | 中程度（カーネルを共有） |
| 用途 | 異なる OS の実行、強い隔離が必要な場合 | マイクロサービス、CI/CD、開発環境 |

### [どちらを選ぶか](#which-to-choose) {#which-to-choose}

コンテナは軽量で起動が速いため、多くの場面で VM の代わりに使われるようになりました

しかし、カーネルを共有するため、隔離レベルは VM より低くなります

「異なる OS カーネルが必要」「より強い隔離が必要」といった要件がある場合は、VM が適しています

コンテナと VM は対立するものではなく、多くの場合は併用されています

たとえば、VM の中でコンテナを動かすことで、VM の強い隔離とコンテナの軽量さの両方を得ることができます

---

## [namespace がコンテナの隔離を作る](#namespace-for-isolation) {#namespace-for-isolation}

namespace は、プロセスが見える<strong>システムリソースの範囲を制限</strong>するカーネル機能です

コンテナは namespace を使って、自分専用の環境（PID 空間、ネットワーク、ファイルシステムなど）を作ります

Linux には以下の 8 種類の namespace があります

{: .labeled}
| namespace | 隔離するもの | コンテナでの役割 |
| ----------------- | -------------------- | --------------------------------------------------------------------------- |
| PID namespace | プロセス ID | コンテナ内のプロセスだけが見える<br>コンテナ内の最初のプロセスが PID 1 になる |
| Network namespace | ネットワークスタック | コンテナ専用のネットワークインターフェース、IP アドレス、ポートを持つ |
| Mount namespace | マウントポイント | コンテナ専用のファイルシステムを持つ |
| UTS namespace | ホスト名 | コンテナ専用のホスト名を持つ |
| IPC namespace | プロセス間通信 | コンテナ内のプロセス同士だけが IPC できる |
| User namespace | UID / GID | コンテナ内の root をホストの一般ユーザーにマッピングできる |
| Cgroup namespace | cgroup の階層ビュー | コンテナから見える cgroup の階層を制限する |
| Time namespace | 時刻 | コンテナ専用の時刻オフセットを持つ（Linux 5.6 以降） |

### [PID namespace](#pid-namespace) {#pid-namespace}

コンテナの中で動くプロセスには、ホストとは別の PID が割り当てられます

コンテナ内の最初のプロセスは PID 1 になります

PID 1 は通常 OS の init プロセス（systemd 等）ですが、コンテナでは起動したアプリケーション自体が PID 1 になります

PID 1 にはシグナルハンドリングに関する特別な扱いがあるため、コンテナのアプリケーション設計に影響します

### [Network namespace](#network-namespace) {#network-namespace}

各コンテナは独自のネットワークスタックを持ちます

これにより、コンテナごとに独立した IP アドレスとポートを使えます

たとえば、2つのコンテナがどちらもポート 80 でリッスンしても、Network namespace が異なるため競合しません

コンテナのネットワークについては [04-network](../04-network/) で詳しく学びます

### [Mount namespace](#mount-namespace) {#mount-namespace}

各コンテナは独自のファイルシステムビューを持ちます

コンテナのルートファイルシステム（/）は、コンテナイメージから作られた専用のファイルシステムです

ホストのファイルシステムとは独立しているため、コンテナ内でファイルを変更してもホストには影響しません

### [User namespace](#user-namespace) {#user-namespace}

User namespace を使うと、コンテナ内の root（UID 0）をホストの一般ユーザー（例：UID 100000）にマッピングできます

これにより、コンテナ内では root 権限で動作しているように見えても、ホストから見ると一般ユーザーとして動作します

この仕組みは<strong>rootless コンテナ</strong>の基盤です（[06-security](../06-security/) で詳しく学びます）

---

## [cgroup がコンテナのリソースを制限する](#cgroup-for-resource-limits) {#cgroup-for-resource-limits}

cgroup は、プロセスグループが使用できるリソースに<strong>上限を設定</strong>するカーネル機能です

namespace が「何が見えるか」を制御するのに対し、cgroup は「どれだけ使えるか」を制御します

### [なぜリソース制限が必要なのか](#why-resource-limits) {#why-resource-limits}

namespace だけでは、あるコンテナが CPU やメモリを大量に消費して、他のコンテナやホストに影響を与えることを防げません

cgroup を使えば、各コンテナが使えるリソースを制限し、1つのコンテナが暴走しても他に影響しないようにできます

### [主要なコントローラ](#main-controllers) {#main-controllers}

cgroup には複数の<strong>コントローラ</strong>があり、それぞれ異なるリソースを制限します

{: .labeled}
| コントローラ | 制限するリソース | コンテナでの用途 |
| ------------ | ---------------- | -------------------------------------------------------- |
| CPU | CPU 使用時間 | コンテナが使える CPU 時間の上限を設定 |
| Memory | メモリ使用量 | コンテナが使えるメモリの上限を設定 |
| I/O | ディスク I/O | コンテナのディスク読み書き速度を制限 |
| PID | プロセス数 | コンテナ内で作成できるプロセス数を制限（fork bomb 対策） |

### [Docker での設定例](#docker-configuration-example) {#docker-configuration-example}

Docker では、コンテナ実行時のオプションが cgroup の設定に変換されます

{: .labeled}
| Docker オプション | cgroup の設定 | 意味 |
| ------------------ | ------------- | --------------------------------- |
| `--cpus=2` | cpu.max | CPU を最大2コア分使用可能 |
| `--memory=512m` | memory.max | メモリを最大 512 MB 使用可能 |
| `--pids-limit=100` | pids.max | プロセスを最大 100 個まで作成可能 |

`docker run --cpus=2 --memory=512m nginx` のように指定すると、Docker はカーネルの cgroup を設定して、そのコンテナのリソースを制限します

---

## [namespace + cgroup だけではコンテナにならない](#beyond-namespace-and-cgroup) {#beyond-namespace-and-cgroup}

ここまで、namespace がプロセスを隔離し、cgroup がリソースを制限することを学びました

しかし、実用的なコンテナを作るには、さらにいくつかの要素が必要です

{: .labeled}
| 要素 | 役割 | 詳細 |
| ------------------ | -------------------- | ------------------------------------------------------------- |
| namespace | プロセスの隔離 | 見える範囲を制限する |
| cgroup | リソースの制限 | 使える量を制限する |
| ファイルシステム | コンテナの中身 | アプリケーションとその依存関係を含むファイルシステム |
| コンテナイメージ | ファイルシステムの元 | レイヤ構造で効率的に管理されたファイルシステムのテンプレート |
| コンテナランタイム | コンテナの管理 | namespace / cgroup の設定、イメージの展開、ライフサイクル管理 |
| ネットワーク | コンテナの通信 | コンテナとホスト、コンテナ同士の通信を実現する |

これらの要素は、このリポジトリの後続トピックで1つずつ学びます

- コンテナランタイム → [02-oci-and-runtime](../02-oci-and-runtime/)
- コンテナイメージとファイルシステム → [03-image](../03-image/)
- ネットワーク → [04-network](../04-network/)

---

## [コンテナのライフサイクル](#container-lifecycle) {#container-lifecycle}

コンテナは以下の状態を遷移します

{: .labeled}
| 状態 | 説明 |
| ------------------- | ---------------------------------------------------------------------------------------------- |
| Creating（作成中） | コンテナの設定（namespace、cgroup、ファイルシステム）を準備している状態 |
| Created（作成済み） | コンテナの設定が準備された状態<br>プロセスはまだ開始していない |
| Running（実行中） | コンテナ内のプロセスが実行されている状態 |
| Stopped（停止済み） | コンテナ内のプロセスが終了した状態<br>コンテナの設定やファイルシステムの変更はまだ保持されている |

これらは OCI Runtime Specification で定義される主要な 4 つの状態です（仕様にはこのほか Pausing / Paused 状態も存在しますが、基本的な流れはこの 4 状態で理解できます）

詳しくは [02-oci-and-runtime](../02-oci-and-runtime/) で学びます

### [状態遷移の流れ](#state-transition-flow) {#state-transition-flow}

```
Creating ──→ Created ──→ Running ──→ Stopped ──→（削除）
```

<strong>Creating → Created</strong>

コンテナランタイムが namespace を作成し、cgroup を設定し、ファイルシステムをマウントします

この時点でコンテナの環境は準備済みですが、プロセスはまだ開始していません

<strong>Created → Running</strong>

コンテナ内のプロセスを開始します

<strong>Running → Stopped</strong>

コンテナ内のプロセスが終了する（正常終了またはシグナルによる終了）と、コンテナは停止状態になります

namespace と cgroup はまだ存在し、ファイルシステムの変更も残っています

<strong>Stopped →（再起動）</strong>

`docker restart` 等のコマンドでコンテナを再起動できます

ただし、OCI Runtime Specification には stopped → running の直接遷移は定義されていません

再起動は Docker 等の高レベルツールが提供する操作であり、内部的には停止→削除→再作成→起動に相当します

Docker の場合、ファイルシステムの変更は保持されたまま、プロセスが再開されます

<strong>Stopped →（削除）</strong>

コンテナを削除すると、namespace、cgroup、コンテナレイヤのファイルシステムの変更がすべて破棄されます

削除はコンテナを消去する<strong>操作</strong>であり、削除されたコンテナは存在しなくなります

---

## [次のトピックへ](#next-topic) {#next-topic}

このトピックでは、以下のことを学びました

- コンテナは「隔離されたプロセス」であり、仮想マシンではない
- namespace がプロセスの見える範囲を制限し、cgroup が使えるリソースを制限する
- namespace + cgroup だけではコンテナにならない（ファイルシステム、ランタイム、ネットワークも必要）
- コンテナには Creating → Created → Running → Stopped のライフサイクルがある

しかし、ここで大きな疑問が残っています

- namespace の作成
- cgroup の設定
- ファイルシステムのマウント

これらを<strong>誰がまとめて管理する</strong>のでしょうか？

`docker run` と打つと、裏側で何が動いているのでしょうか？

次のトピック [02-oci-and-runtime](../02-oci-and-runtime/) では、コンテナを管理する<strong>ランタイム</strong>と、その標準仕様である <strong>OCI</strong> を学びます

---

## [用語集](#glossary) {#glossary}

{: .labeled}
| 用語 | 説明 |
| ------------------ | --------------------------------------------------------------------------------------------------------------- |
| コンテナ | namespace で隔離され、cgroup でリソースが制限されたプロセス実行環境 |
| 仮想マシン（VM） | ハイパーバイザがハードウェアを仮想化し、ゲスト OS を含む完全に独立した環境を提供する仕組み |
| ハイパーバイザ | ハードウェアを仮想化して複数の VM を動作させるソフトウェア |
| ゲスト OS | VM の中で動作する OS（独自のカーネルを含む） |
| ホスト OS | VM やコンテナが動作する物理マシンの OS |
| namespace | プロセスが見えるシステムリソースの範囲を制限するカーネル機能 |
| PID namespace | プロセス ID の空間を隔離する namespace |
| Network namespace | ネットワークスタック（インターフェース、IP アドレス、ポート、ルーティングテーブル）を隔離する namespace |
| Mount namespace | マウントポイントを隔離する namespace |
| UTS namespace | ホスト名とドメイン名を隔離する namespace |
| IPC namespace | System V IPC と POSIX メッセージキューを隔離する namespace |
| User namespace | UID と GID のマッピングを提供する namespace |
| Cgroup namespace | cgroup の階層ビューを隔離する namespace |
| Time namespace | 時刻のオフセットを隔離する namespace |
| cgroup | プロセスグループのリソース使用量を制限・管理するカーネル機能 |
| コントローラ | cgroup でリソースを制限するためのモジュール（CPU、Memory、I/O、PID 等） |
| コンテナランタイム | namespace の作成、cgroup の設定、ファイルシステムのマウントなど、コンテナのライフサイクルを管理するソフトウェア |
| コンテナイメージ | コンテナのファイルシステムのテンプレート<br>アプリケーションとその依存関係を含む |
| rootless コンテナ | User namespace を使い、一般ユーザー権限でコンテナを実行する仕組み |
| fork bomb | プロセスが無限に自分自身を複製し、システムリソースを枯渇させる攻撃 |
| ライフサイクル | コンテナの状態遷移（Creating → Created → Running → Stopped） |

---

## [参考資料](#references) {#references}

このページの内容は、以下のソースに基づいています

<strong>namespace</strong>

- [namespaces(7) - Linux manual page](https://man7.org/linux/man-pages/man7/namespaces.7.html){:target="\_blank"}
  - Linux namespace の概要と 8 種類の namespace の説明

<strong>cgroup</strong>

- [cgroups(7) - Linux manual page](https://man7.org/linux/man-pages/man7/cgroups.7.html){:target="\_blank"}
  - cgroup v1 / v2 の概要、コントローラの説明

<strong>コンテナの標準仕様</strong>

- [OCI Runtime Specification](https://github.com/opencontainers/runtime-spec/blob/main/spec.md){:target="\_blank"}
  - コンテナのライフサイクル（状態遷移）の定義

<strong>Docker ドキュメント</strong>

- [Docker overview](https://docs.docker.com/get-started/overview/){:target="\_blank"}
  - Docker のアーキテクチャとコンテナの概要

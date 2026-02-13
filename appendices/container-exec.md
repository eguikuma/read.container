---
layout: default
title: コンテナへの接続の仕組み
---

# [appendix：コンテナへの接続の仕組み](#container-exec) {#container-exec}

## [はじめに](#introduction) {#introduction}

[01-container](../../01-container/) では、namespace がプロセスの見える範囲を制限することを学びました

コンテナは独自の PID namespace、Network namespace、Mount namespace などを持ち、ホストから隔離されています

しかし、開発やデバッグの場面では、実行中のコンテナの中でコマンドを実行したい場合があります

`docker exec` はまさにそのためのコマンドですが、隔離された namespace の中にどうやって「入る」のでしょうか

この補足資料では、実行中のコンテナの namespace に接続する仕組みを学びます

---

## [setns：既存の namespace に参加する](#setns) {#setns}

Linux カーネルは<strong>setns()</strong> というシステムコールを提供しています

setns() は、呼び出したプロセスを<strong>既存の namespace に参加</strong>させます

### [/proc/\<pid\>/ns/ ディレクトリ](#proc-pid-ns-directory) {#proc-pid-ns-directory}

Linux カーネルは、各プロセスの namespace 情報を `/proc/<pid>/ns/` ディレクトリに公開しています

{: .labeled}
| ファイル | 対応する namespace |
| ----------------------- | ------------------ |
| /proc/\<pid\>/ns/pid | PID namespace |
| /proc/\<pid\>/ns/net | Network namespace |
| /proc/\<pid\>/ns/mnt | Mount namespace |
| /proc/\<pid\>/ns/uts | UTS namespace |
| /proc/\<pid\>/ns/ipc | IPC namespace |
| /proc/\<pid\>/ns/user | User namespace |
| /proc/\<pid\>/ns/cgroup | Cgroup namespace |
| /proc/\<pid\>/ns/time | Time namespace |

これらのファイルは namespace への<strong>参照</strong>です

setns() にこのファイルのファイルディスクリプタを渡すことで、対応する namespace に参加できます

### [setns() の動作](#setns-operation) {#setns-operation}

```
プロセス A（コンテナ内、PID namespace X に所属）

プロセス B（ホスト側）
    │
    │ setns(/proc/A/ns/pid)  ← A の PID namespace に参加
    │ setns(/proc/A/ns/net)  ← A の Network namespace に参加
    │ setns(/proc/A/ns/mnt)  ← A の Mount namespace に参加
    │
    ↓
プロセス B はコンテナと同じ namespace に所属
（コンテナ内のプロセスが見え、コンテナのネットワークを使い、コンテナのファイルシステムが見える）
```

setns() を複数回呼び出すことで、複数の namespace に順番に参加できます

---

## [docker exec の内部動作](#docker-exec-internals) {#docker-exec-internals}

`docker exec -it <container> /bin/sh` を実行すると、内部では以下の処理が行われます

{: .labeled}
| 手順 | コンポーネント | 動作 |
| ---- | -------------- | ---------------------------------------------------------------------------------- |
| 1 | Docker CLI | docker exec コマンドを受け付け、Docker デーモンに API リクエストを送信する |
| 2 | dockerd | containerd に exec リクエストを転送する |
| 3 | containerd | コンテナの namespace 情報を取得し、runc exec を呼び出す |
| 4 | runc | setns() でコンテナの各 namespace に参加し、指定されたコマンド（/bin/sh）を実行する |

[02-oci-and-runtime](../../02-oci-and-runtime/) で学んだように、Docker CLI → dockerd → containerd → runc というランタイムの階層構造がここでも使われています

重要なのは、docker exec は<strong>新しいコンテナを作るのではなく</strong>、既存のコンテナの namespace に新しいプロセスを追加するという点です

---

## [nsenter コマンド](#nsenter-command) {#nsenter-command}

<strong>nsenter</strong> は、setns() を直接使って既存の namespace に入るコマンドラインツールです

Docker や Podman を使わずに、プロセスの PID を指定して namespace に接続できます

```
nsenter --target <pid> --mount --uts --ipc --net --pid -- /bin/sh
```

{: .labeled}
| オプション | 説明 |
| ---------------- | --------------------------------------- |
| --target \<pid\> | 参加先の namespace を持つプロセスの PID |
| --mount | Mount namespace に参加する |
| --uts | UTS namespace に参加する |
| --ipc | IPC namespace に参加する |
| --net | Network namespace に参加する |
| --pid | PID namespace に参加する |

### [docker exec との違い](#difference-from-docker-exec) {#difference-from-docker-exec}

{: .labeled}
| 項目 | docker exec | nsenter |
| ---------- | ---------------------------------------- | ------------------------------------- |
| 必要なもの | Docker デーモンが動作していること | コンテナ内プロセスの PID が分かること |
| 依存 | Docker CLI → dockerd → containerd → runc | nsenter コマンドのみ |
| 操作対象 | Docker が管理するコンテナ | 任意のプロセスの namespace |
| cgroup | コンテナの cgroup に参加する | デフォルトでは cgroup に参加しない |

nsenter は Docker に依存しないため、Docker デーモンが停止している場合や、Docker 以外のランタイムで作られたコンテナにも接続できます

---

## [デバッグでの活用](#debug-usage) {#debug-usage}

### [コンテナ内にデバッグツールがない場合](#when-no-debug-tools) {#when-no-debug-tools}

本番環境のコンテナイメージは、セキュリティとサイズの観点から最小限のツールしか含まないことが多いです

[multi-stage-build](../multi-stage-build/) で学んだ distroless イメージにはシェルすら含まれていません

このような場合、nsenter を使えばホスト側のツールをコンテナの namespace 内で実行できます

```
nsenter --target <pid> --net -- ss -tlnp
```

この例では、ホスト側の `ss`（ソケット統計）コマンドを、コンテナの Network namespace 内で実行しています

コンテナ内にツールをインストールすることなく、コンテナのネットワーク状態を確認できます

### [特定の namespace だけに参加する](#joining-specific-namespace) {#joining-specific-namespace}

nsenter では、参加する namespace を個別に選択できます

たとえば、ネットワークの問題を調べたい場合は Network namespace だけに参加します

```
nsenter --target <pid> --net -- ip addr
```

ファイルシステムの問題を調べたい場合は Mount namespace に参加します

```
nsenter --target <pid> --mount -- ls /etc/
```

必要な namespace だけに参加することで、調査の範囲を限定できます

---

## [参考資料](#references) {#references}

このページの内容は、以下のソースに基づいています

<strong>setns</strong>

- [setns(2) - Linux manual page](https://man7.org/linux/man-pages/man2/setns.2.html){:target="\_blank"}
  - 既存の namespace に参加するシステムコール

<strong>nsenter</strong>

- [nsenter(1) - Linux manual page](https://man7.org/linux/man-pages/man1/nsenter.1.html){:target="\_blank"}
  - namespace に入るコマンドラインツール

<strong>/proc</strong>

- [proc(5) - Linux manual page](https://man7.org/linux/man-pages/man5/proc.5.html){:target="\_blank"}
  - /proc ファイルシステムの説明（/proc/\<pid\>/ns/ を含む）

<strong>Docker</strong>

- [Docker exec](https://docs.docker.com/reference/cli/docker/container/exec/){:target="\_blank"}
  - docker exec コマンドのリファレンス

<div align="right">
<img src="https://img.shields.io/badge/AI-ASSISTED_STUDY-3b82f6?style=for-the-badge&labelColor=1e293b&logo=bookstack&logoColor=white" alt="AI Assisted Study" />
</div>

# appendix：PID 1 問題

## はじめに

[01-container](../01-container.md) では、PID namespace によってコンテナ内の最初のプロセスが PID 1 になることを学びました

そして「PID 1 にはシグナルハンドリングに関する特別な扱いがあるため、コンテナのアプリケーション設計に影響します」と述べました

この補足資料では、PID 1 の特殊性がコンテナでどのような問題を引き起こすのかを詳しく学びます

---

## PID 1 の特殊性

Linux カーネルは PID 1 を<strong>特別なプロセス</strong>として扱います

通常の Linux システムでは、PID 1 は systemd などの<strong>init プロセス</strong>です

init プロセスはシステム全体のプロセス管理を担う重要な役割を持つため、カーネルは PID 1 に以下の特別な扱いを適用します

### シグナルのデフォルト動作が無効化される

通常のプロセスでは、シグナルハンドラを登録していないシグナルに対して<strong>デフォルト動作</strong>が実行されます

たとえば、SIGTERM のデフォルト動作はプロセスの終了です

しかし、PID 1 では<strong>明示的にシグナルハンドラを登録していないシグナルはすべて無視</strong>されます

| プロセス   | SIGTERM を受信（ハンドラ未登録） | 結果         |
| ---------- | -------------------------------- | ------------ |
| 通常の PID | デフォルト動作（終了）           | 終了する     |
| PID 1      | デフォルト動作が無効化           | 何も起きない |

これは、init プロセスが誤って終了するとシステム全体が停止してしまうため、カーネルが init を保護する仕組みです

### PID 1 はゾンビプロセスを回収する責務がある

プロセスが終了すると、カーネルはそのプロセスの終了情報を親プロセスが回収するまで保持します

この状態のプロセスを<strong>ゾンビプロセス</strong>と呼びます

親プロセスが wait() システムコールを呼び出すことで、ゾンビプロセスの終了情報が回収され、プロセスが完全に消滅します

親プロセスが先に終了した場合、その子プロセスは<strong>孤児プロセス</strong>になり、PID 1 に引き取られます

PID 1 は引き取った孤児プロセスの終了を wait() で回収する責務を持ちます

通常のシステムの init プロセス（systemd 等）はこの責務を果たすように設計されています

---

## コンテナでの問題

コンテナでは PID namespace により、コンテナ内で最初に起動するプロセスが PID 1 になります

通常、これはアプリケーション自体（nginx、node、python 等）です

しかし、一般的なアプリケーションは<strong>init プロセスとして動作することを想定していません</strong>

これにより、2 つの問題が発生します

### 問題 1：graceful shutdown ができない

`docker stop` コマンドは、以下の手順でコンテナを停止します

| 手順 | 動作                                                      |
| ---- | --------------------------------------------------------- |
| 1    | コンテナの PID 1 に SIGTERM を送信する                    |
| 2    | 猶予期間（デフォルト 10 秒）待機する                      |
| 3    | 猶予期間内に終了しなければ SIGKILL を送信して強制終了する |

アプリケーションが SIGTERM のハンドラを登録していない場合、PID 1 の特殊性により SIGTERM は無視されます

結果として、猶予期間が過ぎた後に SIGKILL で強制終了されます

SIGKILL による強制終了では、接続中のリクエストの処理完了や、データの書き込み完了を待つことができません

```
SIGTERM ハンドラがある場合：
  docker stop → SIGTERM → アプリが処理を完了 → 正常終了

SIGTERM ハンドラがない場合（PID 1）：
  docker stop → SIGTERM → 無視 → 10秒待機 → SIGKILL → 強制終了
```

### 問題 2：ゾンビプロセスが蓄積する

コンテナ内でアプリケーションが子プロセスを生成し、その子プロセスがさらに子プロセスを生成するケースがあります

子プロセスの親が終了すると、孤児プロセスは PID 1 に引き取られます

一般的なアプリケーションは wait() による孤児プロセスの回収を行わないため、ゾンビプロセスが蓄積していきます

ゾンビプロセス自体は CPU やメモリをほとんど消費しませんが、カーネルのプロセステーブルのエントリを占有します

cgroup の pids.max による制限に達すると、新しいプロセスを作成できなくなります

---

## 解決策：コンテナ用 init プロセス

これらの問題を解決するために、コンテナ用の軽量な init プロセスが開発されています

### tini

<strong>tini</strong> は、コンテナ専用に設計された軽量 init プロセスです

tini は以下の 2 つの責務だけを果たします

| 責務                 | 動作                                                                |
| -------------------- | ------------------------------------------------------------------- |
| シグナルの転送       | tini が受け取ったシグナルを子プロセス（アプリケーション）に転送する |
| ゾンビプロセスの回収 | 孤児プロセスを wait() で回収する                                    |

tini を使うと、PID 1 が tini になり、アプリケーションは PID 2 以降で動作します

```
tini を使わない場合：
  PID 1: nginx（SIGTERM が無視される）

tini を使う場合：
  PID 1: tini（SIGTERM を受け取り nginx に転送する）
  PID 2: nginx（通常のプロセスとして SIGTERM を受け取る）
```

### dumb-init

<strong>dumb-init</strong> は、Yelp が開発した軽量 init プロセスです

tini と同様にシグナルの転送とゾンビプロセスの回収を行います

### Docker の --init オプション

Docker は `--init` オプションでコンテナに tini を自動的に挿入する機能を提供しています

```
docker run --init nginx
```

このオプションを指定すると、Docker が tini をコンテナの PID 1 として起動し、指定したコマンド（nginx）をその子プロセスとして実行します

---

## Dockerfile の書き方との関係

Dockerfile の CMD / ENTRYPOINT の書き方によって、コンテナ内のプロセス構成が変わります

### exec form と shell form

CMD / ENTRYPOINT には 2 つの記法があります

| 記法       | 例                                 | PID 1 になるプロセス |
| ---------- | ---------------------------------- | -------------------- |
| exec form  | CMD ["nginx", "-g", "daemon off;"] | nginx                |
| shell form | CMD nginx -g "daemon off;"         | /bin/sh -c           |

<strong>exec form</strong>

```dockerfile
CMD ["nginx", "-g", "daemon off;"]
```

指定したコマンド（nginx）が直接 PID 1 として起動されます

nginx が SIGTERM のハンドラを登録していれば、`docker stop` で正常終了できます

<strong>shell form</strong>

```dockerfile
CMD nginx -g "daemon off;"
```

この場合、Docker は内部的に `/bin/sh -c "nginx -g 'daemon off;'"` を実行します

PID 1 は `/bin/sh` になり、nginx は `/bin/sh` の子プロセスとして起動されます

`/bin/sh` はシグナルを子プロセスに転送しないため、`docker stop` で SIGTERM を送っても nginx には届きません

### 推奨される書き方

| 方針                                  | 説明                                                      |
| ------------------------------------- | --------------------------------------------------------- |
| exec form を使う                      | アプリケーションが直接 PID 1 になり、シグナルを受け取れる |
| アプリケーションで SIGTERM を処理する | graceful shutdown を実装する                              |
| または --init（tini）を使う           | シグナル転送とゾンビ回収を tini に任せる                  |

---

## 参考資料

このページの内容は、以下のソースに基づいています

<strong>シグナル</strong>

- [signal(7) - Linux manual page](https://man7.org/linux/man-pages/man7/signal.7.html)
  - シグナルのデフォルト動作と PID 1 の特殊性

<strong>Docker</strong>

- [Docker stop](https://docs.docker.com/reference/cli/docker/container/stop/)
  - docker stop のシグナル送信手順
- [Dockerfile reference - CMD](https://docs.docker.com/reference/dockerfile/#cmd)
  - exec form と shell form の違い

<strong>tini</strong>

- [tini - GitHub](https://github.com/krallin/tini)
  - コンテナ専用の軽量 init プロセス

<strong>dumb-init</strong>

- [dumb-init - GitHub](https://github.com/Yelp/dumb-init)
  - Yelp が開発した軽量 init プロセス

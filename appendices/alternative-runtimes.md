<div align="right">
<img src="https://img.shields.io/badge/AI-ASSISTED_STUDY-3b82f6?style=for-the-badge&labelColor=1e293b&logo=bookstack&logoColor=white" alt="AI Assisted Study" />
</div>

# appendix：代替コンテナランタイムと関連技術

## はじめに

[02-oci-and-runtime](../02-oci-and-runtime.md) では、runc、containerd、Docker、Podman を学びました

しかし、コンテナエコシステムにはこれら以外にも多くのランタイムと関連技術が存在します

この補足資料では、主要な代替ランタイムと関連技術の概要を紹介します

---

## コンテナランタイムの分類

```
ユーザー向けツール
├── Docker
├── Podman
└── nerdctl

高レベルランタイム
├── containerd
└── CRI-O

低レベルランタイム
├── runc（標準）
├── crun
├── Kata Containers（VM ベース）
└── gVisor（ユーザー空間カーネル）

OS コンテナ
├── LXC / LXD（Incus）
└── systemd-nspawn
```

---

## CRI-O

<strong>CRI-O</strong> は、Kubernetes 専用に設計された高レベルコンテナランタイムです

| 項目               | 説明                                                       |
| ------------------ | ---------------------------------------------------------- |
| 目的               | Kubernetes の CRI（Container Runtime Interface）を実装する |
| 特徴               | Kubernetes に不要な機能を持たない軽量な設計                |
| 低レベルランタイム | runc（デフォルト）、crun、Kata Containers 等に対応         |

containerd が Docker から独立した汎用ランタイムであるのに対し、CRI-O は Kubernetes での利用に特化しています

---

## crun

<strong>crun</strong> は、C 言語で実装された OCI 準拠の低レベルコンテナランタイムです

| 項目     | 説明                                                          |
| -------- | ------------------------------------------------------------- |
| 実装言語 | C（runc は Go）                                               |
| 特徴     | runc より起動が速く、メモリ使用量が少ない                     |
| 互換性   | OCI Runtime Specification に準拠。runc の代替として使用可能   |
| 採用     | Podman のデフォルトランタイム（一部のディストリビューション） |

crun は runc と同じ OCI 仕様に準拠しているため、runc の代わりに使用できます

---

## Kata Containers

<strong>Kata Containers</strong> は、コンテナを<strong>軽量な VM</strong> の中で実行するランタイムです

| 項目     | 説明                                                     |
| -------- | -------------------------------------------------------- |
| 目的     | VM レベルの強い隔離とコンテナの使いやすさを両立する      |
| 仕組み   | 各コンテナ（またはポッド）ごとに専用の軽量 VM を起動する |
| カーネル | コンテナごとに専用のゲストカーネルを持つ                 |
| OCI 準拠 | はい。runc の代わりとして使用可能                        |

通常のコンテナはホストのカーネルを共有するため、カーネルの脆弱性がリスクになります

Kata Containers は各コンテナに専用のカーネルを提供することで、VM に匹敵する隔離レベルを実現します

ただし、VM を起動するため、通常のコンテナより起動時間が長く、リソース消費も大きくなります

---

## gVisor

<strong>gVisor</strong> は、Google が開発した<strong>ユーザー空間カーネル</strong>です

| 項目           | 説明                                             |
| -------------- | ------------------------------------------------ |
| 目的           | カーネルへの攻撃面を縮小する                     |
| 仕組み         | コンテナのシステムコールをユーザー空間で処理する |
| コンポーネント | runsc（OCI 準拠のランタイム）を提供              |
| OCI 準拠       | はい。runc の代わりとして使用可能                |

通常のコンテナでは、プロセスのシステムコールは直接ホストのカーネルに到達します

gVisor は、コンテナのシステムコールを<strong>ユーザー空間で実装されたカーネル</strong>（Sentry）がインターセプトして処理します

ホストカーネルへのシステムコールは、gVisor が最小限のものだけを代行して発行します

これにより、コンテナからホストカーネルへの攻撃面が大幅に縮小されます

ただし、システムコールのオーバーヘッドがあるため、I/O 性能が低下する場合があります

---

## LXC / LXD（Incus）

<strong>LXC</strong>（Linux Containers）は、Linux の namespace と cgroup を直接使用する<strong>OS コンテナ</strong>技術です

<strong>LXD</strong>（現在は <strong>Incus</strong> としてフォーク）は、LXC を管理するためのツールです

| 項目 | 説明                                                    |
| ---- | ------------------------------------------------------- |
| 目的 | コンテナ内で完全な Linux ディストリビューションを動かす |
| 特徴 | アプリケーションコンテナ（Docker）ではなく、OS コンテナ |
| init | コンテナ内で systemd などの init システムが動作する     |
| 用途 | 軽量 VM の代替、開発環境                                |

Docker や Podman は「アプリケーションコンテナ」であり、1つのコンテナで1つのアプリケーションを動かします

LXC は「OS コンテナ」であり、1つのコンテナで完全な OS 環境を動かします

### アプリケーションコンテナ vs OS コンテナ

|              | アプリケーションコンテナ | OS コンテナ                  |
| ------------ | ------------------------ | ---------------------------- |
| 例           | Docker、Podman           | LXC、LXD                     |
| PID 1        | アプリケーション自体     | init（systemd 等）           |
| サービス管理 | 1コンテナ1プロセスが基本 | systemd で複数サービスを管理 |
| イメージ     | レイヤ構造、Dockerfile   | OS のルートファイルシステム  |
| 用途         | マイクロサービス、CI/CD  | 軽量 VM の代替、開発環境     |

---

## systemd-nspawn

<strong>systemd-nspawn</strong> は、systemd に組み込まれた軽量コンテナツールです

| 項目 | 説明                                                     |
| ---- | -------------------------------------------------------- |
| 目的 | OS コンテナの簡易実行                                    |
| 特徴 | systemd の一部として標準で利用可能。追加インストール不要 |
| 用途 | OS のテスト、ビルド環境、デバッグ                        |

systemd-nspawn は「強力な chroot」として位置づけられています

chroot に namespace と cgroup による隔離とリソース制限を加えたものです

OCI 仕様には準拠しておらず、Docker や Podman のエコシステム（イメージ、レジストリ等）とは互換性がありません

---

## nerdctl

<strong>nerdctl</strong> は、containerd 用の Docker 互換コマンドラインツールです

| 項目 | 説明                                                             |
| ---- | ---------------------------------------------------------------- |
| 目的 | containerd を Docker と同じ感覚で操作する                        |
| 特徴 | Docker CLI と互換性のあるコマンド体系                            |
| 関係 | Docker CLI → dockerd の代わりに、nerdctl → containerd を直接操作 |

Docker Engine（dockerd）を介さずに、containerd を直接操作できます

Docker Compose 互換の `nerdctl compose` もサポートしています

---

## ランタイムの選択指針

| 要件                               | 推奨                    |
| ---------------------------------- | ----------------------- |
| 一般的な開発・本番環境             | Docker または Podman    |
| rootless をデフォルトで使いたい    | Podman                  |
| Kubernetes 環境                    | containerd または CRI-O |
| VM レベルの強い隔離が必要          | Kata Containers         |
| カーネルへの攻撃面を最小化したい   | gVisor                  |
| OS コンテナ（完全な Linux 環境）   | LXC / Incus             |
| 追加インストールなしで簡易コンテナ | systemd-nspawn          |

---

## 参考資料

このページの内容は、以下のソースに基づいています

<strong>CRI-O</strong>

- [CRI-O](https://cri-o.io/)
  - Kubernetes 専用コンテナランタイムの公式サイト

<strong>crun</strong>

- [crun - GitHub](https://github.com/containers/crun)
  - C 言語実装の OCI ランタイム

<strong>Kata Containers</strong>

- [Kata Containers](https://katacontainers.io/)
  - VM ベースのコンテナランタイムの公式サイト

<strong>gVisor</strong>

- [gVisor](https://gvisor.dev/)
  - ユーザー空間カーネルの公式サイト

<strong>LXC / Incus</strong>

- [LXC](https://linuxcontainers.org/lxc/)
  - Linux Containers の公式サイト
- [Incus](https://linuxcontainers.org/incus/)
  - LXD のフォークプロジェクト

<strong>systemd-nspawn</strong>

- [systemd-nspawn(1)](https://www.freedesktop.org/software/systemd/man/latest/systemd-nspawn.html)
  - systemd-nspawn のマニュアル

<strong>nerdctl</strong>

- [nerdctl - GitHub](https://github.com/containerd/nerdctl)
  - containerd 用の Docker 互換 CLI

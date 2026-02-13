---
layout: default
title: コンテナ技術の歴史
---

# [appendix：コンテナ技術の歴史](#container-history) {#container-history}

## [はじめに](#introduction) {#introduction}

[01-container](../../01-container/) では、コンテナが namespace と cgroup を使った「隔離されたプロセス」であることを学びました

しかし、コンテナ技術は突然登場したものではありません

「プロセスを隔離する」というアイデアは、数十年にわたって進化してきました

この補足資料では、コンテナ技術の歴史を辿ります

---

## [年表](#timeline) {#timeline}

{: .labeled}
| 年 | 技術 | OS | 概要 |
| ---- | ------------- | ------- | -------------------------------------------------------------------------- |
| 1979 | chroot | UNIX V7 | ファイルシステムのルートディレクトリを変更する |
| 2000 | FreeBSD jail | FreeBSD | chroot を拡張し、ネットワークやプロセスも隔離する |
| 2004 | Solaris Zones | Solaris | OS レベルの仮想化<br>ゾーンごとに独立した環境を提供 |
| 2008 | cgroup | Linux | プロセスグループのリソース制限（Google が 2006 年に開発、2.6.24 でマージ） |
| 2008 | LXC | Linux | namespace と cgroup を組み合わせた Linux コンテナ |
| 2013 | Docker | Linux | コンテナの作成・配布・実行を統合したプラットフォーム |
| 2015 | OCI 設立 | ─ | コンテナの標準仕様を策定する組織の設立 |
| 2015 | runc | Linux | OCI 準拠の低レベルコンテナランタイム |
| 2017 | containerd | Linux | Docker から分離された高レベルコンテナランタイム |
| 2019 | Podman 1.0 | Linux | デーモンレスのコンテナランタイム |

---

## [chroot（1979）](#chroot) {#chroot}

<strong>chroot</strong>（change root）は、プロセスのルートディレクトリ（/）を別のディレクトリに変更するシステムコールです

```
通常のプロセス：
  / ─── bin/
       etc/
       home/

chroot されたプロセス：
  /new-root/ ─── bin/    ← このプロセスにとっての /
                 etc/
                 lib/
```

chroot されたプロセスは、新しいルートディレクトリの外にあるファイルにアクセスできなくなります

### [chroot の限界](#chroot-limitations) {#chroot-limitations}

chroot はファイルシステムの隔離だけを提供します

{: .labeled}
| 隔離されるもの | 隔離されないもの |
| ------------------------ | ------------------------------------- |
| ファイルシステムのビュー | プロセス（他のプロセスが見える） |
| | ネットワーク（ホストと共有） |
| | リソース制限（CPU、メモリの制限なし） |

また、root 権限があれば chroot 環境から脱出できるため、セキュリティ機構としては不十分です

chroot は「隔離の出発点」として重要ですが、今日のコンテナとは大きく異なります

---

## [FreeBSD jail（2000）](#freebsd-jail) {#freebsd-jail}

<strong>FreeBSD jail</strong> は、chroot を大幅に拡張した FreeBSD の隔離機構です

jail では、ファイルシステムに加えて以下も隔離されます

{: .labeled}
| 隔離されるもの | 説明 |
| ------------------ | ------------------------------------------------- |
| ファイルシステム | chroot と同様 |
| プロセス空間 | jail 内のプロセスは他の jail のプロセスを見えない |
| ネットワーク | jail ごとに独立した IP アドレスを持つ |
| ユーザーとグループ | jail 内の root は jail の外に影響を与えられない |

jail は「OS レベルの仮想化」の先駆けであり、今日のコンテナ技術に大きな影響を与えました

---

## [Solaris Zones（2004）](#solaris-zones) {#solaris-zones}

<strong>Solaris Zones</strong> は、Sun Microsystems（後に Oracle が買収）の Solaris OS で提供された OS レベルの仮想化技術です

ゾーンごとに独立したファイルシステム、プロセス空間、ネットワーク、ユーザー管理を提供します

リソースの制限も組み込まれており、ゾーン間でのリソース分配が可能でした

---

## [Linux の namespace と cgroup（2002〜2013）](#linux-namespace-and-cgroup) {#linux-namespace-and-cgroup}

Linux カーネルに段階的に追加された namespace と cgroup が、現在のコンテナ技術の基盤です

<strong>namespace の追加時期</strong>

{: .labeled}
| 年 | バージョン | namespace |
| ---- | ------------ | ---------------------------- |
| 2002 | Linux 2.4.19 | Mount namespace |
| 2006 | Linux 2.6.19 | UTS namespace、IPC namespace |
| 2008 | Linux 2.6.24 | PID namespace |
| 2009 | Linux 2.6.29 | Network namespace |
| 2013 | Linux 3.8 | User namespace |
| 2016 | Linux 4.6 | Cgroup namespace |
| 2020 | Linux 5.6 | Time namespace |

<strong>cgroup</strong>

2006 年に Google のエンジニアが開発し、2008 年に Linux 2.6.24 でカーネルに取り込まれました

2016 年に cgroup v2 が Linux 4.5 でリリースされ、統一された階層構造が提供されました

---

## [LXC（2008）](#lxc) {#lxc}

<strong>LXC</strong>（Linux Containers）は、namespace と cgroup を組み合わせてコンテナを実現する最初の本格的な Linux コンテナ技術です

LXC は「OS コンテナ」として、コンテナ内で完全な Linux ディストリビューションを動かすことを目的としていました

Docker の初期バージョン（0.x）は、内部で LXC を使用していました

---

## [Docker（2013）](#docker) {#docker}

<strong>Docker</strong> は、2013 年に dotCloud 社（後の Docker 社）が公開したコンテナプラットフォームです

Docker が革新的だったのは、コンテナの「実行」だけでなく、以下を統合したことです

{: .labeled}
| 機能 | 説明 |
| ------------ | --------------------------------------------------------- |
| イメージ | レイヤ構造のイメージ形式<br>Dockerfile による宣言的なビルド |
| レジストリ | Docker Hub によるイメージの共有・配布 |
| CLI | 直感的なコマンドラインインターフェース |
| エコシステム | Docker Compose、Docker Swarm など関連ツールの提供 |

Docker の登場により、コンテナ技術は開発者に広く普及しました

「Docker = コンテナ」と認識されるほど、Docker の影響は大きくなりました

---

## [OCI と標準化（2015〜）](#oci-and-standardization) {#oci-and-standardization}

Docker の普及に伴い、コンテナの標準化が必要になりました

2015 年に <strong>OCI</strong>（Open Container Initiative）が設立され、以下の標準仕様が策定されました

- Runtime Specification（コンテナの実行方法）
- Image Specification（コンテナイメージの構造）
- Distribution Specification（イメージの配布方法）

Docker の内部コンポーネントだった runc と containerd が独立したプロジェクトとなり、OCI 準拠のランタイムとして公開されました

この標準化により、Docker 以外のランタイム（Podman、CRI-O 等）でも互換性のあるコンテナを実行できるようになりました

---

## [コンテナ技術の進化の流れ](#container-technology-evolution) {#container-technology-evolution}

```
chroot（1979）
  │  ファイルシステムの隔離
  ↓
FreeBSD jail（2000）
  │  プロセス・ネットワークの隔離を追加
  ↓
Solaris Zones（2004）
  │  リソース管理を追加
  ↓
Linux namespace + cgroup（2006〜）
  │  カーネルレベルの隔離とリソース制限
  ↓
LXC（2008）
  │  namespace と cgroup の統合利用
  ↓
Docker（2013）
  │  イメージ、レジストリ、CLI の統合
  ↓
OCI 標準化（2015〜）
  │  ランタイム、イメージ、配布の標準仕様
  ↓
Podman、containerd、CRI-O（2017〜）
     OCI 準拠の多様なランタイム
```

---

## [参考資料](#references) {#references}

このページの内容は、以下のソースに基づいています

<strong>chroot</strong>

- [chroot(2) - Linux manual page](https://man7.org/linux/man-pages/man2/chroot.2.html){:target="\_blank"}
  - chroot システムコールの仕様

<strong>OCI</strong>

- [Open Container Initiative](https://opencontainers.org/){:target="\_blank"}
  - OCI の公式サイト

<strong>Docker</strong>

- [Docker overview](https://docs.docker.com/get-started/overview/){:target="\_blank"}
  - Docker のアーキテクチャと歴史

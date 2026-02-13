<div align="right">
<img src="https://img.shields.io/badge/AI-ASSISTED_STUDY-3b82f6?style=for-the-badge&labelColor=1e293b&logo=bookstack&logoColor=white" alt="AI Assisted Study" />
</div>

# appendix：cgroup v1 と v2

## はじめに

[01-container](../01-container.md) では、cgroup がコンテナのリソースを制限する仕組みを学びました

Docker の設定例では `cpu.max` や `memory.max` といったインターフェースファイルが登場しました

実は、cgroup には<strong>v1</strong> と<strong>v2</strong> の 2 つのバージョンがあり、これらのファイル名は v2 のものです

[container-history](./container-history.md) では、2008 年に cgroup が Linux カーネルにマージされ、2016 年に cgroup v2 がリリースされたことを紹介しました

この補足資料では、v1 と v2 の設計の違いを学びます

---

## cgroup v1 の構造

cgroup v1 では、各コントローラ（CPU、メモリ、PID 等）が<strong>独立した階層（ディレクトリツリー）</strong>を持ちます

```
/sys/fs/cgroup/
├── cpu/
│   ├── docker/
│   │   ├── コンテナA/
│   │   │   ├── cpu.cfs_quota_us
│   │   │   └── cpu.cfs_period_us
│   │   └── コンテナB/
│   └── tasks
├── memory/
│   ├── docker/
│   │   ├── コンテナA/
│   │   │   ├── memory.limit_in_bytes
│   │   │   └── memory.usage_in_bytes
│   │   └── コンテナB/
│   └── tasks
└── pids/
    ├── docker/
    │   ├── コンテナA/
    │   │   └── pids.max
    │   └── コンテナB/
    └── tasks
```

CPU の制限は `/sys/fs/cgroup/cpu/` 以下で管理し、メモリの制限は `/sys/fs/cgroup/memory/` 以下で管理します

各コントローラのディレクトリツリーは<strong>完全に独立</strong>しています

プロセスは、コントローラごとに<strong>異なるグループ</strong>に所属できます

たとえば、あるプロセスが CPU コントローラではグループ A に、メモリコントローラではグループ B に所属するという構成が可能です

---

## cgroup v1 の問題点

v1 の設計にはいくつかの問題がありました

| 問題             | 説明                                                                       |
| ---------------- | -------------------------------------------------------------------------- |
| 複雑な階層管理   | コントローラごとに独立した階層を持つため、管理が複雑になる                 |
| 一貫性の欠如     | コントローラ間でインターフェースの命名規則や動作が統一されていない         |
| 委任の難しさ     | 非 root ユーザーにグループの管理を委任するルールが複雑で安全性に懸念がある |
| リソース間の連携 | CPU とメモリなど、コントローラ間でリソースを連携させることが難しい         |

特に「委任の難しさ」は、rootless コンテナ（[06-security](../06-security.md)）の実現を妨げる要因の1つでした

---

## cgroup v2 の設計

cgroup v2 は v1 の問題を解決するために設計されました

最も大きな変更は<strong>統一された単一階層</strong>（unified hierarchy）の採用です

```
/sys/fs/cgroup/
├── cgroup.controllers        （利用可能なコントローラの一覧）
├── cgroup.subtree_control    （子グループで有効にするコントローラ）
├── docker/
│   ├── コンテナA/
│   │   ├── cpu.max
│   │   ├── memory.max
│   │   ├── pids.max
│   │   └── cgroup.procs
│   └── コンテナB/
│       ├── cpu.max
│       ├── memory.max
│       ├── pids.max
│       └── cgroup.procs
└── cgroup.procs
```

v2 では、すべてのコントローラが<strong>同じ階層ツリー</strong>を共有します

1 つのグループ（ディレクトリ）の中に、CPU、メモリ、PID すべてのコントローラのインターフェースファイルが存在します

プロセスは<strong>1 つのグループにのみ</strong>所属します

---

## v1 と v2 の比較

### 階層構造

| 項目                 | v1                                           | v2                                  |
| -------------------- | -------------------------------------------- | ----------------------------------- |
| 階層の数             | コントローラごとに独立した階層               | 単一の統一された階層                |
| プロセスの所属       | コントローラごとに異なるグループに所属できる | 1 つのグループにのみ所属する        |
| コントローラの有効化 | 階層ごとにコントローラがマウントされる       | `cgroup.subtree_control` で制御する |

### インターフェースファイルの違い

| リソース     | v1                                       | v2                             |
| ------------ | ---------------------------------------- | ------------------------------ |
| CPU 上限     | `cpu.cfs_quota_us` + `cpu.cfs_period_us` | `cpu.max`                      |
| メモリ上限   | `memory.limit_in_bytes`                  | `memory.max`                   |
| メモリ使用量 | `memory.usage_in_bytes`                  | `memory.current`               |
| PID 上限     | `pids.max`                               | `pids.max`                     |
| 所属プロセス | `tasks`（スレッド単位）                  | `cgroup.procs`（プロセス単位） |

v2 ではインターフェースファイルの命名規則が統一されています

v1 の `cpu.cfs_quota_us` と `cpu.cfs_period_us` の 2 つのファイルで設定していた CPU 上限が、v2 では `cpu.max` の 1 つのファイルにまとめられています

### 委任モデル

| 項目         | v1                                     | v2                                 |
| ------------ | -------------------------------------- | ---------------------------------- |
| 委任の仕組み | 複雑で安全な委任が難しい               | 明確な委任ルールが定義されている   |
| rootless     | 非 root ユーザーからの操作に制限が多い | systemd と連携した安全な委任が可能 |

v2 の委任モデルは、rootless コンテナの実現に貢献しています

---

## コンテナランタイムの対応

| ランタイム / ツール | v1 対応 | v2 対応 | 備考                                   |
| ------------------- | ------- | ------- | -------------------------------------- |
| Docker              | あり    | あり    | systemd cgroup ドライバ推奨            |
| Podman              | あり    | あり    | v2 推奨                                |
| containerd          | あり    | あり    | Kubernetes 環境で広く使用              |
| runc                | あり    | あり    | 低レベルランタイムとして両方をサポート |

Docker では cgroup ドライバとして `cgroupfs` と `systemd` の 2 つが選択できます

v2 環境では `systemd` ドライバの使用が推奨されています

---

## 主要ディストリビューションの採用

以下のバージョンから cgroup v2 がデフォルトで有効化されています

| ディストリビューション | cgroup v2 がデフォルトのバージョン |
| ---------------------- | ---------------------------------- |
| Fedora                 | 31 以降                            |
| Ubuntu                 | 21.10 以降                         |
| Debian                 | 11（Bullseye）以降                 |
| RHEL                   | 9 以降                             |
| Arch Linux             | 2021 年 8 月以降                   |

---

## 参考資料

このページの内容は、以下のソースに基づいています

<strong>cgroup</strong>

- [cgroups(7) - Linux manual page](https://man7.org/linux/man-pages/man7/cgroups.7.html)
  - cgroup v1 と v2 の概要、インターフェースファイルの説明

<strong>cgroup v2 カーネルドキュメント</strong>

- [Control Group v2 - Linux kernel documentation](https://docs.kernel.org/admin-guide/cgroup-v2.html)
  - cgroup v2 の設計思想、統一階層、委任モデルの詳細

<strong>Docker</strong>

- [Docker cgroup drivers](https://docs.docker.com/engine/containers/runmetrics/#control-groups)
  - Docker の cgroup ドライバ設定

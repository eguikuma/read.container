<div align="right">
<img src="https://img.shields.io/badge/AI-ASSISTED_STUDY-3b82f6?style=for-the-badge&labelColor=1e293b&logo=bookstack&logoColor=white" alt="AI Assisted Study" />
</div>

# 06-security：セキュリティ

## はじめに

ここまでのトピックで、コンテナの主要な構成要素を学びました

- [01-container](./01-container.md)：namespace と cgroup がコンテナを作る
- [02-oci-and-runtime](./02-oci-and-runtime.md)：ランタイムがコンテナを管理する
- [03-image](./03-image.md)：イメージがコンテナの中身を提供する
- [04-network](./04-network.md)：ネットワークでコンテナが通信する
- [05-storage](./05-storage.md)：ストレージでデータを永続化する

しかし、重要な問題がまだ残っています

コンテナは本当に<strong>安全</strong>でしょうか？

namespace でプロセスを隔離し、cgroup でリソースを制限していますが、コンテナのプロセスはホストと<strong>同じカーネル</strong>上で動いています

コンテナ内のプロセスがカーネルの脆弱性を悪用したら、ホストに影響を与えることができてしまいます

namespace と cgroup だけでは、コンテナの安全性は十分とは言えません

このトピックでは、コンテナのセキュリティを強化する仕組みを学びます

---

## 日常の例え

コンテナのセキュリティを「建物のセキュリティ」に例えてみましょう

<strong>namespace と cgroup（壁と電気の契約容量）</strong>

各部屋に壁（namespace）があり、電気の使用量に上限（cgroup）が設定されています

しかし、壁に穴が空いたり、配線を直接いじったりすれば、これらの制限を回避できてしまいます

<strong>capabilities（キーカード）</strong>

部屋ごとに<strong>必要な機能だけ使えるキーカード</strong>を渡すようなものです

「会議室は使えるが、サーバールームには入れない」── 必要な権限だけを渡すことで、万が一カードを落としても被害を最小限にできます

<strong>seccomp（利用規約）</strong>

建物内の<strong>禁止行為リスト</strong>です

「屋上への立ち入り禁止」「電気設備の操作禁止」── 利用者が絶対にやってはいけない操作を明示的に禁止します

<strong>AppArmor / SELinux（監視カメラとアクセスログ）</strong>

すべての行動を<strong>監視し、ルール違反を強制的にブロック</strong>する仕組みです

たとえ管理者権限を持っていても、ポリシーで禁止された行動はブロックされます

<strong>rootless コンテナ（入居者として入る）</strong>

建物に「管理者」としてではなく「入居者」として入るようなものです

入居者には管理者の権限がないため、たとえ部屋から出ても、建物全体を支配することはできません

---

## このページで学ぶこと

このページでは、以下の概念を学びます

<strong>権限の制限</strong>

- <strong>Linux capabilities</strong>
  - root の権限を細分化して必要最小限に制限する仕組み
- <strong>seccomp</strong>
  - コンテナが呼び出せるシステムコールをフィルタリングする仕組み

<strong>アクセス制御</strong>

- <strong>AppArmor / SELinux</strong>
  - 強制アクセス制御によるコンテナの行動制限

<strong>実行権限の分離</strong>

- <strong>rootless コンテナ</strong>
  - root 権限なしでコンテナを実行する仕組み

<strong>セキュリティの全体像</strong>

- <strong>多層防御</strong>
  - 複数のセキュリティ機構を組み合わせてコンテナを保護する考え方

---

## 目次

1. [なぜ namespace と cgroup だけでは不十分か](#なぜ-namespace-と-cgroup-だけでは不十分か)
2. [Linux capabilities](#linux-capabilities)
3. [seccomp](#seccomp)
4. [AppArmor と SELinux](#apparmor-と-selinux)
5. [rootless コンテナ](#rootless-コンテナ)
6. [多層防御](#多層防御)
7. [次のトピックへ](#次のトピックへ)
8. [用語集](#用語集)
9. [参考資料](#参考資料)

---

## なぜ namespace と cgroup だけでは不十分か

namespace はプロセスが「何を見えるか」を制限し、cgroup は「どれだけリソースを使えるか」を制限します

しかし、以下のリスクはこれらだけでは防げません

| リスク               | 説明                                                                                                                       |
| -------------------- | -------------------------------------------------------------------------------------------------------------------------- |
| カーネルの脆弱性     | コンテナのプロセスはホストのカーネルに直接システムコールを発行する。カーネルの脆弱性を悪用してホストに侵入する可能性がある |
| 過剰な権限           | コンテナ内で root として動作するプロセスは、namespace 外の操作を試みることができる                                         |
| 危険なシステムコール | すべてのシステムコールが使えると、カーネルモジュールのロードなど危険な操作が可能になる                                     |

コンテナのセキュリティは、namespace と cgroup に<strong>追加の制限</strong>を重ねることで強化されます

---

## Linux capabilities

伝統的な UNIX では、プロセスの権限は「root（すべての権限）」か「一般ユーザー（制限された権限）」の二択でした

<strong>Linux capabilities</strong> は、root の権限を<strong>細かく分割</strong>し、プロセスに必要な権限だけを付与する仕組みです

### capabilities の例

| capability           | 許可する操作                                                   |
| -------------------- | -------------------------------------------------------------- |
| CAP_NET_BIND_SERVICE | 1024 番未満の特権ポートにバインドする                          |
| CAP_NET_ADMIN        | ネットワーク設定を変更する（ルーティング、ファイアウォール等） |
| CAP_SYS_ADMIN        | さまざまな管理操作（マウント、namespace 操作等）               |
| CAP_SYS_PTRACE       | 他のプロセスをトレース（デバッグ）する                         |
| CAP_CHOWN            | ファイルの所有者を変更する                                     |
| CAP_DAC_OVERRIDE     | ファイルのアクセス権限チェックをバイパスする                   |
| CAP_SYS_MODULE       | カーネルモジュールをロード・アンロードする                     |
| CAP_SYS_BOOT         | システムを再起動する                                           |

### Docker のデフォルト capabilities

Docker はコンテナ起動時に、root 権限から<strong>不要な capabilities を削除</strong>します

デフォルトで許可される capabilities は限定されています

<strong>デフォルトで許可される capabilities（一部）</strong>

| capability              | 理由                                                   |
| ----------------------- | ------------------------------------------------------ |
| CAP_CHOWN               | ファイル所有権の変更（アプリケーションの初期化に必要） |
| CAP_NET_BIND_SERVICE    | 特権ポートへのバインド（Web サーバー等に必要）         |
| CAP_SETUID / CAP_SETGID | UID / GID の変更（プロセスの権限降格に必要）           |
| CAP_KILL                | プロセスへのシグナル送信                               |

<strong>デフォルトで拒否される capabilities（一部）</strong>

| capability     | 理由                                                   |
| -------------- | ------------------------------------------------------ |
| CAP_SYS_ADMIN  | 強力すぎる（マウント、namespace 操作、多数の管理機能） |
| CAP_SYS_MODULE | カーネルモジュールのロードはホストに影響する           |
| CAP_SYS_BOOT   | ホストの再起動が可能になる                             |
| CAP_NET_ADMIN  | ホストのネットワーク設定を変更できる                   |

### capabilities の追加と削除

```
docker run --cap-drop ALL --cap-add NET_BIND_SERVICE nginx
```

この例では、すべての capabilities を削除した上で、NET_BIND_SERVICE だけを追加しています

<strong>最小権限の原則</strong>：アプリケーションに必要な capabilities だけを付与することで、セキュリティリスクを最小化できます

---

## seccomp

<strong>seccomp</strong>（Secure Computing Mode）は、プロセスが呼び出せるシステムコールを<strong>フィルタリング</strong>するカーネル機能です

### なぜシステムコールのフィルタリングが必要か

Linux カーネルには 300 以上のシステムコールがあります

しかし、一般的なアプリケーションが使用するシステムコールはその一部です

使わないシステムコールの中には、カーネルの脆弱性を悪用するために利用されうるものがあります

seccomp を使えば、コンテナが呼び出せるシステムコールを必要なものだけに制限できます

### Docker のデフォルト seccomp プロファイル

Docker はデフォルトで seccomp プロファイルを適用し、危険なシステムコールをブロックします

<strong>ブロックされるシステムコールの例</strong>

| システムコール              | ブロックの理由                         |
| --------------------------- | -------------------------------------- |
| mount / umount              | ホストのファイルシステムを操作できる   |
| reboot                      | ホストを再起動できる                   |
| init_module / delete_module | カーネルモジュールをロード・削除できる |
| clock_settime               | ホストの時刻を変更できる               |
| swapon / swapoff            | ホストのスワップ設定を変更できる       |

※ 一部のシステムコールは capabilities との組み合わせで許可・拒否が決まります

たとえば mount は、seccomp のフィルタに加えて CAP_SYS_ADMIN がないことでも防がれています

<strong>許可されるシステムコールの例</strong>

| システムコール   | 用途                         |
| ---------------- | ---------------------------- |
| read / write     | ファイルの読み書き           |
| open / close     | ファイルのオープン・クローズ |
| socket / connect | ネットワーク通信             |
| fork / exec      | プロセスの作成               |
| mmap / brk       | メモリ管理                   |

### seccomp と capabilities の違い

|          | capabilities                       | seccomp                                  |
| -------- | ---------------------------------- | ---------------------------------------- |
| 制御対象 | 権限の種類（何をする権限があるか） | システムコール（どの操作を呼び出せるか） |
| 粒度     | 約 40 種類の capability            | 300 以上のシステムコール                 |
| 目的     | root の権限を細分化                | カーネルへの攻撃面を縮小                 |

capabilities は「何をする権限があるか」を制御し、seccomp は「カーネルのどの機能を呼び出せるか」を制御します

両方を組み合わせることで、より強固なセキュリティが実現されます

---

## AppArmor と SELinux

<strong>AppArmor</strong> と <strong>SELinux</strong> は、Linux カーネルの<strong>強制アクセス制御</strong>（MAC：Mandatory Access Control）機構です

通常のファイルパーミッション（DAC：Discretionary Access Control）は、ファイルの所有者が権限を設定します

MAC はシステム管理者が定義したポリシーを<strong>強制的に</strong>適用し、root であっても回避できません

### AppArmor

AppArmor は<strong>パスベース</strong>のアクセス制御を行います

プロファイルを定義して、プロセスがアクセスできるファイルパスやネットワーク操作を制限します

Docker は Ubuntu 環境ではデフォルトで AppArmor プロファイルを適用します

### SELinux

SELinux は<strong>ラベルベース</strong>のアクセス制御を行います

すべてのファイルとプロセスにラベル（セキュリティコンテキスト）を付与し、ラベル間のアクセスルールを定義します

Red Hat 系のディストリビューション（RHEL、Fedora、CentOS）で標準的に使われています

### コンテナとの関係

| 機構     | Docker での扱い                                                             |
| -------- | --------------------------------------------------------------------------- |
| AppArmor | Ubuntu 環境でデフォルト適用。コンテナの動作を制限するプロファイルを自動適用 |
| SELinux  | Red Hat 系環境で利用可能。コンテナのファイルアクセスをラベルで制御          |

---

## rootless コンテナ

<strong>rootless コンテナ</strong>は、root 権限を一切使わずにコンテナを実行する仕組みです

### なぜ rootless が重要か

従来の Docker では、Docker デーモン（dockerd）が root 権限で動作していました

```
従来の構成：

  dockerd（root 権限で動作）
      │
      ├── コンテナ A（root → namespace 内の root）
      └── コンテナ B（root → namespace 内の root）
```

この構成では、以下のセキュリティリスクがあります

- Docker ソケット（/var/run/docker.sock）へのアクセスは、事実上の root 権限と同等
- コンテナが namespace を脱出した場合、ホストの root 権限を取得できる

### rootless の仕組み

rootless コンテナは <strong>User namespace</strong> を使って、一般ユーザーの権限でコンテナを実行します

```
rootless の構成：

  Podman（一般ユーザー権限で動作）
      │
      ├── コンテナ A（コンテナ内 root = ホストの UID 100000）
      └── コンテナ B（コンテナ内 root = ホストの UID 100000）
```

[01-container](./01-container.md) で学んだ User namespace により、コンテナ内の root（UID 0）がホストの一般ユーザー（例: UID 100000）にマッピングされます

コンテナ内では root として動作しているように見えますが、ホストから見ると一般ユーザーです

コンテナが namespace を脱出しても、ホストでは一般ユーザーの権限しか持ちません

### Docker と Podman の rootless 対応

|                    | Docker                                                     | Podman                                                     |
| ------------------ | ---------------------------------------------------------- | ---------------------------------------------------------- |
| rootless サポート  | あり（追加設定が必要）                                     | あり（デフォルト）                                         |
| デーモン           | rootless dockerd を別途起動する必要がある                  | デーモン不要                                               |
| ネットワーク       | slirp4netns または pasta を使用                            | slirp4netns または pasta を使用                            |
| ストレージドライバ | fuse-overlayfs（カーネル 5.11 以降は overlay2 も利用可能） | fuse-overlayfs（カーネル 5.11 以降は overlay2 も利用可能） |

### rootless の制限

rootless コンテナにはいくつかの制限があります

| 制限                                          | 理由                                                                                                                   |
| --------------------------------------------- | ---------------------------------------------------------------------------------------------------------------------- |
| 特権ポート（1024 未満）を直接バインドできない | 一般ユーザーには特権ポートをバインドする権限がない                                                                     |
| 一部のネットワーク機能が制限される            | iptables の操作に root 権限が必要                                                                                      |
| ping が動作しない場合がある                   | raw ソケットの作成に権限が必要（ただし多くの最近のディストリビューションでは `ping_group_range` の設定により動作する） |

---

## 多層防御

コンテナのセキュリティは、単一の仕組みに依存するのではなく、<strong>複数の仕組みを重ねる</strong>ことで実現されます

この考え方を<strong>多層防御</strong>（Defense in Depth）と呼びます

### コンテナのセキュリティレイヤ

```
┌─────────────────────────────────────────┐
│ レイヤ 6: rootless コンテナ              │  ← 実行権限の分離
├─────────────────────────────────────────┤
│ レイヤ 5: AppArmor / SELinux            │  ← 強制アクセス制御
├─────────────────────────────────────────┤
│ レイヤ 4: seccomp                        │  ← システムコールのフィルタリング
├─────────────────────────────────────────┤
│ レイヤ 3: capabilities                   │  ← 権限の最小化
├─────────────────────────────────────────┤
│ レイヤ 2: cgroup                         │  ← リソースの制限
├─────────────────────────────────────────┤
│ レイヤ 1: namespace                      │  ← プロセスの隔離
└─────────────────────────────────────────┘
```

各レイヤの役割をまとめると以下のようになります

| レイヤ | 仕組み             | 防ぐもの                                       |
| ------ | ------------------ | ---------------------------------------------- |
| 1      | namespace          | プロセス間の相互干渉                           |
| 2      | cgroup             | リソースの枯渇（DoS）                          |
| 3      | capabilities       | 不要な root 権限の行使                         |
| 4      | seccomp            | 危険なシステムコールの呼び出し                 |
| 5      | AppArmor / SELinux | ポリシー外のファイルアクセスやネットワーク操作 |
| 6      | rootless           | ホストの root 権限の取得                       |

### なぜ多層防御が必要か

セキュリティの世界では「完璧な防御は存在しない」が前提です

1つのレイヤに脆弱性が見つかっても、他のレイヤが攻撃を食い止めます

たとえば、namespace の脆弱性でプロセスが脱出しても、rootless であればホストでは一般ユーザーの権限しか持ちません

capabilities が最小限であれば、危険な操作は制限されます

seccomp でシステムコールがフィルタリングされていれば、カーネルの脆弱性を悪用する手段も限られます

---

## 次のトピックへ

このトピックでは、以下のことを学びました

- namespace と cgroup だけではコンテナのセキュリティは不十分である
- capabilities が root の権限を細分化し、不要な権限を削除する
- seccomp がシステムコールをフィルタリングし、カーネルへの攻撃面を縮小する
- AppArmor / SELinux が強制アクセス制御を提供する
- rootless コンテナが User namespace で実行権限を分離する
- 多層防御により、複数のセキュリティ機構を重ねてコンテナを保護する

ここまでで、単一コンテナの仕組みをすべて学びました

最後のトピックでは、これまでの知識を統合して、<strong>複数のコンテナを組み合わせる</strong>方法を学びます

次のトピック [07-compose](./07-compose.md) では、<strong>Docker Compose</strong> を使った複数コンテナの定義と管理を学びます

---

## 用語集

| 用語                    | 説明                                                                                                 |
| ----------------------- | ---------------------------------------------------------------------------------------------------- |
| Linux capabilities      | root の権限を約 40 種類に細分化し、プロセスに必要な権限だけを付与する仕組み                          |
| CAP_SYS_ADMIN           | 最も強力な capability。マウント、namespace 操作など多数の管理機能を許可する                          |
| CAP_NET_BIND_SERVICE    | 1024 番未満の特権ポートにバインドすることを許可する capability                                       |
| 最小権限の原則          | プロセスに必要最小限の権限だけを付与するセキュリティ原則                                             |
| seccomp                 | プロセスが呼び出せるシステムコールをフィルタリングするカーネル機能                                   |
| seccomp プロファイル    | 許可・拒否するシステムコールを定義した設定                                                           |
| AppArmor                | パスベースの強制アクセス制御（MAC）機構。Ubuntu で標準的に使われる                                   |
| SELinux                 | ラベルベースの強制アクセス制御（MAC）機構。Red Hat 系で標準的に使われる                              |
| 強制アクセス制御（MAC） | システム管理者が定義したポリシーを強制的に適用するアクセス制御。root でも回避できない                |
| 任意アクセス制御（DAC） | ファイルの所有者が権限を設定する従来のアクセス制御                                                   |
| rootless コンテナ       | User namespace を使い、root 権限なしでコンテナを実行する仕組み                                       |
| User namespace          | UID / GID のマッピングを提供する namespace。コンテナ内の root をホストの一般ユーザーにマッピングする |
| slirp4netns             | rootless コンテナ向けのユーザー空間ネットワークスタック                                              |
| pasta                   | rootless コンテナ向けのネットワーク接続ツール                                                        |
| fuse-overlayfs          | rootless コンテナ向けのストレージドライバ。ユーザー権限で overlay filesystem を使用する              |
| 多層防御                | 複数のセキュリティ機構を重ねて、1つが突破されても他で防ぐセキュリティ戦略                            |
| 攻撃面                  | 攻撃者が悪用できるシステムの表面積。小さいほど安全                                                   |

---

## 参考資料

このページの内容は、以下のソースに基づいています

<strong>capabilities</strong>

- [capabilities(7) - Linux manual page](https://man7.org/linux/man-pages/man7/capabilities.7.html)
  - Linux capabilities の全一覧と動作の説明

<strong>seccomp</strong>

- [seccomp(2) - Linux manual page](https://man7.org/linux/man-pages/man2/seccomp.2.html)
  - seccomp システムコールフィルタリングの仕組み

<strong>User namespace</strong>

- [user_namespaces(7) - Linux manual page](https://man7.org/linux/man-pages/man7/user_namespaces.7.html)
  - User namespace と UID / GID マッピングの仕組み

<strong>Docker セキュリティ</strong>

- [Docker security](https://docs.docker.com/engine/security/)
  - Docker のセキュリティ機能（capabilities、seccomp、AppArmor）
- [Docker Engine security](https://docs.docker.com/engine/security/#linux-kernel-capabilities)
  - Docker のデフォルト capabilities とセキュリティ設定

<strong>Podman rootless</strong>

- [Podman rootless](https://docs.podman.io/en/latest/markdown/podman.1.html)
  - Podman の rootless モードの仕組みと制限

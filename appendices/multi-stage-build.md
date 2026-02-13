<div align="right">
<img src="https://img.shields.io/badge/AI-ASSISTED_STUDY-3b82f6?style=for-the-badge&labelColor=1e293b&logo=bookstack&logoColor=white" alt="AI Assisted Study" />
</div>

# appendix：マルチステージビルド

## はじめに

[03-image](../03-image.md) では、Dockerfile の各命令がレイヤを生成し、それらが積み重なってイメージになることを学びました

- FROM でベースイメージを指定
- RUN でコマンドを実行
- COPY でファイルを追加する

それぞれがレイヤを生成します

しかし、アプリケーションを<strong>ビルド</strong>するために必要なツール（コンパイラ、パッケージマネージャ等）は、<strong>実行時</strong>には不要です

ビルドツールがイメージに残ると、イメージサイズが大きくなり、攻撃面も広がります

この補足資料では、FROM を複数回使うことでこの問題を解決する<strong>マルチステージビルド</strong>の仕組みを学びます

---

## 単一ステージビルドの問題

Go アプリケーションを例に考えてみましょう

```dockerfile
FROM golang:1.23

WORKDIR /app
COPY . .
RUN go build -o myapp

CMD ["/app/myapp"]
```

この Dockerfile でビルドすると、最終イメージには以下がすべて含まれます

| 含まれるもの           | 実行時に必要か |
| ---------------------- | -------------- |
| Go コンパイラ          | 不要           |
| Go の標準ライブラリ    | 不要           |
| ソースコード           | 不要           |
| ビルド時の中間ファイル | 不要           |
| ビルドされたバイナリ   | 必要           |

Go コンパイラだけで数百 MB あります

最終的に必要なのはビルドされたバイナリ（数 MB〜数十 MB）だけですが、イメージ全体は 1 GB 近くになります

---

## マルチステージビルドの仕組み

マルチステージビルドでは、Dockerfile 内で<strong>FROM を複数回</strong>使います

各 FROM が新しい<strong>ビルドステージ</strong>を開始し、最終的な FROM のステージだけが最終イメージになります

### Dockerfile の例

```dockerfile
FROM golang:1.23 AS builder

WORKDIR /app
COPY . .
RUN go build -o myapp

FROM debian:bookworm-slim

COPY --from=builder /app/myapp /usr/local/bin/myapp

CMD ["myapp"]
```

### 各ステージの役割

| ステージ              | ベースイメージ       | 役割                                 |
| --------------------- | -------------------- | ------------------------------------ |
| builder（ステージ 1） | golang:1.23          | ソースコードをコンパイルする         |
| 最終ステージ          | debian:bookworm-slim | ビルド済みバイナリだけを含む実行環境 |

### COPY --from の動作

`COPY --from=builder` は、builder ステージのファイルシステムからファイルをコピーします

これは [03-image](../03-image.md) で学んだ COPY 命令の拡張です

通常の COPY はホストのファイルをイメージにコピーしますが、`--from` を指定すると別のステージのファイルシステムからコピーします

---

## レイヤ構造の観点

[03-image](../03-image.md) で学んだレイヤ構造の知識を使って、マルチステージビルドを理解しましょう

### builder ステージのレイヤ

```
レイヤ 3: RUN go build（バイナリを生成）
レイヤ 2: COPY . .（ソースコードをコピー）
レイヤ 1: golang:1.23（Go コンパイラ + Debian）
```

### 最終ステージのレイヤ

```
レイヤ 2: COPY --from=builder（バイナリだけをコピー）
レイヤ 1: debian:bookworm-slim（最小限の Debian）
```

最終イメージには builder ステージのレイヤは<strong>一切含まれません</strong>

builder ステージはビルド時にのみ使用され、最終イメージには最終ステージのレイヤだけが残ります

### サイズの比較

| ビルド方法     | イメージサイズの目安 |
| -------------- | -------------------- |
| 単一ステージ   | 約 1 GB              |
| マルチステージ | 約 80 MB             |

---

## scratch と distroless

マルチステージビルドの最終ステージでは、最小限のベースイメージを使うことでさらにイメージサイズを削減できます

### scratch

<strong>scratch</strong> は、完全に空のベースイメージです

OS のファイル（/bin、/lib、/etc）が一切含まれていません

Go のように静的バイナリを生成できる言語では、scratch をベースに使えます

```dockerfile
FROM golang:1.23 AS builder

WORKDIR /app
COPY . .
RUN CGO_ENABLED=0 go build -o myapp

FROM scratch

COPY --from=builder /app/myapp /myapp

CMD ["/myapp"]
```

scratch ベースのイメージには、バイナリ以外何も含まれないため、イメージサイズが数 MB になります

ただし、シェル（/bin/sh）もないため、`docker exec` でシェルに入ることができません

### distroless

<strong>distroless</strong> は、Google が提供する最小限の実行環境イメージです

アプリケーションの実行に必要な最小限のファイル（libc、CA 証明書等）だけを含みます

| イメージ   | 含まれるもの                                   | シェル |
| ---------- | ---------------------------------------------- | ------ |
| scratch    | 何もない                                       | なし   |
| distroless | libc、CA 証明書、タイムゾーンデータ等          | なし   |
| slim       | パッケージマネージャを除いた OS の基本ファイル | あり   |
| 通常       | OS の基本ファイル + パッケージマネージャ       | あり   |

distroless はシェルを含まないため、攻撃者がコンテナに侵入してもシェルを使った操作ができません

これはセキュリティ上の利点です（[06-security](../06-security.md) の多層防御の考え方に通じます）

---

## 参考資料

このページの内容は、以下のソースに基づいています

<strong>Docker</strong>

- [Multi-stage builds](https://docs.docker.com/build/building/multi-stage/)
  - マルチステージビルドの公式ドキュメント
- [Dockerfile reference - FROM](https://docs.docker.com/reference/dockerfile/#from)
  - FROM 命令と AS によるステージ名の指定

<strong>OCI</strong>

- [OCI Image Specification](https://github.com/opencontainers/image-spec/blob/main/spec.md)
  - コンテナイメージのレイヤ構造の標準仕様

<strong>distroless</strong>

- [distroless - GitHub](https://github.com/GoogleContainerTools/distroless)
  - Google が提供する最小限のコンテナイメージ

# uv 依存性管理ハンズオン

uv の依存性管理（`pyproject.toml` / `uv.lock`）の役割を、手を動かして理解するためのリポジトリです。

## 前提

- uv がインストール済みであること（`uv --version` で確認）
- このリポジトリは `uv init` 済みの状態です

## ハンズオン手順

### Step 0: `uv init` 直後の状態を確認する

まず、現在のファイル構成を確認しましょう。

```bash
cat pyproject.toml
ls -la
```

**`pyproject.toml` の各フィールド:**

| フィールド | 説明 |
|---|---|
| `[project]` | プロジェクトメタデータのセクション（[PEP 621](https://peps.python.org/pep-0621/) で標準化） |
| `name` | パッケージ名。`uv init` 時にディレクトリ名から自動設定される |
| `version` | パッケージのバージョン |
| `description` | パッケージの説明文 |
| `requires-python` | 対応する Python バージョンの範囲。`.python-version` とは別の役割（後述） |
| `dependencies` | 直接依存するパッケージのリスト。初期状態では空 |

> **`requires-python` と `.python-version` の違い:**
> - `requires-python` — このパッケージが動作する Python バージョンの**範囲**を宣言（例: `>=3.13`）
> - `.python-version` — 開発時に使う Python の**具体的なバージョン**を固定（例: `3.13`）

**注目ポイント:**

- `pyproject.toml` の `dependencies = []` が空であること
- `uv.lock` はまだ存在しないこと
- `.python-version` で Python バージョンが固定されていること

---

### Step 1: `uv add requests` — 最初のパッケージを追加する

```bash
uv add requests
```

実行後、以下を確認してください。

```bash
# pyproject.toml の変化
cat pyproject.toml

# uv.lock が生成されたことを確認
ls -la uv.lock

# uv.lock の中身を確認（最初は短いので全体を読める）
cat uv.lock

# .venv が作成されたことを確認
ls -la .venv/
```

**`uv.lock` の読み方:**

`uv.lock` は TOML 形式で、`[[package]]` ブロックの繰り返しで構成されています。

```toml
[[package]]
name = "requests"          # パッケージ名
version = "2.33.1"         # 固定されたバージョン（範囲ではなく1つに確定）
source = { registry = "https://pypi.org/simple" }  # 取得元（PyPI）
dependencies = [           # requests が動作するために必要なパッケージ
    { name = "certifi" },
    { name = "charset-normalizer" },
    { name = "idna" },
    { name = "urllib3" },
]
sdist = { url = "...", hash = "sha256:...", size = 134120 }  # ソース配布物
wheels = [                 # ビルド済み配布物（環境ごとに複数ある）
    { url = "...", hash = "sha256:..." },
]
```

| フィールド | 説明 |
|---|---|
| `name` / `version` | パッケージ名と固定バージョン |
| `source` | パッケージの取得元（通常は PyPI） |
| `dependencies` | このパッケージが動作するために必要な他のパッケージ。uv が自動的に解決・インストールする |
| `sdist` | ソースコード配布物の URL とハッシュ |
| `wheels` | ビルド済み配布物の URL とハッシュ（OS/Python バージョンごとに複数） |
| `hash` | 改ざん検知のためのハッシュ値。`uv sync` 時にダウンロードしたファイルと照合される |

> **PyPI（Python Package Index）とは:**
> Python パッケージの公開レジストリ（https://pypi.org）。`uv add` や `pip install` でパッケージをインストールすると、デフォルトでここからダウンロードされる。npm における npmjs.com、Ruby における RubyGems.org に相当する。

> **sdist と wheel の違い:**
>
> | | sdist（Source Distribution） | wheel（Built Distribution） |
> |---|---|---|
> | 中身 | ソースコード + ビルドスクリプト | ビルド済みのファイル一式 |
> | 拡張子 | `.tar.gz` | `.whl` |
> | インストール時 | ビルドが必要（C拡張がある場合はコンパイラも必要） | 展開するだけ（高速） |
> | 用途 | フォールバック・ソースコード参照用 | 通常のインストールで使われる |
>
> uv は wheel を優先的に使い、対応する wheel がない環境でのみ sdist からビルドする。そのため `uv.lock` には両方の URL が記録されている。

> **wheels が大量にあるパッケージ**: `charset-normalizer` のように C 拡張を含むパッケージは、OS × CPU × Python バージョンの組み合わせごとに wheel が用意されるため、`uv.lock` の行数が大きくなります。純 Python パッケージ（`py3-none-any.whl`）は 1 つだけです。

**直接依存と推移的依存:**

| 用語 | 定義 | 今回の例 |
|---|---|---|
| 直接依存（direct dependency） | 自分が明示的に `uv add` したパッケージ | `requests` |
| 推移的依存（transitive dependency） | 直接依存が動作するために内部で必要としているパッケージ | `certifi`, `charset-normalizer`, `idna`, `urllib3` |

この 2 つは記録される場所が異なります。

- `pyproject.toml` — **直接依存のみ**を記録する（自分が何を使うかの宣言）
- `uv.lock` — **直接依存 + 推移的依存のすべて**を記録する（依存ツリー全体のスナップショット）

**注目ポイント:**

- `pyproject.toml` の `dependencies` に `requests` のバージョン指定がどう書かれるか（`>=2.x` のような範囲指定）
- `uv.lock` には `requests` 本体だけでなく、推移的依存（`certifi`, `charset-normalizer`, `idna`, `urllib3`）も記録されていること
- `uv.lock` には各パッケージの **正確なバージョン** と **ハッシュ** が記録されていること
- `pyproject.toml`（範囲指定）と `uv.lock`（完全固定）の **役割の違い** を意識する

---

### Step 2: `uv add fastapi` — 推移的依存が膨らむ様子を確認する

```bash
uv add fastapi
```

実行後、以下を確認してください。

```bash
# pyproject.toml の変化
cat pyproject.toml

# uv.lock の行数を確認（大幅に増える）
wc -l uv.lock

# どんなパッケージが追加されたか確認
grep 'name = ' uv.lock
```

**注目ポイント:**

- `pyproject.toml` の `dependencies` に追加されたのは `fastapi` の 1 行だけ
- しかし `uv.lock` には `starlette`, `pydantic`, `pydantic-core`, `anyio`, `sniffio`, `typing-extensions` など多数のパッケージが追加される
- **直接依存**（自分が `uv add` したもの）と **推移的依存**（依存の依存）の違いを体感する
- `uv.lock` の行数が Step 1 から大幅に増えることで、ロックファイルが「依存ツリー全体のスナップショット」であることがわかる

---

### Step 3: `uv add --dev pytest` — dev 依存の分離を確認する

```bash
uv add --dev pytest
```

実行後、以下を確認してください。

```bash
# pyproject.toml の変化
cat pyproject.toml

# dev 依存がどこに記録されたか確認
grep -A5 'dependency-groups' pyproject.toml
```

**注目ポイント:**

`pyproject.toml` を見ると、`pytest` は `requests` や `fastapi` とは **別の場所** に記録されています。

```toml
[project]
dependencies = [
    "requests>=2.x",
    "fastapi>=0.x",
]

[dependency-groups]
dev = [
    "pytest>=8.x",
]
```

- `[project] dependencies` — 本番で動かすのに必要なパッケージ（`requests`, `fastapi`）
- `[dependency-groups] dev` — 開発時にだけ使うパッケージ（`pytest`）

**なぜ分けるのか？**

本番環境にデプロイするとき、テスト用の `pytest` は不要です。分離しておくことで、本番では `uv sync --no-group dev` のように dev 依存を除外してインストールできます。これにより本番環境を軽量に保てます。

**`uv.lock` はどうなるか？**

`uv.lock` には dev 依存も含めて **すべてのパッケージ** が記録されます。ロックファイルは「どの環境でインストールするか」ではなく「何が存在するか」の完全な記録なので、dev / 本番の区別はありません。実際にどれをインストールするかは `uv sync` のオプションで制御します。

---

### Step 4（応用）: ロックファイルの再現性を確認する

ここまでの変更を体感したら、ロックファイルの再現性を確認してみましょう。

```bash
# 現在インストールされているパッケージを確認
uv pip list

# .venv を削除
rm -rf .venv

# 削除後にもう一度確認（.venv がないのでエラーになる）
uv pip list

# uv.lock から .venv を再作成
uv sync

# 再作成後に確認（削除前と同じ結果になることを確認）
uv pip list
```

**注目ポイント:**

- `.venv` を削除すると `uv pip list` が失敗する → パッケージは `.venv` の中にしか存在しない
- `uv sync` 後の `uv pip list` が削除前とまったく同じ結果になる → `uv.lock` から環境が完全に再現された
- `uv.lock` があれば、誰がいつ `uv sync` しても同じバージョンの依存がインストールされる
- これが「再現可能なビルド」の基盤であり、チーム開発で `uv.lock` を Git にコミットする理由

---

## 初期状態に戻す

ハンズオンをやり直したい場合は、以下のコマンドで `uv init` 直後の状態にリセットできます。

```bash
git checkout .
git clean -fd
rm -rf .venv
```

| コマンド | 効果 |
|---|---|
| `git checkout .` | 変更されたファイル（`pyproject.toml`）を初期コミットの状態に戻す |
| `git clean -fd` | Git 管理外のファイル（`uv.lock`）を削除する |
| `rm -rf .venv` | 仮想環境を削除する |

---

## まとめ

| ファイル | 役割 | Git 管理 |
|---|---|---|
| `pyproject.toml` | 直接依存の宣言（バージョン範囲） | する |
| `uv.lock` | 全依存の完全なスナップショット（バージョン+ハッシュ固定） | する |
| `.venv/` | ローカルの仮想環境（`uv.lock` から再現可能） | しない |

### 覚えておくこと

- `pyproject.toml` は「何が必要か」を宣言する
- `uv.lock` は「実際に何を使うか」を固定する
- この 2 つの分離により、柔軟性（バージョン範囲）と再現性（完全固定）を両立している

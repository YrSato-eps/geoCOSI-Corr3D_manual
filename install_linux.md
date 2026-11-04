# geoCosiCorr3D 環境構築手順書（研究室 Linux 計算機・SSH 経由）

> **対象環境**：Rocky Linux 9.7（x86_64）、一般ユーザー権限  
> **元手順書作成日**：2026-10-09  
> **Markdown 化**：2026-11-04  
> **元ファイル**：`geocosicorr3d_rockylinux_setup_updated.txt`

## 目的

研究室の Linux 計算機に SSH で接続し、geoCosiCorr3D の **correlation 計算**を実行できる環境を用意する。

システム全体の設定を変更せず、ホームディレクトリに Miniconda と geoCosiCorr3D を配置する。

## 1. SSH でログイン

```bash
ssh ユーザー名@ホスト名
```

## 2. 計算機環境の確認（読み取り専用）

以下のコマンドで、OS、アーキテクチャ、CPU、メモリ、ディスク容量、および既存のコマンドを確認する。

```bash
cat /etc/os-release
uname -m
nproc
free -h
df -h ~
command -v conda
conda --version
python3 --version
git --version
```

### 今回確認した環境

| 項目 | 確認結果 |
| --- | --- |
| OS | Rocky Linux 9.7 |
| アーキテクチャ | x86_64 |
| CPU | 20 コア |
| メモリ | 30 GiB |
| `/home` の空き容量 | 約 101 GB（確認時点） |
| システムの Python | 3.9.25 |
| 初期状態 | `conda`、`git` は未インストール |

> **補足**  
> `conda` や `git` が見つからない場合、関連コマンドは失敗することがあるが、この段階では問題ない。OS から `Install package ...? [N/y]` と表示されても、今回の作業では `N` を選び、OS 全体へのインストールは行わない。

## 3. Miniconda のインストール

まず、ダウンロードに使用するコマンドとインストール先の状態を確認する。

```bash
cd ~
command -v curl
command -v wget
ls -ld ~/miniconda3
```

`~/miniconda3` が存在しないことを確認した上で、`curl` が利用できる場合は以下を実行する。

```bash
curl -L -o Miniconda3-latest-Linux-x86_64.sh \
  https://repo.anaconda.com/miniconda/Miniconda3-latest-Linux-x86_64.sh

bash Miniconda3-latest-Linux-x86_64.sh -b -p "$HOME/miniconda3"
```

Conda を現在のシェルで利用可能にする。

```bash
source ~/miniconda3/etc/profile.d/conda.sh
conda --version
```

今回確認した Conda バージョン：`26.7.1`

> **注意**  
> Miniconda がすでにインストールされている場合、この節のインストール操作を繰り返す必要はない。

## 4. geoCosiCorr3D ソースコードの取得（Git 不要）

Git を使わず、GitHub 上のソースコードを ZIP 形式で取得する。

```bash
cd ~

curl -L -o Geospatial-COSICorr3D.zip \
  https://github.com/SaifAati/Geospatial-COSICorr3D/archive/refs/heads/main.zip

python3 -m zipfile -e Geospatial-COSICorr3D.zip .

cd ~/Geospatial-COSICorr3D-main
ls
```

`install_cosicorr.sh`、`geoCosiCorr3D.yml`、`scripts`、`lib` などがあるか確認する。

> **注意**  
> `main` ブランチは今後更新されるため、再構築時には依存パッケージのバージョンや手順が変わる可能性がある。

## 5. geoCosiCorr3D 専用 Conda 環境の構築

```bash
source ~/miniconda3/etc/profile.d/conda.sh
cd ~/Geospatial-COSICorr3D-main

bash install_cosicorr.sh --conda
```

- Conda の規約への同意を求められた場合は、内容を確認して判断する。
- 所要時間や依存ライブラリの解決状況は環境によって異なる。

## 6. パッケージ本体のインストールと import テスト（重要）

Conda 環境の構築後、Python が `geoCosiCorr3D` パッケージを認識できることを確認する。

環境作成時にパッケージが自動インストールされる場合もあるが、**明示的にインストール・検証する**。

### 6.1 Conda 環境を有効化する

```bash
source ~/miniconda3/etc/profile.d/conda.sh
conda activate geoCosiCorr3D
cd ~/Geospatial-COSICorr3D-main
```

### 6.2 使用中の Python を確認する

```bash
which python
python --version
```

Conda 環境の Python を使用していることを確認する。

### 6.3 パッケージ本体をインストールする

```bash
python -m pip install -e . --no-deps
```

ソースツリーを**編集可能モード（editable mode）**で Python 環境に登録する。`--no-deps` を付け、Conda 環境内の依存関係をこの操作で変更しない。

### 6.4 import テストを行う

```bash
python -c "import geoCosiCorr3D; print(geoCosiCorr3D.__file__)"
```

期待される出力例：

```text
/home/ユーザー名/Geospatial-COSICorr3D-main/geoCosiCorr3D/__init__.py
```

> **確認事項**
>
> - `__file__` が `None` などと表示される構成もあるため、まずエラーの有無で判断する。
> - `ModuleNotFoundError` が出た場合は、`conda activate` の状態と `which python` を確認する。
> - このインストールは通常 1 回だけでよく、SSH で再接続するたびに行う必要はない。

## 7. geoCosiCorr3D CLI の動作確認

```bash
source ~/miniconda3/etc/profile.d/conda.sh
conda activate geoCosiCorr3D
cd ~/Geospatial-COSICorr3D-main

export LD_LIBRARY_PATH="$PWD/lib:${LD_LIBRARY_PATH:-}"

python scripts/cosicorr.py correlate -h
```

`correlate` コマンドのヘルプが表示されれば、**CLI の起動確認は完了**。

> **注意**  
> この段階で確認できるのは CLI の起動のみ。実データを用いた correlation 計算の正常動作は、別途テストする必要がある。

## 8. SSH で再接続した後の基本操作

次回以降は、環境を作り直す必要はない。SSH でログインし、Conda 環境とライブラリパスを設定する。

```bash
ssh ユーザー名@ホスト名

source ~/miniconda3/etc/profile.d/conda.sh
conda activate geoCosiCorr3D
cd ~/Geospatial-COSICorr3D-main

export LD_LIBRARY_PATH="$PWD/lib:${LD_LIBRARY_PATH:-}"

python scripts/cosicorr.py correlate -h
```

## 注意事項

- 上記のパスは、すべてホームディレクトリに環境を構築した場合の例である。
- 共用計算機では、CPU・メモリの利用や長時間ジョブの実行について研究室のルールを守る。
- 大量の画像を配置する前に、`df -h ~` などでディスクの空き容量を確認する。
- SSH 切断後も計算を継続する方法（`tmux` やジョブスケジューラなど）は、別途設定する。
- この手順書は**今回実行した操作の記録**であり、すべての Rocky Linux 環境での再現を保証するものではない。

## 参考

- [SaifAati/Geospatial-COSICorr3D（GitHub）](https://github.com/SaifAati/Geospatial-COSICorr3D)

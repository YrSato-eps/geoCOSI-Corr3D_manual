# geoCosiCorr3D 環境構築手順書（Windows ローカル PC・WSL2 経由）

> **対象環境**：Windows（Home）上の WSL2、Ubuntu 22.04.5 LTS（x86_64）  
> **元の環境構築・動作確認**：2026-09-19  
> **手順書作成日**：2026-11-04  
> **対応文書**：[`install_rockylinux.md`](install_rockylinux.md)（研究室 Linux 向け）

## 目的

Windows 上で **WSL2（Windows Subsystem for Linux）**を使って Ubuntu を動作させ、オープンソース版 geoCosiCorr3D の **correlation 計算**を実行できる環境を用意する。

Windows に直接 Python の各種依存ライブラリを入れるのではなく、Ubuntu 上に Miniconda と geoCosiCorr3D を導入する。以下は、過去にローカル PC で動作させた構成を再現するための手順である。

> **記録と補足の区別**  
> WSL2、`geoCosiCorr3D_cf` 環境、Python 3.9.23、GDAL 3.10.2、CLI の起動と画像相関計算の成功は、過去の作業記録で確認されている。コマンドのうち、環境確認、バックアップ、import テスト等は手順書として安全性・再現性を高めるために補足したもので、当時の全操作を逐語的に再現したログではない。

## 1. Windows に WSL2 と Ubuntu を導入

以下のコマンドは **Windows の PowerShell（管理者として実行）**で入力する。Ubuntu の端末ではない点に注意する。

当時は最初に WSL 本体を導入した。

```powershell
wsl.exe --install --no-distribution
```

必要に応じて PC を再起動し、Windows の仮想化関連機能を有効にする。ハードウェア仮想化が無効の場合は、PC の設定も確認する。その後 Ubuntu 22.04 を導入する。

```powershell
wsl --set-default-version 2
wsl --install -d Ubuntu-22.04
```

Ubuntu を初回起動した際は、案内に従って Linux 側のユーザー名とパスワードを設定する。

導入後、PowerShell で WSL のバージョンを確認する。

```powershell
wsl --list --verbose
```

Ubuntu の `VERSION` が `2` となっていれば WSL2 で動作している。

> **補足**  
> WSL2 と Ubuntu がすでに利用可能なら、この節を繰り返す必要はない。Windows の更新状況によってインストール時の表示や再起動の要否は変わる。

## 2. Ubuntu の環境を確認する

**ここから先は Ubuntu（WSL）のターミナル**で実行する。

```bash
cat /etc/os-release
uname -m
nproc
free -h
python3 --version
command -v git
command -v wget
command -v conda
```

### 当時確認した環境

| 項目 | 確認結果 |
| --- | --- |
| ホスト OS | Windows Home |
| Linux 実行環境 | WSL2、Ubuntu 22.04.5 LTS |
| アーキテクチャ | x86_64 |
| 論理 CPU | 22 スレッド |
| WSL 上のメモリ | 約 14 GiB |
| スワップ | 約 4 GiB |
| 最終的に使用した Python | 3.9.23（Conda 環境内） |
| 最終的に使用した GDAL | 3.10.2（Conda 環境内） |

不足している場合に限り、Git やダウンロード用コマンドを Ubuntu 側に導入する。

```bash
sudo apt update
sudo apt install -y git wget curl
```

> **注意**  
> `sudo` が必要なのは、このように Ubuntu 全体へパッケージを入れる操作である。geoCosiCorr3D 自体は原則としてユーザーのホームディレクトリと Conda 環境に配置する。

## 3. Miniconda のインストール

Ubuntu のホームディレクトリに移動し、すでに導入済みでないか確認する。

```bash
cd ~
ls -ld ~/miniconda3
```

未導入の場合、Miniconda の Linux x86_64 版を取得してインストールする。

```bash
wget https://repo.anaconda.com/miniconda/Miniconda3-latest-Linux-x86_64.sh
bash Miniconda3-latest-Linux-x86_64.sh
```

インストーラではライセンス等の案内を確認し、インストール先を `~/miniconda3` とする。当時はシェルの初期化を許可した。既存のインストールがある場合は上書きしない。

現在のシェルから Conda を利用できるようにする。

```bash
source ~/miniconda3/etc/profile.d/conda.sh
conda --version
```

## 4. geoCosiCorr3D のソースコードを取得

当時は GitHub からクローンし、ソースコードを `~/Geospatial-COSICorr3D` に配置した。

```bash
cd ~
git clone https://github.com/SaifAati/Geospatial-COSICorr3D.git
cd ~/Geospatial-COSICorr3D
ls
```

`geoCosiCorr3D.yml`、`install_cosicorr.sh`、`scripts`、`lib` などが存在することを確認する。

> **注意**  
> GitHub の `main` ブランチは更新される可能性がある。別の日に再インストールする場合、依存関係やファイル構成が当時と一致しないことがある。再現性を重視する場合は、取得時のコミット ID を `git rev-parse HEAD` で記録する。

## 5. Conda 環境ファイルの調整（当時の重要な回避策）

当時、配布されている `geoCosiCorr3D.yml` による環境構築では、**Conda のチャネルの混在**と、**`gdal=3.6.2` の固定**に起因する依存関係の解決問題があった。このため、元のファイルを残したまま、Conda-Forge 専用の `geoCosiCorr3D_cf.yml` を作成した。

### 5.1 元の YAML をコピーする

```bash
cd ~/Geospatial-COSICorr3D
cp geoCosiCorr3D.yml geoCosiCorr3D_cf.yml
```

> **注意**  
> `geoCosiCorr3D_cf.yml` がすでに存在する場合、`cp` を実行すると上書きされる。再実行時は必要に応じて既存ファイルを退避する。

### 5.2 環境名・チャネル・GDAL の指定を変更する

コピーしたファイルを編集し、次のようにする。

**環境名：**

```yaml
name: geoCosiCorr3D_cf
```

**チャネル：** `anaconda` と `defaults` を外し、`conda-forge` のみとする。

```yaml
channels:
  - conda-forge
```

**GDAL の指定：** `gdal=3.6.2` の固定を解除する。

```yaml
dependencies:
  - python=3.9
  - pip=23.2
  - gdal
  # ほかの依存関係は原則として元の YAML を維持
```

以上は変更部分の例であり、**ファイル全体を上の短い YAML で置き換えない**こと。元ファイルのほかの依存パッケージと、末尾の `pip:` セクション（`-e .` を含む）は残す。

以下のコマンドを実行すれば、**環境名の変更・不要なチャネルの削除・GDAL のバージョン固定解除**をまとめて行える。

```bash
# 環境名を変更する
sed -i 's/^name: geoCosiCorr3D$/name: geoCosiCorr3D_cf/' geoCosiCorr3D_cf.yml

# channels から anaconda と defaults を削除する
sed -i '/^[[:space:]]*-[[:space:]]*anaconda[[:space:]]*$/d' geoCosiCorr3D_cf.yml
sed -i '/^[[:space:]]*-[[:space:]]*defaults[[:space:]]*$/d' geoCosiCorr3D_cf.yml

# gdal=3.6.2 のバージョン固定を解除し、gdal にする
sed -i 's/gdal=3\.6\.2/gdal/' geoCosiCorr3D_cf.yml
```

- `sed -i` は指定したファイルを**直接編集**する。
- 2 つのチャネル削除コマンドは、`anaconda` と `defaults` の行だけを削除する。元の YAML にある `conda-forge` の行は残す。
- GDAL のコマンドは、元の YAML に `gdal=3.6.2` と記述されている場合に対応する。
- 以上は**今回のリポジトリの YAML の構成を前提**とした操作であり、ファイルの内容が異なる場合は事前に確認する。

変更後、以下を実行して編集結果を確認する。

```bash
# 変更した行を確認する
grep -nE '^(name:|channels:|[[:space:]]*-[[:space:]]*(anaconda|defaults|conda-forge|gdal))' geoCosiCorr3D_cf.yml

# 元の YAML との差分を確認する（差分が表示されるのが正常）
diff -u geoCosiCorr3D.yml geoCosiCorr3D_cf.yml || true
```

`name: geoCosiCorr3D_cf`、`conda-forge` のみのチャネル、バージョン指定のない `gdal` になっていることを確認する。

### 5.3 Conda のチャネル設定を確認する

環境ファイルとは別に、Conda 本体の設定から `defaults` が読み込まれて衝突する場合がある。まず設定の出所を確認する。

```bash
conda config --show-sources
conda config --show channels
```

当時は `defaults` の混在が解消されず、**`~/miniconda3/.condarc` を取り除くことで解決**した。ただし、既存の `.condarc` には別の重要な設定が含まれる可能性があるので、無条件に削除することは推奨しない。該当ファイルが実際に競合の原因と分かった場合は、まず退避する。

```bash
# ファイルが存在し、問題の原因と確認できた場合のみ実行
mv ~/miniconda3/.condarc ~/miniconda3/.condarc.backup
```

必要に応じて `conda config --remove channels defaults` や `conda config --add channels conda-forge` でチャネル設定を調整する。これらは Conda の設定を書き換えるため、`conda config --show-sources` で変更対象を確認してから実行する。

> **注意**  
> 当時の解決策は、当時の Conda 設定と依存関係に対するものである。ほかの PC で同じエラーが起きるとは限らず、チャネルの変更や GDAL のバージョン固定解除が常に必要とも限らない。

## 6. geoCosiCorr3D 専用 Conda 環境の構築

```bash
source ~/miniconda3/etc/profile.d/conda.sh
cd ~/Geospatial-COSICorr3D

conda env create -f geoCosiCorr3D_cf.yml
```

成功したら、環境を有効化する。

```bash
conda activate geoCosiCorr3D_cf
which python
python --version
gdalinfo --version
```

当時の成功時に確認した環境：

```text
Python 3.9.23
GDAL 3.10.2
```

> **補足**  
> 元 YAML の `pip:` セクションに `-e .` が含まれており、環境作成時にソースツリーを editable mode で登録する構成だった。このため当時は `pip install -e .` を別途実行することが必須とは確認されていない。

## 7. パッケージ本体の import テストと CLI の動作確認

### 7.1 import テスト

```bash
cd ~/Geospatial-COSICorr3D
conda activate geoCosiCorr3D_cf

python -c "import geoCosiCorr3D; print(geoCosiCorr3D.__file__)"
```

エラーなく import できれば、Conda 環境からパッケージを参照できている。

もし `ModuleNotFoundError: No module named 'geoCosiCorr3D'` が出た場合は、環境名・Python の場所を確認した上で、必要に応じて次を **1 回**実行する。

```bash
which python
python -m pip install -e . --no-deps
python -c "import geoCosiCorr3D; print(geoCosiCorr3D.__file__)"
```

`--no-deps` は、pip によって既存の Conda 依存関係が変更されることを避けるための指定である。

### 7.2 共有ライブラリのパスを設定する

```bash
export LD_LIBRARY_PATH="$HOME/Geospatial-COSICorr3D/lib:$CONDA_PREFIX/lib:${LD_LIBRARY_PATH:-}"
```

この設定により、リポジトリに付属するライブラリと Conda 環境内のライブラリを参照可能にする。当時、IPP の共有ライブラリ（`.so.10` など）に関する対処にもこのパスの指定が関係した。

### 7.3 CLI の起動を確認する

```bash
cd ~/Geospatial-COSICorr3D
python scripts/cosicorr.py correlate -h
```

`correlate` のヘルプが表示されれば、CLI の起動確認は完了。

**当時はさらに実画像の相関計算にも成功**しており、出力には変位 2 成分と SNR を含む 3 バンドの画像が得られた（計算の実行方法は別文書 [`correlation_tutorial.md`](correlation_tutorial.md) に整理する予定）。

## 8. 次回以降、WSL を起動して使用する際の基本操作

Windows 側で Ubuntu のターミナルを起動した後、以下を実行する。

```bash
source ~/miniconda3/etc/profile.d/conda.sh
conda activate geoCosiCorr3D_cf
cd ~/Geospatial-COSICorr3D

export LD_LIBRARY_PATH="$PWD/lib:$CONDA_PREFIX/lib:${LD_LIBRARY_PATH:-}"

python scripts/cosicorr.py correlate -h
```

Conda 環境の作成、YAML の編集、`pip install -e .` は通常、起動のたびに繰り返す必要はない。

## 注意事項

- この手順は **Windows ネイティブ Python ではなく、WSL2 の Ubuntu 上**で実行する。PowerShell と Ubuntu のコマンドを混同しない。
- ソースコードは当時 `~/Geospatial-COSICorr3D` に配置した。研究室 Linux 版の `~/Geospatial-COSICorr3D-main` とはディレクトリ名が異なる。
- ローカル PC で使用した Conda 環境名は `geoCosiCorr3D_cf` であり、研究室 Linux 版の `geoCosiCorr3D` とは異なる。
- WSL2 のリソース割り当てや Windows のファイルシステムとの境界によって、実行速度・ファイルパスの扱いが変わる。
- `LD_LIBRARY_PATH` はシェルごとの設定であり、上の `export` を実行したシェルで有効になる。
- この手順書は過去に成功した構成をもとにした記録であり、すべての Windows / Ubuntu / Conda の組み合わせでの再現を保証しない。

## 参考

- [SaifAati/Geospatial-COSICorr3D（GitHub）](https://github.com/SaifAati/Geospatial-COSICorr3D)
- [Microsoft Learn：WSL のインストール](https://learn.microsoft.com/ja-jp/windows/wsl/install)

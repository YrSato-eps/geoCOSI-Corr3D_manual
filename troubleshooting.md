# geoCosiCorr3D トラブルシューティング（研究室 Linux／Windows WSL2 共通）

> **対象環境**：研究室 Linux（Rocky Linux 9.7）／Windows ローカル PC（WSL2・Ubuntu 22.04）  
> **作成日**：2026-11-04  
> **対象ソフトウェア**：オープンソース版 [Geospatial-COSICorr3D](https://github.com/SaifAati/Geospatial-COSICorr3D)  
> **対応文書**：[`install_rockylinux.md`](install_rockylinux.md)、[`install_windows_wsl.md`](install_windows_wsl.md)、[`correlation_tutorial.md`](correlation_tutorial.md)

## 目的と使い方

geoCosiCorr3D のインストール、CLI の起動、画像相関計算、結果の確認で問題が発生したときに、**症状 → 原因の切り分け → 修正 → 再確認**の順に対処する。

> **過去の作業記録と一般的な対処の区別**  
> **【実例】**は、これまでの環境構築・解析時に遭遇した問題に基づく。**【一般的な確認】**は、再現性を高めるため追加した診断方法であり、すべてのエラーを実際に再現・修正したという意味ではない。ソフトウェアの更新でコマンドや依存関係が変わる可能性がある。
>
> 以下のコマンドは、特に断りがなければ **Linux または WSL の Ubuntu ターミナル**で実行する。Windows PowerShell とは区別する。

<a id="section-1"></a>

## 1. 症状から調べる（早見表）

| 症状・エラーメッセージ | 最初に確認する箇所 | 詳細 |
| --- | --- | --- |
| `conda: command not found` | Conda の読み込み、インストール先 | [第 3 節](#section-3) |
| `ModuleNotFoundError: No module named 'geoCosiCorr3D'` | 環境名、Python の場所、editable install | [第 4 節](#section-4) |
| `LibMambaUnsatisfiableError`／依存関係の衝突 | YAML のチャネル・GDAL 固定 | [第 5 節](#section-5) |
| `error while loading shared libraries`／`.so` が見つからない | `LD_LIBRARY_PATH`、Conda のライブラリ | [第 6 節](#section-6) |
| `gdalinfo: command not found`／GDAL の起動失敗 | 環境の有効化、GDAL の依存関係 | [第 7 節](#section-7) |
| `python: can't open file 'scripts/cosicorr.py'` | 作業ディレクトリ | [第 8 節](#section-8) |
| 入力 TIFF が見つからない | パス、WSL と Windows の対応 | [第 9 節](#section-9) |
| 相関計算が失敗／出力が作成されない | CLI オプション、入力の CRS・範囲など | [第 10 節](#section-10) |
| 計算が極端に遅い／SSH 切断で停止 | 画像サイズ、メモリ、`tmux` | [第 11 節](#section-11) |

<a id="section-2"></a>

## 2. 最初に行う共通チェック

まず、**どの計算機・Conda 環境・ソースコードで実行しているか**を確認する。エラー調査中は、作業環境を頻繁に変更しない。

### 2.1 研究室 Linux（Rocky Linux）

```bash
source ~/miniconda3/etc/profile.d/conda.sh
conda activate geoCosiCorr3D
cd ~/Geospatial-COSICorr3D-main
export LD_LIBRARY_PATH="$PWD/lib:${LD_LIBRARY_PATH:-}"
```

### 2.2 ローカル PC（WSL2・Ubuntu）

```bash
source ~/miniconda3/etc/profile.d/conda.sh
conda activate geoCosiCorr3D_cf
cd ~/Geospatial-COSICorr3D
export LD_LIBRARY_PATH="$PWD/lib:$CONDA_PREFIX/lib:${LD_LIBRARY_PATH:-}"
```

### 2.3 診断用コマンド（両環境共通）

```bash
pwd
conda info --envs
which python
python --version
python -m pip --version
python -c "import geoCosiCorr3D; print(geoCosiCorr3D.__file__)"
python scripts/cosicorr.py correlate -h
```

**期待する状態**：適切な Conda 環境が有効、`which python` がその環境の Python を指し、import と CLI のヘルプ表示が例外なく完了する。

> **注意**  
> 研究室 Linux と WSL では**環境名もリポジトリのディレクトリ名も異なる**。片方の手順をそのままもう片方に貼り付けない。

<a id="section-3"></a>

## 3. `conda` または Python が使えない

**【一般的な確認】** `conda: command not found` は、Miniconda 自体がない場合と、インストール済みだが現在のシェルで読み込まれていない場合がある。

```bash
ls -ld ~/miniconda3
ls -l ~/miniconda3/etc/profile.d/conda.sh
source ~/miniconda3/etc/profile.d/conda.sh
conda --version
```

`conda.sh` が存在しなければ、インストール先を調べる。上のパスに何もない場合は各インストール手順書に戻る。**既存の `~/miniconda3` を確認せずに再インストールしない**。

`python` の参照先が想定と違うときは：

```bash
conda info --envs
which python
python -c "import sys; print(sys.executable)"
```

Conda 環境の Python が表示されているか確認する。WSL では Windows 側の Python と Ubuntu 側の Python を混同しない。

<a id="section-4"></a>

## 4. `ModuleNotFoundError`：`geoCosiCorr3D` が import できない

**【実例：研究室 Linux】** `python scripts/cosicorr.py correlate ...` の実行時に、次のエラーが発生した。

```text
ModuleNotFoundError: No module named 'geoCosiCorr3D'
```

**原因**：Conda の依存パッケージを用意しても、リポジトリ内の `geoCosiCorr3D` パッケージ本体が実行中の Python 環境に登録されていなかった。

### 4.1 まず環境とディレクトリを確認

研究室 Linux の場合：

```bash
source ~/miniconda3/etc/profile.d/conda.sh
conda activate geoCosiCorr3D
cd ~/Geospatial-COSICorr3D-main
which python
python --version
ls
```

WSL の場合は、`conda activate geoCosiCorr3D_cf` と `cd ~/Geospatial-COSICorr3D` に読み替える。

### 4.2 パッケージを登録して import を検証

**必ずリポジトリのルートディレクトリで**実行する。

```bash
python -m pip install -e . --no-deps
python -c "import geoCosiCorr3D; print(geoCosiCorr3D.__file__)"
python scripts/cosicorr.py correlate -h
```

- `-e .`：ソースコードを editable mode で登録する。
- `--no-deps`：pip に依存パッケージを勝手に追加・更新させない。
- import が成功しても、共有ライブラリの問題が残る場合は次節以降を確認する。

> **注意**  
> このインストールは通常 1 回だけ行う。SSH や WSL を再起動するたびに繰り返す必要はない。WSL 版では YAML の `pip:` に `-e .` が含まれ、環境作成時に登録される構成だったため、追加実行が常に必要なわけではない。

<a id="section-5"></a>

## 5. Conda 環境が作成できない（依存関係の衝突）

**【実例：WSL2・Ubuntu】** 元の `geoCosiCorr3D.yml` は `anaconda`、`conda-forge`、`defaults` を組み合わせ、`gdal=3.6.2` を固定していた。当時の環境では依存関係の解決が難しく、**`conda-forge` のみに統一し、GDAL の固定を解除**することで構築できた。

### 5.1 元の環境ファイルを残してコピー

```bash
cd ~/Geospatial-COSICorr3D
# コピー先が既に存在する場合は先に内容を確認する
cp geoCosiCorr3D.yml geoCosiCorr3D_cf.yml
```

### 5.2 `sed` で環境名・チャネル・GDAL の指定を変更

```bash
# 環境名を変更する
sed -i 's/^name: geoCosiCorr3D$/name: geoCosiCorr3D_cf/' geoCosiCorr3D_cf.yml

# anaconda と defaults を除外し、conda-forge は維持する
sed -i '/^[[:space:]]*-[[:space:]]*anaconda[[:space:]]*$/d' geoCosiCorr3D_cf.yml
sed -i '/^[[:space:]]*-[[:space:]]*defaults[[:space:]]*$/d' geoCosiCorr3D_cf.yml

# GDAL のバージョン固定を解除する
sed -i 's/gdal=3\.6\.2/gdal/' geoCosiCorr3D_cf.yml
```

変更の確認：

```bash
grep -nE '^(name:|channels:|[[:space:]]*-[[:space:]]*(anaconda|defaults|conda-forge|gdal))' geoCosiCorr3D_cf.yml
diff -u geoCosiCorr3D.yml geoCosiCorr3D_cf.yml || true
```

元のほかの依存パッケージと、`pip:` の `-e .` は維持する。編集済みの YAML を短い例示 YAML で**丸ごと置き換えない**。

### 5.3 Conda の全体設定を確認してから再実行

```bash
conda config --show-sources
conda config --show channels
conda env create -f geoCosiCorr3D_cf.yml
```

以前は `~/miniconda3/.condarc` によるチャネル設定の混在も問題になった。**設定の出所を確認せず `.condarc` を削除しない**。実際に問題の原因と分かった場合のみ、バックアップした上で調整する。詳しくは [`install_windows_wsl.md`](install_windows_wsl.md) 第 5 節を参照。

当時の動作確認環境は **Python 3.9.23、GDAL 3.10.2**。これは過去に成功した構成であり、以降のソフトウェア更新でこの組み合わせが自動的に選ばれる保証はない。

> **注意**  
> `conda env create` が中途半端に失敗した場合、同名の環境が残っていることがある。`conda info --envs` で確認し、状態を把握する前に環境を削除・上書きしない。

<a id="section-6"></a>

## 6. 共有ライブラリ（`.so`）が読み込めない

**【実例／一般的な確認】** `error while loading shared libraries`、`ImportError: ... .so ...` などが表示される場合、共有ライブラリの検索パス不足または依存ライブラリのバージョン不整合が考えられる。

### 6.1 ライブラリパスを設定する

研究室 Linux：

```bash
cd ~/Geospatial-COSICorr3D-main
export LD_LIBRARY_PATH="$PWD/lib:${LD_LIBRARY_PATH:-}"
python scripts/cosicorr.py correlate -h
```

ローカル WSL：

```bash
conda activate geoCosiCorr3D_cf
cd ~/Geospatial-COSICorr3D
export LD_LIBRARY_PATH="$PWD/lib:$CONDA_PREFIX/lib:${LD_LIBRARY_PATH:-}"
python scripts/cosicorr.py correlate -h
```

**【実例：WSL】** IPP 関連のライブラリ（`.so.10` など）の読み込みへの対処として、リポジトリ内の `lib` と Conda 環境の `lib` を参照する設定を行った。

### 6.2 何が不足しているか調べる

```bash
printf '%s\n' "$LD_LIBRARY_PATH"
ls -l "$CONDA_PREFIX/lib" 2>/dev/null | head
ls -l ./lib | head
```

共有ライブラリの依存関係を確認する例（対象の `.so` のファイル名に置き換える）：

```bash
ldd /path/to/problem_library.so
```

`not found` の行を確認する。`LD_LIBRARY_PATH` が通っていても、**必要な SONAME と実際にインストールされているライブラリのバージョンが違えば解決しない**。

**【参考実例：別の Linux 環境】** `gdalinfo` 実行時に `libnetcdf.so.19` が要求されたのに、環境内に `libnetcdf.so.22` しかなかったことがある。これは単なる検索パスではなく、バイナリ互換性の不整合を疑う症状である。

> **重要**  
> `.so.22` を `.so.19` に見せかけるシンボリックリンクなど、**異なる共有ライブラリのバージョンを無理に結び付けることは推奨しない**。まず Conda のパッケージ群・チャネル・バージョンの整合性を確認し、必要なら元の YAML に立ち返って環境を再構築する。

<a id="section-7"></a>

## 7. GDAL が実行できない

`gdalinfo: command not found` は GDAL の実行ファイルが見つからない場合に、`error while loading shared libraries` は依存ライブラリの読み込みに失敗した場合に表示される。

```bash
which gdalinfo
which python
conda list gdal
conda list libnetcdf
gdalinfo --version
```

`gdalinfo` が存在するのに起動できない場合、`ldd` で不足している共有ライブラリを確認する。

```bash
ldd "$(command -v gdalinfo)" | grep -E 'not found|netcdf|gdal'
```

> **注意**  
> この `ldd` コマンドは `command -v gdalinfo` がパスを返した場合にのみ使う。無条件に `pip install gdal` などを実行すると、既存の Conda 環境との整合性を崩す可能性がある。

<a id="section-8"></a>

## 8. CLI を起動できない

`python: can't open file 'scripts/cosicorr.py'` の場合、**実行するディレクトリが違う**ことが多い。

```bash
pwd
ls -l scripts/cosicorr.py
python scripts/cosicorr.py correlate -h
```

正しいソースコードのルートに移動する（第 2 節参照）。`cosicorr.py` のある場所を確認できている場合は絶対パスを指定してもよいが、ライブラリパスと Python 環境も整合させる。

`unrecognized arguments`、`invalid choice`、引数不足などの表示がある場合は、まず使用中のバージョンに対応するヘルプを読む。

```bash
python scripts/cosicorr.py -h
python scripts/cosicorr.py correlate -h
```

CLI には `--method frequency`／`spatial`、`--window_size`（4 値）、`--step`（2 値）などの引数がある。`--search_range` は `spatial` 法向けであり、`frequency` 法の設定と混同しない。<https://github.com/SaifAati/Geospatial-COSICorr3D#correlation>

<a id="section-9"></a>

## 9. 入力ファイルが見つからない

```bash
ls -lh ~/test/Pre_Turkey.tif ~/test/Post_Turkey.tif
pwd
```

- ファイル名の**大文字・小文字**を確認する。Linux では別の名前として扱われる。
- 相対パスは `pwd` に依存する。迷ったら絶対パスを指定する。
- WSL の `/mnt/c/` は通常 Windows の `C:\` に対応する。

Windows の `Downloads` から WSL へコピーする例：

```bash
# YOUR_WINDOWS_USER を実際の Windows ユーザー名に置き換える
cp /mnt/c/Users/YOUR_WINDOWS_USER/Downloads/Pre_Turkey.tif ~/test/
cp /mnt/c/Users/YOUR_WINDOWS_USER/Downloads/Post_Turkey.tif ~/test/
```

> **注意**  
> フォルダをコピーする際に `cp: -r not specified; omitting directory` が出た場合は、**ディレクトリに限り** `cp -r コピー元 コピー先` とする。通常の TIFF ファイルのコピーには `-r` は不要。

<a id="section-10"></a>

## 10. 相関計算に失敗する／出力が作成されない

### 10.1 入力画像の情報を確認

```bash
gdalinfo ~/test/Pre_Turkey.tif
gdalinfo ~/test/Post_Turkey.tif
```

次の項目を比較する。

| 項目 | 確認ポイント |
| --- | --- |
| 座標系（CRS） | 同一の座標系か。変位を m で扱うなら適切な投影座標系か |
| 解像度（GSD） | 両画像の画素寸法が一致しているか |
| 画像範囲・原点 | 十分な重複があり、画素グリッドが整合しているか |
| バンド数 | `--base_band`、`--target_band` の番号が存在するか |
| NoData・雲・影 | 有効画素や模様のある重複領域が十分か |
| ファイル容量・空き容量 | 入力が破損していないか、出力先に容量があるか |

**同じ CRS と GSD であっても、画像間の共登録が高精度とは限らない**。処理が成功しても、位置ずれによる系統誤差が残ることがある。

### 10.2 既知のコマンドを再実行する

ソースコードのルートに移動し、まず以前使った設定で実行する。

```bash
python scripts/cosicorr.py correlate \
  ~/test/Pre_Turkey.tif \
  ~/test/Post_Turkey.tif \
  --method frequency \
  --window_size 32 32 32 32 \
  --step 8 8 \
  --nb_iters 2 \
  --output_path ~/test/Turkey_eqstude.tif
```

別の試行と比較するときは `--output_path` に**新しいファイル名**を指定し、既存の結果を上書きしない。

### 10.3 出力を確認する

```bash
ls -lh ~/test/Turkey_eqstude.tif
gdalinfo ~/test/Turkey_eqstude.tif
```

通常、出力は **Band 1：東西相対変位、Band 2：南北相対変位、Band 3：SNR** である。ファイルの作成だけでなく、バンド数・数値の範囲・有効画素を確かめる。単位は入力画像の地理参照と `--pixel_based` の指定に依存するため、**すべての出力を無条件にメートルとみなさない**。

<a id="section-11"></a>

## 11. 計算が遅い／メモリ不足／SSH 切断

### 12.1 リソースの確認

```bash
nproc
free -h
df -h ~
```

出力ファイルサイズと画像サイズを確認し、大きなシーンでは必要に応じて小範囲でテストする。`--step` を小さくすると処理する相関点が増え、計算量・出力サイズが増える傾向にある。共用サーバーでは利用ルールを優先する。

WSL では、仮想環境へ割り当てられたメモリ・スワップと、Windows 側のディスク空き容量も確認する。

### 12.2 SSH 切断後も計算を続ける

研究室の運用ルールが許す場合、`tmux` を使用する。**起動した `tmux` の中で Conda の有効化と `LD_LIBRARY_PATH` の設定を行う**。

```bash
command -v tmux
tmux new -s cosicorr
```

`tmux` 内で実行：

```bash
source ~/miniconda3/etc/profile.d/conda.sh
conda activate geoCosiCorr3D
cd ~/Geospatial-COSICorr3D-main
export LD_LIBRARY_PATH="$PWD/lib:${LD_LIBRARY_PATH:-}"
```

その後に相関計算コマンドを実行する。`Ctrl+b`、続けて `d` でデタッチする。再接続時は：

```bash
tmux attach -t cosicorr
```

> **注意**  
> `tmux` のセッションは通常の SSH 切断には耐えられるが、**サーバー再起動・プロセス強制終了・メモリ不足による終了までは防げない**。研究室でジョブスケジューラが指定されている場合はその方法を使用する。

<a id="section-12"></a>

## 12. エラーの記録と、相談前にまとめる情報

原因が特定できない場合は、**エラー全文と実行環境**を一緒に記録する。画面の最後の 1 行だけでなく、最初の例外から全文を残す。

### 13.1 環境情報を保存

```bash
mkdir -p ~/test/logs
{
  date
  cat /etc/os-release
  uname -m
  pwd
  conda info --envs
  which python
  python --version
  python -m pip --version
  conda list
} > ~/test/logs/environment.txt 2>&1
```

Conda の導入や有効化ができない場合、このコマンドには失敗する行があるため、得られた情報を確認する。環境ファイルは、必要に応じて共有前に個人情報や内部パスを伏せる。

### 13.2 相関計算のログを残す

以下は **bash で実行**する例。実際のパス・設定に置き換える。

```bash
set -o pipefail
python scripts/cosicorr.py correlate \
  ~/test/Pre_Turkey.tif \
  ~/test/Post_Turkey.tif \
  --method frequency \
  --window_size 32 32 32 32 \
  --step 8 8 \
  --nb_iters 2 \
  --output_path ~/test/Turkey_eqstude_logtest.tif \
  2>&1 | tee ~/test/logs/correlation.log
```

ログとともに**終了コード**、入力画像名・バンド、CRS・GSD、環境名、geoCosiCorr3D の取得日（Git 管理ならコミット ID）、変更した YAML と比較結果を保存する。

> **注意**  
> 同名の出力 GeoTIFF を繰り返し指定しない。既存の結果を残すため、試行ごとに異なる出力名を用いる。

<a id="section-13"></a>

## 13. 解決できない場合の確認順序

1. [第 2 節](#section-2)で **Conda 環境・Python・作業ディレクトリ**が正しいか確認。
2. import エラーなら [第 4 節](#section-4)、ライブラリエラーなら [第 6 節](#section-6)を優先。
3. CLI が動くなら [第 9～10 節](#section-9)で**入力・出力 GeoTIFF**を検査。
4. それでも解決しなければ [第 12 節](#section-12)のログと、実際に使用したバージョンを添えて問題を切り分ける。

## 注意事項

- 既存の Conda 環境、YAML、`.condarc`、入力画像を**原因を調べずに削除・上書きしない**。
- システム全体への `sudo pip install` や、異なる SONAME の `.so` を無理につなぐ対処を安易に行わない。
- Linux／WSL の相違（環境名・配置パス）を確認する。**SSH 再接続時は再インストールではなく環境の再有効化が基本**。
- `correlate -h` の成功は CLI の起動確認に過ぎず、画像相関の精度や地殻変動の正しさを保証しない。
- 本書は過去の作業記録と追加の一般的な切り分け方法を整理したもので、すべての環境・リポジトリの将来の更新に対する動作保証ではない。

## 参考

- [SaifAati/Geospatial-COSICorr3D（GitHub）](https://github.com/SaifAati/Geospatial-COSICorr3D)
- [公式 README：`correlate` の使用法](https://github.com/SaifAati/Geospatial-COSICorr3D#correlation)
- [公式の Conda 環境ファイル `geoCosiCorr3D.yml`](https://github.com/SaifAati/Geospatial-COSICorr3D/blob/main/geoCosiCorr3D.yml)
- [公式のインストールスクリプト `install_cosicorr.sh`](https://github.com/SaifAati/Geospatial-COSICorr3D/blob/main/install_cosicorr.sh)

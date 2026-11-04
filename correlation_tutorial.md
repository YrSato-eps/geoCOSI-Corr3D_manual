# geoCosiCorr3D 画像相関解析手順書（Correlation Tutorial）

> **対象環境**：研究室 Linux（Rocky Linux 9.7）／Windows ローカル PC（WSL2・Ubuntu 22.04）  
> **作成日**：2026-11-04  
> **対象ソフトウェア**：オープンソース版 [Geospatial-COSICorr3D](https://github.com/SaifAati/Geospatial-COSICorr3D)  
> **対応文書**：[`install_rockylinux.md`](install_rockylinux.md)、[`install_windows_wsl.md`](install_windows_wsl.md)、[`troubleshooting.md`](troubleshooting.md)（予定）

## 目的

オープンソース版 geoCosiCorr3D のコマンドラインインターフェース（CLI）を使い、**2 枚の光学画像間のピクセルオフセット（相対変位）**を推定する。

本書では、これまで使用した `Pre_Turkey.tif`（基準画像）と `Post_Turkey.tif`（比較画像）を例に、**実行環境の準備 → 入力画像の確認 → 相関計算 → 結果の確認 → QGIS による可視化 → 品質評価**まで扱う。

> **対象範囲**  
> 本書は、原則として **オルソ補正済み・同一座標系／同一解像度で、互いに重なる画像**が用意されている段階から始める。オルソ補正、精密位置合わせ、モザイク作成、雲・影のマスキングなどの前処理を自動的に代行するものではない。
>
> **過去の操作と補足**  
> 第 4 節に掲載する `Pre_Turkey.tif` / `Post_Turkey.tif` のコマンドは、以前使用した設定を踏襲している。入力検査、QGIS 操作、品質管理、長時間計算の補足は、再現可能な手順書として追加した項目である。すべての操作を今回新たに実行・検証したわけではない。

## 1. 解析の流れと前提条件

処理の流れ：

1. 解析に使用する2枚の画像を用意する。
2. 画像の座標系（CRS）、ピクセルサイズ、重なり、バンドを確認する。
3. Linux／WSL で Conda 環境を有効化する。
4. `cosicorr.py correlate` で2枚の画像を相関処理する。
5. 出力された GeoTIFF（東西変位・南北変位・SNR）を調べる。
6. QGIS で変位分布と低信頼領域を可視化する。
7. 安定領域や地震前同士のペアと比較して、変位とノイズを評価する。

### 入力画像に必要な条件

| 項目 | 確認内容 |
| --- | --- |
| 入力形式 | 本手順では GeoTIFF（`.tif`）を想定 |
| 座標系 | **両画像で同じ CRS**。変位をメートルで読みたい場合、UTM などの**メートル単位の投影座標系**が望ましい |
| 地上画素寸法 | 両画像で同じ GSD（pixel size） |
| ピクセルグリッド | 原点と解像度が整合していること。位置のずれがあると処理失敗や系統誤差につながる |
| 重複領域 | 十分な共通領域があること |
| 画像品質 | 雲・影・水域・植生変化などが少なく、両時期の模様に対応があること |
| バンド | 比較するバンドを選ぶ（指定しなければ Band 1） |

**重要**：同じ CRS／GSD であることは、シーン間の精密位置合わせが完了したことを意味しない。数分の1ピクセルの位置ずれでも、対象とする変位の大きさによっては重大な誤差となる。

## 2. 解析環境を起動する

以下は **Linux のシェル（WSL の場合は Ubuntu ターミナル）**で実行する。環境によってソースコードの場所と Conda 環境名が異なるため、該当する一方だけを使用する。

### 2.1 研究室 Linux（Rocky Linux 9.7）

研究室計算機に接続した後：

```bash
# ローカル PC 側で SSH 接続（ログイン済みなら不要）
ssh ユーザー名@ホスト名

# 接続先の Linux で実行
source ~/miniconda3/etc/profile.d/conda.sh
conda activate geoCosiCorr3D
cd ~/Geospatial-COSICorr3D-main
export LD_LIBRARY_PATH="$PWD/lib:${LD_LIBRARY_PATH:-}"
```

### 2.2 ローカル Windows PC（WSL2・Ubuntu）

```bash
# WSL の Ubuntu ターミナルで実行
source ~/miniconda3/etc/profile.d/conda.sh
conda activate geoCosiCorr3D_cf
cd ~/Geospatial-COSICorr3D
export LD_LIBRARY_PATH="$PWD/lib:$CONDA_PREFIX/lib:${LD_LIBRARY_PATH:-}"
```

### 2.3 Python、import、CLI の動作確認

```bash
which python
python --version
python -c "import geoCosiCorr3D; print(geoCosiCorr3D.__file__)"
python scripts/cosicorr.py correlate -h
```

`correlate` のヘルプがエラーなく表示されれば、CLI を起動できている。

> **注意**  
> `LD_LIBRARY_PATH` の指定は**今のシェル内だけ**で有効である。SSH や Ubuntu ターミナルを開き直した場合は設定し直す。詳しくは各インストール手順書を参照する。

## 3. 入力画像を配置し、情報を確認する

### 3.1 画像を配置する

本書では、作業ディレクトリをホームディレクトリの `~/test` とする。

```bash
mkdir -p ~/test
ls -lh ~/test/Pre_Turkey.tif ~/test/Post_Turkey.tif
```

**あらかじめ**2枚の画像を `~/test` に配置しておく。`ls` にファイル名とサイズが表示されればよい。ファイルがなければ、パスと名前を見直す。

Windows のファイルを WSL にコピーする場合、Windows の `C:\` は WSL では通常 `/mnt/c/` に対応する。例：

```bash
# パスの YOUR_WINDOWS_USER は実際の Windows ユーザー名に置き換える
cp /mnt/c/Users/YOUR_WINDOWS_USER/Downloads/Pre_Turkey.tif ~/test/
cp /mnt/c/Users/YOUR_WINDOWS_USER/Downloads/Post_Turkey.tif ~/test/
```

上記のコピーは**ファイルが Windows 側に存在する場合のみ**行う。すでに `~/test` に配置済みなら不要。

### 3.2 GeoTIFF のメタデータを確認する

GDAL のコマンドが使用できる場合：

```bash
command -v gdalinfo

gdalinfo ~/test/Pre_Turkey.tif
gdalinfo ~/test/Post_Turkey.tif
```

主に次の情報を確認する。

- `Size is`：画像の列数・行数
- `Coordinate System`：座標系／EPSG コード
- `Pixel Size`：画素寸法と符号
- `Origin`、`Corner Coordinates`：画像の範囲とグリッド位置
- `Band 1` など：バンド数、データ型、NoData の設定

**同一座標系、同一解像度、十分な重複**があることを確認する。厳密なグリッド位置合わせの確認は、QGIS での重ね合わせや GDAL のジオトランスフォームを用いて別途行う。

> **補足**  
> `gdalinfo: command not found` と表示された場合は、まず正しい Conda 環境を有効化しているか確認する。GDAL を無条件に追加インストールせず、既存の環境・依存関係を確認する。

## 4. 画像相関計算を実行する（実際に使用した設定）

### 4.1 実行コマンド

第 2 節で geoCosiCorr3D のディレクトリに移動していることを確認してから、次を実行する。

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

これは以前に用いた**周波数領域相関（frequency correlator）**の実行例である。入力順は `base_image` → `target_image`、すなわち **Pre → Post** とする。

成功すれば、指定したパスに相関結果の GeoTIFF が作られる。**同名ファイルが既にある場合は上書きの可能性がある**ため、試行条件を変えるときは出力名も変える。

### 4.2 コマンドの意味

| 引数 | この例の指定 | 意味 |
| --- | --- | --- |
| `correlate` | ― | 2画像間の相関処理を実行 |
| `base_image` | `Pre_Turkey.tif` | 基準画像（通常、地震前など） |
| `target_image` | `Post_Turkey.tif` | 比較画像（通常、地震後など） |
| `--method` | `frequency` | 周波数領域相関を使用。`spatial` も選択可 |
| `--window_size` | `32 32 32 32` | 初期相関窓の X・Y、最終相関窓の X・Y（画素数） |
| `--step` | `8 8` | 相関窓を動かす X・Y 方向の間隔（入力画像の画素単位） |
| `--nb_iters` | `2` | 周波数領域相関における反復回数 |
| `--output_path` | `~/test/Turkey_eqstude.tif` | 出力ファイルのパス |

**設定していない項目**には、インストールされたコード側のデフォルトが使われる。2026-11-04 に確認した公開 CLI ソースでは、`--mask_th` の初期値は `0.95`、`--base_band` と `--target_band` の初期値は各 `1`。異なるコミットでは値が変わる可能性があるため、実環境の `correlate -h` でも確認する。

### 4.3 主要パラメータの考え方

**`--window_size`（例：`64 64 32 32`）**

- 前半 `64 64`：初期相関窓（X, Y）。比較的大きなずれの概略を求める段階。
- 後半 `32 32`：最終相関窓（X, Y）。サブピクセル変位を求める段階。
- 大きな窓は一般に相関を安定化させやすいが、断層近傍などの局所的な変位勾配を平均化しやすい。
- 小さな窓は局所構造を追いやすい反面、模様の少ない領域やノイズに敏感になる。

**`--step`（例：`8 8`）**

- 出力変位を推定する地点の間隔。入力画像の 8 ピクセルごとに相関窓を配置する設定。
- Step を小さくすると出力点は密になるが、窓同士は強く重なるため、**独立した空間分解能がそのまま向上するわけではない**。
- 入力画素がメートル単位の投影座標系で表されている場合、出力の格子間隔は概ね **Step × 入力画素サイズ**となる。

**`--nb_iters` と `--mask_th`**

- `--nb_iters` は相関計算に使う適応的な周波数マスクなどの反復処理回数。
- `--mask_th` は周波数領域相関のマスク設定に関する閾値であり、**出力 SNR の合否閾値ではない**。
- パラメータを変更したら出力を別名で保存し、ノイズレベルと断層近傍の信号の再現性を比較する。

**その他の主な引数**

```bash
python scripts/cosicorr.py correlate -h
```

| オプション | 用途・注意点 |
| --- | --- |
| `--base_band 1 --target_band 1` | 多バンド画像から使用バンドを選択（1始まり） |
| `--grid` | 地図座標の格子に出力を揃えるオプション。画像原点等に制約があるため、まず既定値で実行し、必要に応じて試す |
| `--show` | 相関結果の図を作成。SSH サーバーなどでは描画環境に注意 |
| `--pixel_based` | 地図座標によらないピクセルベースの相関。出力成分の単位・解釈に注意 |
| `--search_range X Y` | `spatial` 法用。`frequency` 法では使わない |

> **注意**  
> 空間相関（`--method spatial`）では探索範囲など別の設定が必要になる。この手順書の実行例は `frequency` 法について検証・整理したもので、`spatial` 法を同じ設定で動作確認したことを意味しない。

## 5. 実行結果を確認する

### 5.1 出力ファイルを確認する

```bash
ls -lh ~/test/Turkey_eqstude.tif
```

可能であれば、出力 TIFF のバンド数・座標系・解像度を確認する。

```bash
gdalinfo ~/test/Turkey_eqstude.tif
```

通常、出力は以下の **3 バンド**からなる（公開ソースで確認）。

| バンド | 内容 | 単位・解釈 |
| --- | --- | --- |
| **Band 1** | East / West（東西方向の相対変位） | 地図ベースの相関では投影座標系の単位（通常 m）、東向きを正 |
| **Band 2** | North / South（南北方向の相対変位） | 地図ベースの相関では投影座標系の単位（通常 m）、北向きを正 |
| **Band 3** | SNR | 相関結果の品質評価指標。変位ではない |

**重要な単位の注意：**

- 入力が UTM など**メートル単位**の投影座標系で、地図ベース相関が行われている場合、Band 1・2 の変位は通常メートル単位。
- `--pixel_based` を指定した場合、あるいは有効な地理参照情報がなくピクセルベースで処理された場合は、Band 1・2 は**画素単位**になり得る。
- 経緯度座標系（度単位）を使った場合、出力値を自動的に「メートル」とみなしてはいけない。必要に応じて入力をメートル単位の適切な投影座標系へ再投影する。

### 5.2 実行が成功したか判定する

- Python の例外・共有ライブラリエラーが出ていない。
- GeoTIFF が出力され、`gdalinfo` で読み取れる。
- Band 1〜3 を確認できる。
- 変位の値が有限であり、対象領域と画像の重複範囲が妥当。

> **注意**  
> **ファイルが出力されたことは、地表変位を正しく測定できたことの証明ではない。** マッチングの破綻や画像間の位置合わせ誤差が、大きな「見かけの変位」を作ることがある。

## 6. QGIS で結果を表示する

### 6.1 GeoTIFF を読み込む

1. QGIS を起動する。
2. **［レイヤ］→［レイヤの追加］→［ラスタレイヤを追加］**で `Turkey_eqstude.tif` を開く（ファイルをドラッグ＆ドロップしてもよい）。
3. レイヤを右クリックし、**［プロパティ］→［シンボロジ］**を開く。
4. レンダリング方法を **「単バンド疑似カラー」**に変更する。
5. 表示したいバンドを指定して、色の範囲・カラーマップを調整する。

WSL 側にファイルがある場合、Windows のエクスプローラーから `\\wsl$\Ubuntu-22.04\home\Linuxユーザー名\test` などで開けることがある（ディストリビューション名・ユーザー名は環境に合わせる）。別案として WSL から Windows 側へ結果をコピーしてから QGIS で開いてもよい。

### 6.2 バンドごとの表示方法

- **Band 1（東西変位）**：0 を中心に、負（西向き）と正（東向き）が区別できる発散型カラーマップを用いる。
- **Band 2（南北変位）**：同じく 0 を中心に、南向きと北向きを区別する。
- **Band 3（SNR）**：低 SNR 領域の分布を調べる。SNR が低い画素を機械的にすべて除外できるとは限らない。

複数の結果を比較するときは、変位図の**表示範囲（最小値・最大値）を同一にする**と解釈しやすい。外れ値に引きずられないよう、分布と NoData の扱いも確認する。

### 6.3 印刷レイアウトで出力する（任意）

地図を画像として保存したい場合は、QGIS の **［プロジェクト］→［新規印刷レイアウト］**からレイアウトを作成し、地図、凡例、縮尺、座標目盛などを配置して PNG としてエクスポートする。

## 7. よくあるエラーと初期確認

| 症状 | まず確認すること |
| --- | --- |
| `ModuleNotFoundError: No module named 'geoCosiCorr3D'` | `conda activate` の環境名、`which python`、パッケージの editable インストール |
| `.so` など共有ライブラリの読み込みエラー | `LD_LIBRARY_PATH`、`$CONDA_PREFIX/lib`、インストール手順書の設定 |
| 画像が見つからない | `ls` でパスと大文字／小文字を確認 |
| 座標系や解像度が違う | `gdalinfo` で CRS・GSD を確認し、必要な前処理を行う |
| 重複領域不足／グリッド不整合 | 入力画像の範囲・原点・解像度を確認する |
| ノイズが大きい／不自然な変位 | 元画像の位置ずれ・雲・影・地形・バンド・相関窓と Step を点検 |
| SNR が低い | 模様の乏しい領域、地表状態の変化、初期・最終相関窓設定を点検 |
| 計算が長い／メモリを使う | 画像サイズ、相関窓・Step、使用 CPU・メモリ、共用計算機の利用ルールを確認 |

詳細な対処方法は今後の [`troubleshooting.md`](troubleshooting.md) にまとめる。

## 8. SSH 切断後も計算を継続したい場合（任意）

研究室サーバーで長い計算を実行するときは、研究室の運用ルールに従い、利用可能なら `tmux` やジョブスケジューラを使用する。`tmux` を使う場合の例：

```bash
# tmux が利用できるか確認
command -v tmux

# 利用できる場合に開始
tmux new -s cosicorr
```

新しいシェル内で **第 2.1 節の Conda 環境有効化とライブラリパス設定**を実行してから、第 4 節の相関コマンドを実行する。`Ctrl+b` の後に `d` でセッションから離れる。再接続後は：

```bash
tmux attach -t cosicorr
```

> **注意**  
> `tmux` が入っていない、あるいはジョブスケジューラの使用が必須の場合は、研究室の指定方法に従う。ログアウト後の継続を確認せずに長時間計算を始めない。

## 9. 解析条件の記録と再現性

解析結果とともに、少なくとも以下を記録する。

- 使用した **Pre / Post 画像名**、取得日時、バンド
- オルソ補正・共登録・モザイクなどの前処理内容
- 入力画像の **CRS／EPSG・GSD・撮像幾何**
- `--method`、`--window_size`、`--step`、`--nb_iters`、`--mask_th` などの設定
- 出力ファイル名、実行日、エラー・警告、安定領域におけるノイズ量
- geoCosiCorr3D のバージョン／Git コミット（Git クローンを使った場合）

Git 管理されたソースコードであれば、以下でコミットを確認できる。

```bash
git rev-parse HEAD
```

ZIP 配布物の場合は Git の履歴が含まれないことがある。取得日や配布 ZIP の情報を記録する。

## 注意事項

- 本書のコマンドは、**Linux / WSL のターミナル**向けである。Windows のコマンドプロンプトや PowerShell にそのまま入力するものではない。
- 入力 TIFF がオルソ補正済みでも、画像間の高精度な位置合わせは別途検証する。
- `--output_path` に既存ファイルを指定した場合の上書きに注意する。
- 画像の座標系・地上画素寸法・処理モードによって、出力値の単位・符号・地理参照の可否を確認する。
- `--show` など画像表示を伴うオプションは、ディスプレイのない SSH サーバーでは制限される場合がある。
- コマンドや既定値はリポジトリの更新に伴い変わる可能性がある。実際に使用するソースコードの `correlate -h` を優先する。

## 参考

- [SaifAati/Geospatial-COSICorr3D（GitHub）](https://github.com/SaifAati/Geospatial-COSICorr3D)
- [公式 README：CLI `correlate` の引数一覧](https://github.com/SaifAati/Geospatial-COSICorr3D#correlation)
- [公式ソース：`scripts/cosicorr.py`（CLI 引数定義）](https://github.com/SaifAati/Geospatial-COSICorr3D/blob/main/scripts/cosicorr.py)
- [公式ソース：`geoImageCorrelation/correlate.py`（出力バンド・スケーリング）](https://github.com/SaifAati/Geospatial-COSICorr3D/blob/main/geoCosiCorr3D/geoImageCorrelation/correlate.py)
- [Caltech：COSI-Corr User's Guide（相関窓・Step 等の考え方）](https://tecto.gps.caltech.edu/slip_history/spot_coseis/pdf_files/CosiCorr-Guide2017.pdf)

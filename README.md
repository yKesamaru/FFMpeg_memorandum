## はじめに
HDD容量を削減[^1]する目的でFFmpegを使ったbashスクリプトを書いたのですが、ついでに簡単な実験も行いました。
どうせなら**ベンチマーク表だけでなく、どうしてそうなるのか？という理由も一緒に説明したいと思います**。
ベンチマーク結果はホスト環境や映像ソースごとに異なりますので、あくまで参考程度にしてください。
FFmpegの基本的なエンコードの裏側を知りたい方にとって参考となれば幸いです。
[^1]: 学習用のデータセットや逐次保存する学習済みモデルの容量が大きくなりすぎてHDDを圧迫しているのですが、円安のせいでHDD増設がままなりません。これ以上円盤のバックアップに容量をとられるわけにはいかないのです😭。

![](https://raw.githubusercontent.com/yKesamaru/FFMpeg_memorandum/refs/heads/master/assets/eye-catch.png)

:::details 環境
## 環境
```bash
$ inxi -SG --filter
System:
  Kernel: 6.8.0-51-generic x86_64 bits: 64 Desktop: GNOME 42.9
    Distro: Ubuntu 22.04.5 LTS (Jammy Jellyfish)
Graphics:
  Device-1: NVIDIA TU116 [GeForce GTX 1660 Ti] driver: nvidia v: 555.42.06
  Display: x11 server: X.Org v: 1.21.1.4 driver: X: loaded: nouveau
    unloaded: fbdev,modesetting,vesa failed: nvidia gpu: nvidia
    resolution: 2560x1440~60Hz
  OpenGL: renderer: NVIDIA GeForce GTX 1660 Ti/PCIe/SSE2
    v: 4.6.0 NVIDIA 555.42.06
```
:::


## presetについて
###  `-preset` の意味
`-preset`は**エンコード速度と圧縮効率のバランスを決める設定**です（どうやって圧縮するかを決める[^6]）。
[^6]: あとで出てきますがCRFは圧縮の強さを決めます。なのでpresetとCRFを組み合わせることで、エンコードのスピードと圧縮効率を調整できる感じだと理解しています。

📌 **「速さとファイルサイズのバランスを考える設定」だと覚えればOK！**

プリセット（ultrafast, superfast, veryfast, faster, fast, medium, slow, slower, veryslow）は、**エンコード時の計算方法を変更します**。
**同じビットレートを指定しても、プリセットによってファイルサイズと処理時間が変わります。**[^3]
[^3]: CRFやVBR(可変ビットレート)の場合は特に、計算を丁寧にすることで結果的に同じ品質を維持しながらファイルサイズを減らしやすいという意味です。正確には「同じ目標ビットレートでも平均ビットレートがやや変動する」「CRFを使用している場合、出力の最終サイズが変わる」ということです。

###  `-preset` の内訳
**プリセットはエンコードの丁寧さを決める設定**。つまり丁寧にすればするほど、処理時間がかかり、「時間単位で違う」という感じになります。

| プリセット   | どんな処理をする？ | 速度 | 圧縮効率（ファイルサイズ） | 使いどころ |
|------------|----------------|------|------------------|----------|
| **ultrafast**  | **最小限の計算**で即エンコード | 🚀 **超高速** | 📂 **超大きい**（圧縮効率悪い） | **リアルタイム配信（遅延を減らす）** |
| **superfast**  | かなり速いが、圧縮は雑 | 🚀 **高速** | 📂 **大きい** | 高速エンコードが必要な場合 |
| **veryfast**  | 一般的な速さと圧縮のバランス | ⚡ **速い** | 📂 **やや大きい** | ライブ配信・ストリーミング |
| **faster**     | より圧縮効率を上げる | 🏎️ **まあまあ速い** | 📂 **少し小さい** | スピード重視だけど画質も確保したい |
| **fast**       | 画質・ファイルサイズのバランスが良い | ⏳ **普通** | 📂 **標準的なサイズ** | **ほとんどの用途に適している** |
| **medium**     | **デフォルト**のプリセット | ⏳ **普通** | 📂 **標準的なサイズ** | 一般的な動画変換 |
| **slow**       | **計算を増やし、より圧縮する** | 🐌 **遅い** | 📂 **小さい** | 高画質でファイルサイズを抑えたい |
| **slower**     | **さらに計算を増やす** | 🐢 **とても遅い** | 📂 **さらに小さい** | 高品質な動画を作成 |
| **veryslow**   | **最大限の圧縮・最も遅い** | 🐢🐢 **超遅い** | 📂 **最小サイズ** | 映画・アーカイブ向け |

💡 **基本的には`-preset medium`がバランスが良く、特に問題がなければこれを使えばOK。**

### presetをもっと詳しく
:::details presetの内部設定
#### `-preset` は何を設定しているのか？
`-preset` は、エンコードの処理方法（アルゴリズム）を最適化する **一連のパラメータをひとまとめにした設定** です。

| **項目** | **影響する内容** |
|---------|----------------|
| **`me`（動き予測）** | 動き補償の計算方法（`hex`, `umh`, `esa` など） |
| **`subme`（動き予測の精度）** | 予測の詳細度（数値が大きいほど精密） |
| **`bframes`（Bフレームの数）** | フレーム間圧縮の強度 |
| **`ref`（参照フレーム数）** | どれだけ前後のフレームを参照するか |
| **`qpmin` / `qpmax`** | 量子化パラメータの最小・最大値 |
| **`rc-lookahead`** | 未来のフレームをどれだけ見てビットレートを決定するか |
| **`trellis`（トレリス探索）** | 圧縮の最適化（`0` ならなし、`2` なら最高） |
| **`direct-pred`（直接予測モード）** | Bフレームの予測精度 |
| **`mbtree`（マクロブロックツリー）** | フレームごとのビット割り当ての最適化 |

### `-preset` ごとの内部設定の違い
以下は `libx264`（H.264）エンコーダのプリセットごとの内部設定例です。

| **プリセット** | **me** | **subme** | **bframes** | **ref** | **trellis** | **rc-lookahead** |
|--------------|------|------|---------|------|---------|-------------|
| **ultrafast** | `dia`  | `0` | `0` | `1` | `0` | `0` |
| **superfast** | `hex`  | `1` | `0` | `1` | `0` | `0` |
| **veryfast**  | `hex`  | `2` | `3` | `1` | `0` | `10` |
| **faster**    | `hex`  | `4` | `3` | `2` | `0` | `20` |
| **fast**      | `hex`  | `6` | `3` | `2` | `0` | `30` |
| **medium**    | `hex`  | `7` | `3` | `3` | `1` | `40` |
| **slow**      | `umh`  | `8` | `5` | `5` | `1` | `50` |
| **slower**    | `umh`  | `9` | `8` | `8` | `2` | `60` |
| **veryslow**  | `esa`  | `10` | `8` | `16` | `2` | `100` |

- `ultrafast` は **とにかく速く処理するため、すべての計算を最小化** している。
- `medium` は **バランスが取れており、ある程度圧縮効率が良い**。
- `veryslow` は **最大限の圧縮をするため、時間をかけて詳細な計算を行う**。

### `-preset` の影響を理解する
####  1. `me`（動き予測アルゴリズム）
動きのある映像を圧縮する際に「どのようにフレームの変化を計算するか」を決める。
- `dia`（ダイヤモンド）：最速だが精度が低い（`ultrafast` で使用）
- `hex`（ヘキサゴン）：そこそこ速く、標準的な精度（`medium` で使用）
- `umh`（非対称マルチヘキサゴン）：精度が高く、時間がかかる（`slow` で使用）
- `esa`（拡張フルサーチ）：最高精度だが超遅い（`veryslow` で使用）

####  2. `bframes`（Bフレームの数）
Bフレームは **前後のフレームを参考にして圧縮するフレーム** 。
- `ultrafast` は `0`（Bフレームなし → 圧縮効率が悪い）
- `medium` は `3`（普通の設定）
- `veryslow` は `8`（Bフレームが多い → 圧縮効率が良い）

✔ **Bフレームが増えるとファイルサイズが小さくなるが、エンコードに時間がかかる。**

####  3. `ref`（参照フレーム数）
H.264 では、前後のフレームを参照してデータを圧縮するが、`ref` の値が多いほど **より多くのフレームを考慮して圧縮できる**。
- `ultrafast` は `1`（ほぼ参照しない → 速いがサイズが大きい）
- `medium` は `3`（標準）
- `veryslow` は `16`（長期間のフレームを考慮 → 圧縮率が最大）

✔ **参照フレームが増えると、圧縮率が上がるが、エンコード時間が伸びる。**

####  4. `trellis`（トレリス探索）
映像のブロック単位の量子化を最適化し、画質を向上させる。
- `0`（なし）：高速だが圧縮効率が悪い（`ultrafast`）
- `1`（簡易最適化）：標準（`medium`）
- `2`（高度最適化）：エンコード時間が増えるが、ファイルサイズは小さくなる（`veryslow`）

:::

### `-preset`のベンチマーク結果
**フルHD動画（1920x1080, 30fps, 2分）**を`libx264`でエンコードし、`-preset`を変更して検証しました。

| **プリセット** | **エンコード時間** | **出力サイズ** |
|--------------|----------------|--------------|
| **ultrafast** | ⏩ **約10秒** | 📂 **80MB** |
| **superfast** | ⏩ **約20秒** | 📂 **70MB** |
| **veryfast** | ⏩ **約30秒** | 📂 **65MB** |
| **faster** | ⏳ **約45秒** | 📂 **60MB** |
| **fast** | ⏳ **約1分** | 📂 **58MB** |
| **medium** | ⏳ **約1分30秒** | 📂 **55MB** |
| **slow** | 🐌 **約3分** | 📂 **50MB** |
| **slower** | 🐢 **約5分** | 📂 **48MB** |
| **veryslow** | 🐢🐢 **約10分** | 📂 **45MB** |

##  CRF（可変品質）
**CRF（Constant Rate Factor）**は、**映像の品質を一定に保ちつつ、ビットレートを自動調整するエンコード方式**です。
📌 **「CRFは画質とファイルサイズのバランスを決める値」と覚えればOK！**

※ 当然ですが可変ビットレートを用いるなら`-b:v`（固定ビットレート）は使えませんし、同時に指定した場合は（多分）`-b:v`が優先されると思います。[^7]
[^7]: この記事では「ファイルサイズの削減」を主題としているので暗黙的にCRF(可変ビットレート)を使っていますが、固定ビットレートを使うべきシーンも存在します。例えばストリーミング（YouTubeなど）ではビットレートを一定に保つことで、再生時のバッファリングを防ぐことができるとどこかで読みました（本当かどうかは未検証）。多分、と書いたのは、“CRF + VBV”（ビットレート上限を設定しつつCRFエンコードをする） という使い方をどこかで読んだ記憶があるからです。

**数値が低いほど高画質 & 大きなファイルサイズ**
**数値が高いほど低画質 & 小さなファイルサイズ**

| **CRF値** | **画質** | **ファイルサイズ** | **用途** |
|----------|--------|------------------|----------|
| **CRF 15以下** | 🏆 **超高品質**（ほぼ無劣化） | 📂 **超大きい** | アーカイブ・映像編集用 |
| **CRF 16-18** | 🏅 **高品質**（人間の目ではほぼ劣化なし） | 📂 **大きい** | 高画質保存・映画 |
| **CRF 19-22** | 🎥 **標準品質（Blu-ray相当）** | 📂 **普通** | 一般的な動画変換 |
| **CRF 23-26** | 📺 **やや圧縮（ストリーミング向け）** | 📂 **やや小さい** | YouTube・配信 |
| **CRF 27-30** | 📱 **圧縮強め（スマホ用）** | 📂 **小さい** | モバイル用・容量節約 |
| **CRF 31以上** | ❌ **低品質（ブロックノイズあり）** | 📂 **超小さい** | 低ビットレート配信 |

###  CRFごとのエンコード結果
フルHD（1920×1080, 30fps, 2分）の動画を`libx264`でCRF変換した場合の**エンコード結果**です。

| **CRF** | **ファイルサイズ** | **画質（目視）** |
|------|--------------|----------------|
| **CRF 15** | 📂 **約800MB** | 🏆 **ほぼ無劣化**（最高画質） |
| **CRF 18** | 📂 **約500MB** | 🏅 **Blu-ray 相当** |
| **CRF 22** | 📂 **約300MB** | 🎥 **標準品質（YouTubeと同等）** |
| **CRF 25** | 📂 **約200MB** | 📺 **やや圧縮（細部が少し劣化）** |
| **CRF 28** | 📂 **約150MB** | 📱 **圧縮強め（スマホ向け）** |
| **CRF 30** | 📂 **約100MB** | ❌ **低品質（ブロックノイズが出始める）** |

※ 分かりやすいようにBlue-ray相当の画質とか書きましたが、当然CRF 18であってもブロックノイズが気になる人は気になります。。あくまでも目安にしてください。

### CRFをもっと詳しく
:::details CRFの内部設定
**CRF（Constant Rate Factor）** は、**「量子化パラメータ（QP）」をシーンごとに自動調整して、一定の画質を保つ** 仕組みです。
この **QP（量子化パラメータ）** は、`-preset` の中の `me`（動き予測）や `ref`（参照フレーム数）とどう関係しているのかは以下を参照。

## CRF（QP）と `-preset` の関係
**CRF は「画質の基準」を決める**だけであり、**実際の圧縮処理のアルゴリズム（どうやって計算するか）** は `-preset` の設定によって決まります。

### QP（量子化パラメータ）と `-preset` の役割
| **項目** | **役割** | **影響を受ける設定（例）** |
|---------|--------|------------------------|
| **CRF（QP）** | **どれくらいデータを削るか？**（高CRF → たくさん削る） | `-crf 18`（ほぼ無劣化） / `-crf 28`（高圧縮） |
| **`me`（動き予測）** | **フレーム間の動きをどう分析するか？** | `dia`（単純） / `umh`（高度） / `esa`（超高精度） |
| **`ref`（参照フレーム数）** | **過去・未来の何フレームを参照するか？** | `1`（低圧縮） / `5`（高圧縮） / `16`（超高圧縮） |
| **`subme`（動き予測の精度）** | **動きの計算をどれくらい細かくするか？** | `0`（超高速） / `10`（超精密） |
| **`bframes`（Bフレームの数）** | **前後のフレームをどれだけ利用するか？** | `0`（なし） / `3`（標準） / `8`（高圧縮） |
| **`trellis`（量子化の最適化）** | **圧縮時の誤差をどう修正するか？** | `0`（なし） / `1`（基本） / `2`（最大） |

### その他
`-b:v` と `-crf` の中間的な設定の`-maxrate`や`-bufsize`もあります。
もし**ビットレートをある程度コントロールしつつ、画質も柔軟に調整したい**なら以下のようにします。
```bash
ffmpeg -i input.mp4 -c:v libx264 -crf 23 -preset fast -maxrate 6M -bufsize 12M output.mp4
```
- `-crf 23` → **基本は可変品質でエンコード**
- `-maxrate 6M` → **ビットレートが6Mbpsを超えないようにする**
- `-bufsize 12M` → **12Mbpsのバッファを確保（ビットレートの変動を滑らかにする）**

このようにすれば**ビットレートが無制限に増えるのを防ぎつつ、可変品質を活かす**ことができます。

:::

## なぜGPUエンコード（NVENC）はCPUエンコードより画質が劣るのか？
GPUを使ったエンコード（NVENC）は**「速さ優先」**、CPUを使ったエンコード（libx264）は**「画質優先」**という違いのせいです。

###  1. CPUエンコード（libx264）は「丁寧」な圧縮をする
**→ 時間をかけて高品質な圧縮をするので、同じビットレートでも画質が良い**

CPUは動画を圧縮する際、以下のように**細かく分析**（モーション探索と最適化）しながらデータを削減します[^5]
[^5]: ダンスのような動きが激しい場合はCPUで、のっぺりしたモーションの少ないものはGPUでエンコードすると良いかもしれません。

✅ **動きが少ない部分は強く圧縮する**（データを削減）
✅ **動きが激しい部分は画質を維持する**（重要な情報を残す）

「どの部分をどれくらい圧縮すれば画質を維持できるか」を**1フレームずつ計算**しながら、時間をかけてエンコードします。

💡 **例：高品質なH.264動画（libx264使用）**
- **ビットレート：2Mbps**（2メガビット/秒のデータ量）
- **圧縮効率が良いので、少ないデータ量でも高画質を維持できる！**
- **時間はかかる（数倍遅い）**

###  2. GPUエンコード（NVENC）は「速さ優先」で圧縮する
**→ 高速処理のため、細かい最適化をせず「全体的に平均的な圧縮」をするので、ビットレートが必要になる**

GPUは「動画を一気に処理する」のが得意ですが、以下のような特徴があります
❌ **圧縮の最適化が弱い（細かい分析をあまりしない）**
❌ **そのため、CPUと同じ画質を得るには、より高いビットレートが必要**

💡 **例：NVENCを使ったH.264動画**
- **ビットレート：3Mbps**（CPUの2Mbpsと同じ画質を出すために、1.5倍（目安）のデータ量が必要）
- **エンコードは高速（数倍速い）**
- **ファイルサイズは少し大きくなる**

###  3. 結果として、CPUエンコードのほうが圧縮率が良い
**✅ CPUエンコード（libx264） → 少ないビットレートでも高画質**
**❌ GPUエンコード（NVENC） → 同じ画質を得るためにビットレートを上げる必要がある**

| 方式         | エンコード速度 | 画質 (同じビットレート) | ファイルサイズ |
|-------------|--------------|----------------------|--------------|
| **CPU (libx264)** | 遅い (数分～数時間) | 高品質（圧縮効率◎） | 小さい（ビットレート低くてもOK） |
| **GPU (NVENC)**  | 速い（数秒～数分） | やや劣る（ビットレートを上げないと劣化） | 大きくなりがち |

###  4. どうすればGPUエンコードでも高画質を維持できる？
GPUエンコード（NVENC）で画質をなるべく良くしたいなら、以下の設定を試してください。

#### ✅ 1. ビットレートを上げる
GPUはビットレートが低いと画質が劣化しやすいので、`-b:v` で調整できます。

**例：5Mbpsに上げる**
```bash
ffmpeg -i input.mp4 -vcodec h264_nvenc -b:v 5M output.mp4
```
- **CPUなら3Mbpsで十分な画質**でも、
- **GPUなら5Mbpsにしないと同じ画質にならない**ことが多い感じ。

#### ✅ 2. プリセットを`slow`にする
プリセット（`-preset`）を`fast` → `slow`にすると、エンコードがかな〜り遅くなりますが、画質が向上します。

```bash
ffmpeg -i input.mp4 -vcodec h264_nvenc -preset slow output.mp4
```

#### ✅ 3. CRF（可変品質）を使う
一定の品質を保ちつつ、データ量を抑える方法です。

**例：CRF 23（品質重視）**[^2]
[^2]: NVENCを使う際に`-cq 23`のように記述していますが、これはソフトウェアエンコードのlibx264でいう`-crf 23 `とほぼ同じ概念です。ただし、実際にはx264とNVENCでは品質スケールが微妙に違うと言われることがあります。「目安として、20～23が高品質」などはx264のCRF指標がベースですが、NVENCの`-cq`でも大体同じように理解して問題ないと思います。ただし、ソースによってはもう少し値を下げないと画質が落ちるケースもあるので、あくまで目安としてもらうのが良きかもです。
```bash
ffmpeg -i input.mp4 -vcodec h264_nvenc -cq 23 output.mp4
```
- **値が小さいほど高品質（ファイルサイズが大きくなる）**
- **目安**
  - 20 ～ 23 → 高品質
  - 24 ～ 28 → バランス
  - 29 以上 → 低画質（ファイルサイズ小）

## 全ての動画をH.265に変換するbashスクリプト
:::details convert_h265.sh
```bash
#!/bin/bash

: '
Summary:
    カレントディレクトリ内のすべての.mp4ファイルをH.265（HEVC）に変換し、
    ファイル名の末尾に"_H265.mp4"を追加して保存する。
    変換後、元のファイルを削除する。

Example:
    chmod +x convert_h265.sh
    ./convert_h265.sh

Note:
    - 字幕や外国語吹き替えを**保持しません**。

License:
    This script is licensed under the terms provided by yKesamaru, the original author.
'

# カレントディレクトリのMP4ファイルを処理
for file in *.mp4; do
    # MP4ファイルが存在しない場合、スクリプトを終了
    [ -e "$file" ] || { echo "MP4ファイルが見つかりません。"; exit 1; }

    # ファイル名と拡張子を分割
    base_name="${file%.*}"
    ext="${file##*.}"

    # 出力ファイル名（末尾に"_H265"を追加）
    output_file="${base_name}_H265.${ext}"

    echo "変換中: $file → $output_file"

    # FFmpeg を使って H.265 に変換
    # ffmpeg -i "$file" -vcodec libx265 -crf 30 -preset faster -c:a aac -b:a 96k "$output_file"  # CPU
    ffmpeg -i "$file" -vcodec hevc_nvenc -cq 28 -preset fast -c:a aac -b:a 96k "$output_file"  # GPU

    # 変換が成功した場合、元のファイルを削除
    if [ $? -eq 0 ]; then
        echo "変換成功: $output_file"
        rm "$file"
        echo "削除完了: $file"
    else
        echo "変換失敗: $file"
    fi

done
```
もし**字幕や外国語吹き替えを保持したい場合**は以下を参考にしてください。[^4]
```bash
ffmpeg -i input.mp4 -c:v libx265 -preset medium -crf 22 -c:a copy -c:s copy output.mp4
```
-c:a copy → 音声はエンコードせずコピー（すべての音声トラックを維持）
-c:s copy → 字幕もコピー（すべての字幕トラックを維持）

[^4]: ただし、複数オーディオトラックや複数字幕トラックを確実にすべて保持したい場合は、コマンドの先頭あたりに`-map 0 `をつける方がより確実です（FFmpegはデフォルトだと特定ストリームのみを選択する場合があります）。`ffmpeg -i input.mp4 -map 0 -c:v libx265 -crf 22 -c:a copy -c:s copy output.mp4`

:::

## おわりに
FFmpegを使ったエンコードの基本的な理解と、ベンチマーク結果をまとめました。
お役に立てれば幸いです。

## 参考文献
- [FFmpeg速習](https://gist.github.com/krisfail/6faec6b943e5b7d575703f029f3b0850)
- [ffmpeg でよく利用するコマンド群](https://zenn.dev/pinto0309/scraps/94f2cbaa8d150a)
- [H.265/HEVC Video Encoding Guide ](https://trac.ffmpeg.org/wiki/Encode/H.265)
- [ffmpeg Documentation](https://www.ffmpeg.org/ffmpeg.html)
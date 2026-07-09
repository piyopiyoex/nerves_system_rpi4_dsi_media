# Raspberry Pi 4 向け DSI 表示対応 Nerves システムのカスタマイズ概要

## 概要

この文書は、`nerves_system_rpi4` をもとにした独自 Nerves システムの主なカスタマイズ内容をまとめる。

この fork は、[kurokouji](https://github.com/kurokouji) 氏および [piyopiyo.ex](https://github.com/piyopiyoex) メンバーによって作成・検証されたものである。

この fork は、Raspberry Pi 4 で DSI 接続の表示装置を使い、Scenic による画面表示、RTSP カメラ映像表示、MP4 再生、DRM/KMS 検証を行いやすくすることを目的としている。

## 基本方針

hardware、boot、kernel、映像処理に関わるコマンド群は Nerves システム側に置く。

一方で、表示内容、camera stream の扱い、画面構成などは Elixir/Scenic アプリ側に置く。

## 主な変更ファイル

```text
README.md
VERSION
linux-6.18.defconfig
nerves_defconfig
mix.exs
```

## DSI overlay files

ファームウェアイメージの boot partition には、DSI 表示用の overlay files が含まれている。

```text
overlays/vc4-kms-dsi-7inch.dtbo
overlays/vc4-kms-dsi-ili9881-5inch.dtbo
overlays/vc4-kms-dsi-ili9881-7inch.dtbo
```

この fork では、これらを使う前提で kernel 側の DSI/DRM 関連機能を module ではなく kernel 組み込み寄りに調整している。

## kernel 設定

DSI/DRM 表示系を起動時の早い段階で使えるようにするため、主要な表示関連機能を module ではなく kernel 組み込み寄りにする。

主な設定は次の通り。

```text
CONFIG_DRM=y
CONFIG_DRM_VC4=y
CONFIG_DRM_PANEL_ILITEK_ILI9881C=y
CONFIG_DRM_PANEL_SIMPLE=y
CONFIG_DRM_TOSHIBA_TC358762=y
CONFIG_DRM_FBDEV_EMULATION=y
CONFIG_FRAMEBUFFER_CONSOLE=y
CONFIG_BACKLIGHT_CLASS_DEVICE=y
CONFIG_BACKLIGHT_RPI=y
CONFIG_REGULATOR_RASPBERRYPI_TOUCHSCREEN_ATTINY=y
CONFIG_REGULATOR_RASPBERRYPI_TOUCHSCREEN_V2=y
```

これにより、DSI 表示系が module の読み込み順に依存しにくくなる。

## FFmpeg

RTSP カメラ映像の表示、MP4 再生、録画時間の取得のために FFmpeg を含める。

```text
BR2_PACKAGE_FFMPEG=y
BR2_PACKAGE_FFMPEG_FFMPEG=y
BR2_PACKAGE_FFMPEG_FFPROBE=y
BR2_PACKAGE_FFMPEG_SWSCALE=y
```

用途は次の通り。

- `ffmpeg` による RTSP カメラ映像表示
- `ffmpeg` による MP4 再生
- `ffprobe` による録画時間取得
- `swscale` による映像の拡大縮小

## GStreamer

DRM overlay plane の検証用に GStreamer を含める。

```text
BR2_PACKAGE_GSTREAMER1=y
BR2_PACKAGE_GSTREAMER1_INSTALL_TOOLS=y
BR2_PACKAGE_GST1_PLUGINS_BASE=y
BR2_PACKAGE_GST1_PLUGINS_GOOD=y
BR2_PACKAGE_GST1_PLUGINS_BAD=y
BR2_PACKAGE_GST1_PLUGINS_BAD_PLUGIN_KMS=y
BR2_PACKAGE_GST1_PLUGINS_GOOD_PLUGIN_V4L2=y
BR2_PACKAGE_GST1_PLUGINS_GOOD_PLUGIN_V4L2_PROBE=y
```

想定している検証経路は次の通り。

```text
rtspsrc -> rtph264depay -> h264parse -> v4l2h264dec -> videoconvert -> kmssink
```

`V4L2_PROBE` は、`v4l2h264dec` などの decoder elements を GStreamer から見えるようにするために必要である。

## software decode fallback

hardware decoder の挙動確認中でも検証を止めないため、`gst-libav` を含めて software decoder を使えるようにしている。

```text
BR2_PACKAGE_GST1_LIBAV=y
```

## DRM/KMS 診断

対象機器上で DRM/KMS の状態を確認するため、libdrm test tools を含める。

```text
BR2_PACKAGE_LIBDRM_INSTALL_TESTS=y
```

主に次のコマンドを使う。

```text
modetest
drm_info
```

## 確認方法

build 時に確認する。

```sh
mix deps.get
mix compile
mix nerves.system.lint nerves_defconfig

grep -n "CONFIG_DRM=y" linux-6.18.defconfig
grep -n "CONFIG_DRM_VC4=y" linux-6.18.defconfig
grep -n "CONFIG_DRM_PANEL_ILITEK_ILI9881C=y" linux-6.18.defconfig
grep -n "BR2_PACKAGE_FFMPEG" nerves_defconfig
grep -n "BR2_PACKAGE_GST1" nerves_defconfig
```

対象機器上で確認する。

```sh
ls -l /dev/dri
modetest -M vc4
drm_info

ffmpeg -version
ffprobe -version

gst-launch-1.0 --version
gst-inspect-1.0 kmssink
gst-inspect-1.0 v4l2h264dec
gst-inspect-1.0 avdec_h264
```

アプリ側では、次を確認する。

- DSI 表示装置が起動時に点灯する
- Scenic が画面を描画できる
- RTSP カメラ映像を表示できる
- MP4 再生と録画時間取得が動作する

## 運用上の注意

上流の `nerves_system_rpi4` に追従する場合は、DSI 関連の kernel 設定、FFmpeg、GStreamer、`gst-libav`、libdrm test tools が引き続き有効か確認する。

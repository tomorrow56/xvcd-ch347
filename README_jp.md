# xvcd-ch347

CH347 Xilinx Virtual Cable

xvcd (https://github.com/tmbinc/xvcd) をベースにした、XVC (Xilinx Virtual Cable) プロトコル対応の CH347 向け実装です。

[English README](README.md)

## ビルド方法

### Windows

Git と、Visual Studio（商用利用はライセンス必要）または Build Tools for Visual Studio（Microsoft より無償提供）をインストールしてください。

**Developer PowerShell for Visual Studio** を開き、以下のコマンドを実行します：

```powershell
git clone https://github.com/tomorrow56/xvcd-ch347.git
New-Item -Path 'xvcd-ch347\build' -ItemType Directory
Set-Location 'xvcd-ch347\build'
cmake ..
cmake --build . --config Release
```

`CH347DLLA64.DLL` は以下からダウンロードしてください：
[https://www.wch.cn/downloads/CH341PAR\_ZIP.html](https://www.wch.cn/downloads/CH341PAR_ZIP.html)

### Linux

依存パッケージをインストールします：

```bash
sudo apt-get update
sudo apt-get install libusb-1.0-0-dev
# libusb 0.0.1 系は通常不要です
# sudo apt-get install libusb-dev
sudo apt install build-essential cmake
```

または libusb を手動でビルドすることも可能です：https://github.com/libusb/libusb.git

```bash
git clone https://github.com/tomorrow56/xvcd-ch347.git
mkdir -p xvcd-ch347/build
cd xvcd-ch347/build
cmake ..
make
```

## Linux での注意事項

### Ubuntu での CH347T デバイス認識

Ubuntu では CH347T は以下のように認識されます（`lsusb` で確認済み）：

| IF番号 | USBクラス | 機能 | デバイスノード |
|--------|-----------|------|---------------|
| IF0 | CDC Communications (ACM) | UART 制御 | `/dev/ttyACM0` |
| IF1 | CDC Data | UART データ | IF0 とペア |
| IF2 | Vendor Specific | JTAG | libusb 直接アクセス |

- UART は `cdc_acm` カーネルドライバが自動ロードされ、**`/dev/ttyACM0`** として使用可能です。
- JTAG は IF2（Vendor Specific）を libusb 経由で直接アクセスします。
- **UART と JTAG は独立したインターフェースのため同時使用が可能**です。

### USB インターフェース番号について

CH347T (PID: `0x55DD`) と CH347F (PID: `0x55DE`) では、JTAG に使用する USB インターフェース番号が異なります：

| モデル | PID | インターフェース数 | JTAG インターフェース番号 |
|--------|-----|-------------------|--------------------------|
| CH347T | 0x55DD | 3 (IF0〜2) | 2 |
| CH347F | 0x55DE | 5 (IF0〜4) | 4 |

本フォークではこの問題を修正済みです。PID を自動判定して正しいインターフェース番号を選択します。

### udev ルール（root なしで実行する場合）

`/etc/udev/rules.d/99-ch347.rules` を作成して以下を記述します：

```
SUBSYSTEM=="usb", ATTR{idVendor}=="1a86", ATTR{idProduct}=="55dd", MODE="0666"
SUBSYSTEM=="usb", ATTR{idVendor}=="1a86", ATTR{idProduct}=="55de", MODE="0666"
```

ルールを反映します：

```bash
sudo udevadm control --reload-rules
sudo udevadm trigger
```

## 使い方

```bash
./xvcd_ch347_Linux [オプション]
```

| オプション | 説明 | デフォルト |
|-----------|------|-----------|
| `-h`, `--help` | ヘルプを表示 | |
| `-a`, `--address <アドレス>` | ホストアドレスを指定 | `127.0.0.1` |
| `-p`, `--port <ポート番号>` | ソケットポートを指定 | `2542` |
| `-i`, `--index <インデックス>` | CH347 デバイスインデックスを指定 | `0` |
| `-s`, `--speed <速度>` | JTAG クロック速度を指定（Hz） | `30000000` (30MHz) |
| `-S`, `--serialnum <シリアル番号>` | 使用する CH347 のシリアル番号を指定 | |

Vivado / Impact から接続する場合は以下の接続文字列を使用します：

```
xilinx_xvc host=localhost:2542 disableversioncheck=true
```

## LAN 経由でのリモート接続

CH347T を接続した Ubuntu マシンに対して、同一 LAN 上の別 PC から JTAG および UART にアクセスできます。

### XVC サーバーへの LAN 経由接続

`xvcd_ch347_Linux` はデフォルトで `0.0.0.0:2542`（全インターフェース）でリッスンするため、LAN 内の別 PC から直接アクセス可能です。

**Ubuntu 側：**

```bash
# IPアドレスを確認
ip addr show

# ファイアウォールでポート 2542 を開放（必要な場合）
sudo ufw allow 2542/tcp

# XVC サーバーを起動
./xvcd_ch347_Linux
```

**Vivado 側（別 PC）：**

```
xilinx_xvc host=<UbuntuのIPアドレス>:2542 disableversioncheck=true
```

例：Ubuntu の IP が `192.168.1.10` の場合：

```
xilinx_xvc host=192.168.1.10:2542 disableversioncheck=true
```

### シリアルポート（UART）の LAN 経由公開

`socat` を使って `/dev/ttyACM0` を TCP で公開できます。

**Ubuntu 側：**

```bash
sudo apt install socat

# /dev/ttyACM0 をポート 5000 で公開（ボーレートは環境に合わせて変更）
socat TCP-LISTEN:5000,reuseaddr,fork /dev/ttyACM0,raw,echo=0,b115200
```

**別 PC（Linux/Mac）からの接続：**

```bash
socat - TCP:<UbuntuのIP>:5000
```

**Windows からの接続：**

PuTTY などで TCP 接続（ホスト: `192.168.1.10`、ポート: `5000`）。

### JTAG と UART の同時リモート使用

CH347T では JTAG と UART が独立したインターフェースのため、両方を同時に公開できます：

| サービス | コマンド | ポート | 用途 |
|---------|---------|--------|------|
| XVC サーバー | `./xvcd_ch347_Linux` | 2542 | JTAG (Vivado/Impact) |
| socat | `socat TCP-LISTEN:5000,reuseaddr,fork /dev/ttyACM0,...` | 5000 | UART |

## FAQ

1. **クロック設定について**

   対応クロック：`KHZ(468.75)`, `KHZ(937.5)`, `MHZ(1.875)`, `MHZ(3.75)`, `MHZ(7.5)`, `MHZ(15)`, `MHZ(30)`, `MHZ(60)`

   10MHz が必要な場合は 7.5MHz または 15MHz を選択してください。
   `xvcd.c` 内の `DEFAULT_JTAG_SPEED` 変数を変更することで初期値を調整できます（例：15MHz の場合は `15000000`）。

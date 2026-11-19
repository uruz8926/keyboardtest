# nemu-link 対応パッチ (torabo-tsuki-lp)

このzipの中身は、`uruz8926/zmk-keyboard-torabo-tsuki-lp`(2026-11-14時点の最新)に**上書き/追加するファイルだけ**です。
フォルダ構成はリポジトリと同じなので、そのまま展開してコミットしてください。
(このファイル `NEMU-LINK-SETUP.md` はリポジトリの `README.md` とは別名です。不要なら削除してOK)

## 1. 反映するファイル

| パス | 内容 |
|---|---|
| `.github/workflows/build.yml` | ビルドworkflowを `@v0.3` → `@main` に変更 |
| `build.yaml` | 左右に`PERIPHERAL_ID`(0/1)を付与、`nemu-link-dongle`のビルドを追加 |
| `config/west.yml` | ZMKを nemu-link フォーク(`v0.4+nemu-link`)へ、`zmk-feature-nemu-link`を追加、BMP Boost / paw3222 / cdc-acm のrevisionを変更 |
| `config/keymap.keymap` | `&nl_mode`の追加、chord設定、ドングル用ガード(他のキー配置は変更なし) |
| `config/nemu-link-dongle.keymap` | ドングル用keymap(上のkeymapをそのまま読み込む) |
| `config/nemu-link-dongle.conf` | ドングルのKconfig |
| `config/nemu-link-dongle.overlay` | ドングルの物理レイアウト(71キー)、radioノード、トラックボール受信 |
| `boards/shields/torabo_tsuki_lp/torabo_tsuki_lp_{left,right}.conf` | nemu-linkのKconfigを追加 |
| `boards/shields/torabo_tsuki_lp/torabo_tsuki_lp_{left,right}.overlay` | 鍵保存用4KiB partition(BMP Boost用)とradioノードを追加 |

## 2. 操作キー (useful レイヤー = Enter長押し / iPadは ipad_useful)

| 位置 | キー | 動作 |
|---|---|---|
| 32 | `&nl_mode 1` | **2.4GHzドングル**に切り替え |
| 33 | `&nl_mode 0` | BLE(無線モード)に戻す。BLE/有線のどちらも、まずこれ |
| 34 | `&nl_mode 2` | BLE と 2.4GHz のトグル |
| 46 | `&out OUT_USB` | **有線**(右側のUSBケーブルからPCへ出力) |
| 47 | `&out OUT_BLE` | BLE出力 |

既存の `&bt BT_SEL 1〜3` / `BT_CLR` / `BT_DISC` などは、そのまま変わりません。起動直後はこれまで通りBLEです。

- **BLE**: 33(`nl_mode 0`) → 47(`OUT_BLE`)。通常はこれまで通りで、切替操作は不要です。
- **有線**: 33(`nl_mode 0`) → 46(`OUT_USB`)。右側にUSBケーブルをつなぎます。
- **ドングル**: 32(`nl_mode 1`)。左右とも自動でドングル経由になります。

## 3. 2.4GHzからBLEへ戻れなくなったとき(長押し操作)

左右それぞれのキーを、**その半分のキー2つ同時に長押し**します。

| 半分 | キー | 時間 |
|---|---|---|
| 左 | Esc + Tab (位置12, 24) | 2.5秒(2.4GHz→BLE)/ 1.5秒(BLE→2.4GHz) |
| 右 | ← + → (位置63, 65) | 同上 |

位置は `config/keymap.keymap` 冒頭の `nemu_link` ノードで変更できます。

## 4. 初回の手順

1. このzipの中身をリポジトリに反映(同名ファイルは上書き)してコミット → GitHub Actionsでビルド。
2. 成果物(UF2): `torabo_tsuki_lp_left_peripheral` / `torabo_tsuki_lp_right_central` / `nemu-link-dongle`
   を、左・右・ドングルにそれぞれ書き込む。
   (ドングルはボタンを押しながらPCに接続するとUF2ブートローダが起動)
3. Chrome/Edgeで <https://nemu-link-studio.pages.dev/> を開き、「nemu-link管理」タブで鍵とaddressを生成してバックアップ。
4. 左・右・ドングルをUSB接続し、**同じ鍵/address**を3台に登録。抜き差しで通信開始。

## 5. 未検証・要確認

この環境ではビルドできないため、**ビルドと実機動作は未確認**です。

- ZMKが v0.3 → nemu-linkフォーク(`v0.4+nemu-link`、Zephyr 4.1)に上がるため、他モジュールの互換性は次の推測です。
  エラーが出たらログを教えてください。
  - `zmk-component-bmp-boost`: 新しいボード形式のため `master`
  - `zmk-driver-paw3222`: `torabo-tsuki-nemu-link` ブランチ
  - `zmk-feature-cdc-acm-bootloader-trigger`: `main`
  - `zmk-driver-iqs7211e`(トラックパッド)、`zmk-feature-status-led`、`kscan-4-direction-switch`: 変更なし
- `config/boards/arm/bmp_boost/` は旧形式のボード定義です。ビルドで衝突したらフォルダごと削除してください。
- 鍵保存partition(`0xd7000`, 4KiB)は、nemu-link README記載のBMP Boost用の値です(BMP Boost `master` の `code_partition` と一致)。
- ドングル側のトラックボールは、右側の `pointing_listener` がESBへ送る前提(listener index 0 = reg 0)で組んでいます。
  トラックボールがドングル経由で動かないときは、nemu-linkの `sekigon,nemu-link-input-reg` で対応づけを調整します。
- 右側は「BLE時はセントラル / 2.4GHz時はペリフェラル」の両方になるため、`CONFIG_ZMK_USB=y` を右に足しています。
- `CONFIG_ZMK_STUDIO`(ZMK Studio)は、nemu-link経由での動作は確認していません。
- ドングルがESB中に`&nl_mode 0`でBLEへ戻す処理はバイナリ(blob)内なので読めておらず、上記の長押しを保険にしてください。

# CLAUDE.md

このファイルは、リポジトリ内のコードを操作する際に Claude Code (claude.ai/code) へ提供するガイダンスです。

## プロジェクト概要

**STM32F4 Discovery ボード**（ARM Cortex-M4）向けのベアメタル・プリエンプティブタスクスケジューラ。C 言語とインライン ARM アセンブリで実装。RTOS は使用せず、スケジューラ・コンテキストスイッチ・割り込み処理をすべてスクラッチで実装している。

## ビルドコマンド

`arm-none-eabi-gcc` ツールチェーンと `make` が必要。

```bash
make          # ファームウェアをビルド → final.elf
make semi     # セミホスティング付きビルド（GDB の stdio リダイレクト）→ final_sh.elf
make clean    # *.o、*.elf、*.map を削除
make load     # OpenOCD 経由でフラッシュ書き込み（board/stm32f4discovery.cfg）
```

コンパイラフラグ: `-mcpu=cortex-m4 -mthumb -mfloat-abi=soft -std=gnu11 -Wall -O0`

自動テストフレームワークはなく、動作確認はハードウェアへの書き込みと LED の挙動観察で行う。

## アーキテクチャ

### スケジューラ（`main.c`）

- **タスクスロット数: 5**（ユーザータスク 4 + アイドルタスク 1）。各タスクに 1 KB のスタック（256 ワード）を割り当て。
- **タスクコントロールブロック（TCB）**: PSP・ブロックカウント・状態・ハンドラ関数ポインタを保持。
- **スタック**は `SRAM_END` から下方向に伸びる。カーネルは MSP を使用し、各ユーザータスクは個別の PSP を持つ（`CONTROL` レジスタで切り替え）。
- **SysTick** は 1000 Hz（16 MHz HSI / 1000）で発火し、各タスクのブロックカウントをデクリメント、実行可能タスクをアンブロックしたのち `PendSV` をペンディング状態にする。
- **`PendSV_Handler`**（最低優先度）が実際のコンテキストスイッチを担当: R4–R11 を保存・復元し、次の実行可能タスクの PSP に切り替える。
- **`task_delay(ticks)`** は指定ティック数だけ実行中タスクをブロックし、即座に再スケジュールをトリガーする。

### LED ドライバ（`led.c` / `led.h`）

ポート D（AHB1 バス）の GPIO レジスタを直接操作。ピン 12〜15 がそれぞれ Green・Orange・Red・Blue LED に対応。関数: `led_init_all()`、`led_on(pin)`、`led_off(pin)`。

### メモリマップ（`stm32_ls.ld`）

| 領域  | アドレス     | サイズ   |
|-------|------------|--------|
| FLASH | 0x08000000 | 1024 KB |
| SRAM  | 0x20000000 | 128 KB  |

セクション: `.isr_vector` と `.text` → FLASH；`.data`（FLASH からロード）と `.bss` → SRAM。

### 主要ヘッダ

- `main.h`: タスク数、スタックサイズ（`1024` バイト）、SRAM ベース・終端アドレス、`INTERRUPT_DISABLE`/`ENABLE` マクロ。
- `led.h`: LED ピン番号、ビジーウェイト用ディレイ定数。

### スタートアップ（`stm32_startup.c`）

ベクタテーブルを定義し、`.data` を FLASH から SRAM へコピー、`.bss` をゼロクリアしたのち `main()` を呼び出す。フォルトハンドラ（HardFault・MemManage・BusFault）もここで定義。

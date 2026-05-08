# MMF2 (Multi-Media Framework v2) API 仕様書

## 概要

MMF2 は Realtek Ameba Pro2 向けの組み込みマルチメディアパイプラインフレームワークです。
各メディア処理機能を「モジュール」として実装し、それらを「リンカー」でパイプライン状に接続することで、
映像・音声の取得・エンコード・ネットワーク送信・ファイル保存・AI 推論などの一連の処理を構成できます。

---

## ディレクトリ構成

```
mmfv2/
├── mmf2.h                   # フレームワーク全体のインクルードヘッダ
├── mmf2_module.h            # モジュール基盤・データ型定義
├── mmf2_link.h              # モジュール間接続コマンド定義
├── mmf2_siso.h              # SISO リンカー (Single-Input Single-Output)
├── mmf2_miso.h              # MISO リンカー (Multi-Input Single-Output)
├── mmf2_simo.h              # SIMO リンカー (Single-Input Multi-Output)
├── mmf2_mimo.h              # MIMO リンカー (Multi-Input Multi-Output)
├── mmf2_dbg.h               # デバッグマクロ
├── mmf2_mediatime_8735b.h   # メディアタイム管理 (8735B 専用)
├── memory_encoder.h         # エンコーダ用メモリプール
├── module_video.h/.c        # 映像ソースモジュール
├── module_audio.h/.c        # 音声 I/O モジュール
├── module_rtsp2.h/.c        # RTSP ストリーミングモジュール
├── module_rtp.h/.c          # RTP 受信モジュール
├── module_aac.h/.c          # AAC エンコーダモジュール
├── module_aad.h/.c          # AAC デコーダモジュール
├── module_g711.h/.c         # G.711 コーデックモジュール
├── module_i2s.h/.c          # I2S 音声インターフェースモジュール
├── module_opusc.h/.c        # Opus エンコーダモジュール
├── module_opusd.h/.c        # Opus デコーダモジュール
├── module_mp4.h/.c          # MP4 録画モジュール
├── module_fmp4.h/.c         # フラグメント MP4 モジュール
├── module_demuxer.h/.c      # MP4 デマルチプレクサモジュール
├── module_queue.h/.c        # キューバッファモジュール
├── module_array.h/.c        # 配列データソースモジュール
├── module_fileloader.h/.c   # ファイルローダーモジュール
├── module_filesaver.h/.c    # ファイルセーバーモジュール
├── module_md.h/.c           # モーション検知モジュール
├── module_eip.h/.c          # 拡張画像処理モジュール
├── module_vipnn.h/.c        # VIP ニューラルネット推論モジュール
├── module_facerecog.h/.c    # 顔認識モジュール
├── module_httpfs.h/.c       # HTTP ファイルサーバーモジュール
├── module_uvcd.h/.c         # USB UVC デバイスモジュール
├── module_web_viewer.h/.c   # WebSocket ビューアモジュール
└── module_dup.h/.c          # データ複製モジュール
```

---

## 1. コアフレームワーク

### 1.1 モジュール基盤 (`mmf2_module.h`)

#### モジュールタイプ定数

| 定数 | 値 | 説明 |
|------|----|------|
| `MM_TYPE_NONE` | `0x00` | 出力なし |
| `MM_TYPE_VSRC` | `0x10` | 映像ソース (ISP / ファイル) |
| `MM_TYPE_ASRC` | `0x11` | 音声ソース (I2S / ファイル / RTP) |
| `MM_TYPE_VDSP` | `0x20` | 映像 DSP (H.264 ENC / JPEG ENC) |
| `MM_TYPE_ADSP` | `0x21` | 音声 DSP (AAC ENC / AAC DEC) |
| `MM_TYPE_VSINK` | `0x41` | 映像シンク (RTSP / ファイル) |
| `MM_TYPE_ASINK` | `0x42` | 音声シンク (RTSP / I2S / ファイル) |
| `MM_TYPE_AVSINK` | `0x43` | 音声映像シンク (RTSP / ファイル) |

#### 共通コマンド (`MM_CMD_*`)

| 定数 | 値 | 説明 |
|------|----|------|
| `MM_CMD_INIT_QUEUE_ITEMS` | `0x00` | 静的キューアイテムの初期化 |
| `MM_CMD_SET_QUEUE_LEN` | `0x01` | キュー長の設定 |
| `MM_CMD_SET_QUEUE_NUM` | `0x02` | キュー数の設定 |
| `MM_CMD_SELECT_QUEUE` | `0x03` | キューの選択 (0〜3) |
| `MM_CMD_CLEAR_QUEUE_ITEMS` | `0x04` | 全使用ポートのキューアイテムクリア |
| `MM_CMD_SET_DATAGROUP` | `0x05` | データグループの設定 |
| `MM_CMD_CLEAR_QUEUE_ITEMS_SEL_PORT` | `0x06` | 指定ポートのキューアイテムクリア |

#### モジュール状態定数

| 定数 | 値 | 説明 |
|------|----|------|
| `MM_STAT_INIT` | `0x00` | 初期化済み |
| `MM_STAT_READY` | `0x01` | 準備完了 |
| `MM_STAT_ERROR` | `0x10` | エラー |
| `MM_STAT_ERR_MALLOC` | `0x11` | メモリ確保エラー |
| `MM_STAT_ERR_QUEUE` | `0x12` | キューエラー |
| `MM_STAT_ERR_NEWITEM` | `0x13` | 新規アイテム生成エラー |

#### 構造体

##### `mm_module_t` — モジュール定義構造体

```c
typedef struct mm_module_s {
    void *(*create)(void *);                          // インスタンス生成
    void *(*destroy)(void *);                         // インスタンス破棄
    int  (*control)(void *, int, int);                // コマンド制御
    int  (*handle)(void *, void *, void *);           // データ処理ハンドラ
    void *(*new_item)(void *);                        // 出力バッファ確保
    void *(*del_item)(void *, void *);                // 出力バッファ解放
    void *(*rsz_item)(void *, void *, int);           // 出力バッファリサイズ
    void *(*vrelease_item)(void *, void *, int);      // バッファ参照解放
    uint32_t    output_type;                          // 出力データ型
    uint32_t    module_type;                          // モジュール種別
    const char *name;                                 // モジュール名
} mm_module_t;
```

##### `mm_context_t` — モジュールインスタンス構造体

```c
typedef struct mm_contex_s {
    union {
        struct {
            xQueueHandle output_ready;    // 出力準備完了キュー
            xQueueHandle output_recycle;  // 出力リサイクルキュー
            int32_t      item_num;
        };
        mm_conveyor_t port[4];            // マルチポート対応
    };
    mm_module_t *module;
    void        *priv;                    // モジュール固有データ
    int          group_role;             // データシェアグループフラグ
    uint32_t     state;                  // モジュール状態
    int32_t      queue_num;             // キュー数
    int32_t      curr_queue;            // 現在選択中のキュー
} mm_context_t;
```

##### `mm_queue_item_t` — キューアイテム構造体

```c
typedef struct mm_queue_item_s {
    uint32_t flag;          // MMQI_FLAG_STATIC / DYNAMIC / READY / USED
    uint32_t data_addr;     // データアドレス
    void    *sema[4];       // データアドレス用ミューテックス
    void   **ref_sema;      // ソースミューテックスへのポインタ
    uint32_t timestamp;     // タイムスタンプ
    uint32_t hw_timestamp;  // ハードウェアタイムスタンプ
    uint32_t type;          // データ種別
    uint32_t size;          // データサイズ
    union {
        uint32_t index;     // RTSP 用インデックス
        uint32_t priv_data;
    };
    uint32_t in_idx;        // 入力インデックス
    char     name[256];     // アイテム名
} mm_queue_item_t;
```

#### 関数

| 関数 | シグネチャ | 説明 |
|------|-----------|------|
| `mm_module_open` | `mm_context_t *mm_module_open(mm_module_t *mm)` | モジュールインスタンスを生成・オープンする |
| `mm_module_close` | `mm_context_t *mm_module_close(mm_context_t *ctx)` | モジュールインスタンスをクローズ・解放する |
| `mm_module_ctrl` | `int mm_module_ctrl(mm_context_t *ctx, int cmd, int arg)` | モジュールにコマンドを送信する |

---

### 1.2 リンカー接続コマンド (`mmf2_link.h`)

モジュール間のパイプライン接続を制御するコマンド群。

#### 入力・出力ポート定義

| 定数 | 値 | 説明 |
|------|----|------|
| `MMIC_CMD_ADD_INPUT` | `0x00` | 入力ポート 0 の追加 |
| `MMIC_CMD_ADD_INPUT1` | `0x01` | 入力ポート 1 の追加 |
| `MMIC_CMD_ADD_INPUT2` | `0x02` | 入力ポート 2 の追加 |
| `MMIC_CMD_ADD_INPUT3` | `0x03` | 入力ポート 3 の追加 |
| `MMIC_CMD_ADD_OUTPUT` | `0x10` | 出力ポート 0 の追加 |
| `MMIC_CMD_ADD_OUTPUT1` | `0x11` | 出力ポート 1 の追加 |
| `MMIC_CMD_ADD_OUTPUT2` | `0x12` | 出力ポート 2 の追加 |
| `MMIC_CMD_ADD_OUTPUT3` | `0x13` | 出力ポート 3 の追加 |

#### タスク制御コマンド

| 定数 | 値 | 説明 |
|------|----|------|
| `MMIC_CMD_SET_STACKSIZE` | `0x20` | タスクスタックサイズの設定 |
| `MMIC_CMD_SET_TASKPRIORITY` | `0x21` | タスク優先度の設定 |
| `MMIC_CMD_SET_SECURE_CONTEXT` | `0x22` | セキュアコンテキストの設定 |
| `MMIC_CMD_SET_CTRL_TIMEOUT` | `0x23` | 制御タイムアウトの設定 |
| `MMIC_CMD_SET_TASKNANE` | `0x30` | タスク名の設定 (ポート 0〜3 対応) |

#### 出力状態コマンド

| 定数 | 値 | 説明 |
|------|----|------|
| `MMIC_CMD_SET_OUT_STAT` | `0x40` | 出力ステータスの設定 (ポート 0〜3) |
| `MMIC_CMD_GET_OUT_STAT` | `0x50` | 出力ステータスの取得 (ポート 0〜3) |

#### ステータス定数

| 定数 | 値 | 説明 |
|------|----|------|
| `MMIC_STAT_INIT` | `0x00` | 初期化済み |
| `MMIC_STAT_EXIT` | `0x01` | 停止 |
| `MMIC_STAT_RUN` | `0x02` | 実行中 |
| `MMIC_STAT_PAUSE` | `0x04` | 一時停止 |
| `MMIC_STAT_SET_EXIT` | `0x10` | 停止要求 |
| `MMIC_STAT_SET_PAUSE` | `0x20` | 一時停止要求 |
| `MMIC_STAT_SET_RUN` | `0x40` | 実行要求 |

---

### 1.3 SISO リンカー (`mmf2_siso.h`)

Single-Input Single-Output のパイプライン接続。

#### 構造体 `mm_siso_t`

| フィールド | 型 | 説明 |
|-----------|-----|------|
| `input` | `mm_context_t *` | 入力モジュールコンテキスト |
| `output` | `mm_context_t *` | 出力モジュールコンテキスト |
| `input_port_idx` | `int` | 入力ポートインデックス (デフォルト: 0) |
| `status` | `uint32_t` | 状態 |
| `stack_size` | `uint32_t` | タスクスタックサイズ |
| `task_priority` | `uint32_t` | タスク優先度 |
| `taskname` | `char[16]` | タスク名 |
| `task` | `xTaskHandle` | FreeRTOS タスクハンドル |

#### 関数

| 関数 | シグネチャ | 説明 |
|------|-----------|------|
| `siso_create` | `mm_siso_t *siso_create(void)` | SISO インスタンスを生成する |
| `siso_delete` | `mm_siso_t *siso_delete(mm_siso_t *siso)` | SISO インスタンスを解放する |
| `siso_start` | `int siso_start(mm_siso_t *siso)` | SISO パイプラインを開始する |
| `siso_stop` | `void siso_stop(mm_siso_t *siso)` | SISO パイプラインを停止する |
| `siso_pause` | `void siso_pause(mm_siso_t *siso)` | SISO パイプラインを一時停止する |
| `siso_resume` | `void siso_resume(mm_siso_t *siso)` | SISO パイプラインを再開する |
| `siso_ctrl` | `void siso_ctrl(mm_siso_t *siso, uint32_t cmd, uint32_t arg1, uint32_t arg2)` | SISO にコマンドを送信する |

---

### 1.4 SIMO リンカー (`mmf2_simo.h`)

Single-Input Multi-Output のパイプライン接続。1 入力から最大 4 出力に分配。

#### 構造体 `mm_simo_t`

| フィールド | 型 | 説明 |
|-----------|-----|------|
| `input` | `mm_context_t *` | 入力モジュールコンテキスト |
| `output_cnt` | `int` | 出力数 |
| `output[4]` | `mm_context_t *` | 出力モジュールコンテキスト (最大 4) |
| `pause_mask` | `uint32_t` | 一時停止マスク (ビット対応) |
| `status[4]` | `uint32_t` | 各出力の状態 |
| `task[4]` | `xTaskHandle` | 各出力タスクのハンドル |

#### 関数

| 関数 | シグネチャ | 説明 |
|------|-----------|------|
| `simo_create` | `mm_simo_t *simo_create(void)` | SIMO インスタンスを生成する |
| `simo_delete` | `mm_simo_t *simo_delete(mm_simo_t *simo)` | SIMO インスタンスを解放する |
| `simo_start` | `int simo_start(mm_simo_t *simo)` | SIMO パイプラインを開始する |
| `simo_stop` | `void simo_stop(mm_simo_t *simo)` | SIMO パイプラインを停止する |
| `simo_pause` | `void simo_pause(mm_simo_t *simo, uint32_t pause_mask)` | 指定出力を一時停止する |
| `simo_resume` | `void simo_resume(mm_simo_t *simo)` | 全出力を再開する |
| `simo_ctrl` | `void simo_ctrl(mm_simo_t *simo, uint32_t cmd, uint32_t arg1, uint32_t arg2)` | SIMO にコマンドを送信する |

---

### 1.5 MISO リンカー (`mmf2_miso.h`)

Multi-Input Single-Output のパイプライン接続。最大 4 入力を 1 出力にマージ。

#### 構造体 `mm_miso_t`

| フィールド | 型 | 説明 |
|-----------|-----|------|
| `input_cnt` | `int` | 入力数 |
| `input[4]` | `mm_context_t *` | 入力モジュールコンテキスト (最大 4) |
| `input_port_idx[4]` | `int` | 各入力のポートインデックス |
| `output` | `mm_context_t *` | 出力モジュールコンテキスト |
| `pause_mask` | `uint32_t` | 一時停止マスク |
| `status` | `uint32_t` | 状態 |

#### 関数

| 関数 | シグネチャ | 説明 |
|------|-----------|------|
| `miso_create` | `mm_miso_t *miso_create(void)` | MISO インスタンスを生成する |
| `miso_delete` | `mm_miso_t *miso_delete(mm_miso_t *miso)` | MISO インスタンスを解放する |
| `miso_start` | `int miso_start(mm_miso_t *miso)` | MISO パイプラインを開始する |
| `miso_stop` | `void miso_stop(mm_miso_t *miso)` | MISO パイプラインを停止する |
| `miso_pause` | `void miso_pause(mm_miso_t *miso, uint32_t pause_mask)` | 指定入力を一時停止する |
| `miso_resume` | `void miso_resume(mm_miso_t *miso)` | 全入力を再開する |
| `miso_ctrl` | `void miso_ctrl(mm_miso_t *miso, uint32_t cmd, uint32_t arg1, uint32_t arg2)` | MISO にコマンドを送信する |

---

### 1.6 MIMO リンカー (`mmf2_mimo.h`)

Multi-Input Multi-Output のパイプライン接続。最大 4 入力・4 出力。

#### 構造体 `mm_mimo_t`

| フィールド | 型 | 説明 |
|-----------|-----|------|
| `input_cnt` | `int` | 入力数 |
| `input[4]` | `mm_context_t *` | 入力モジュールコンテキスト (最大 4) |
| `output_cnt` | `int` | 出力数 |
| `output[4]` | `mm_context_t *` | 出力モジュールコンテキスト (最大 4) |
| `output_dep[4]` | `uint32_t` | 各出力が依存する入力 (ビットマスク) |
| `input_mask[4]` | `uint32_t` | 各入力を参照する出力 (ビットマスク) |
| `pause_mask[4]` | `uint32_t` | 各出力の一時停止マスク |
| `status[4]` | `uint32_t` | 各出力の状態 |

#### 関数

| 関数 | シグネチャ | 説明 |
|------|-----------|------|
| `mimo_create` | `mm_mimo_t *mimo_create(void)` | MIMO インスタンスを生成する |
| `mimo_delete` | `mm_mimo_t *mimo_delete(mm_mimo_t *mimo)` | MIMO インスタンスを解放する |
| `mimo_start` | `int mimo_start(mm_mimo_t *mimo)` | MIMO パイプラインを開始する |
| `mimo_stop` | `void mimo_stop(mm_mimo_t *mimo)` | MIMO パイプラインを停止する |
| `mimo_pause` | `void mimo_pause(mm_mimo_t *mimo, uint32_t pause_mask)` | 指定出力を一時停止する |
| `mimo_resume` | `void mimo_resume(mm_mimo_t *mimo)` | 全出力を再開する |
| `mimo_ctrl` | `void mimo_ctrl(mm_mimo_t *mimo, uint32_t cmd, uint32_t arg1, uint32_t arg2)` | MIMO にコマンドを送信する |

---

### 1.7 デバッグ (`mmf2_dbg.h`)

#### デバッググループフラグ

| 定数 | 値 | 対象 |
|------|----|------|
| `_MMF_DBG_RTSP_` | `0x00000001` | RTSP |
| `_MMF_DBG_RTP_` | `0x00000002` | RTP |
| `_MMF_DBG_ISP_` | `0x00000004` | ISP |
| `_MMF_DBG_ENCODER_` | `0x00000008` | エンコーダ |
| `_MMF_DBG_VIDEO_` | `0x00000010` | ビデオ |
| `_MMF_DBG_AUDIO_` | `0x00000020` | オーディオ |
| `_MMF_DBG_MODULE_` | `0x00000040` | モジュール |
| `_MMF_DBG_LINKER_` | `0x00000080` | リンカー |

#### デバッグ制御マクロ

| マクロ | 説明 |
|--------|------|
| `DBG_MMF_ERR_MSG_ON(x)` | 指定グループのエラーメッセージを有効化 |
| `DBG_MMF_WARN_MSG_ON(x)` | 指定グループの警告メッセージを有効化 |
| `DBG_MMF_INFO_MSG_ON(x)` | 指定グループの情報メッセージを有効化 |
| `DBG_MMF_ERR_MSG_OFF(x)` | 指定グループのエラーメッセージを無効化 |
| `DBG_MMF_WARN_MSG_OFF(x)` | 指定グループの警告メッセージを無効化 |
| `DBG_MMF_INFO_MSG_OFF(x)` | 指定グループの情報メッセージを無効化 |

#### デバッグ出力マクロ (各グループ共通パターン)

各グループ (`RTSP`, `RTP`, `ISP`, `ENCODER`, `VIDEO`, `AUDIO`, `MODULE`, `LINKER`) に対して以下のマクロが定義されています。

- `<GROUP>_DBG_ERROR(...)` — エラーログ出力
- `<GROUP>_DBG_WARNING(...)` — 警告ログ出力
- `<GROUP>_DBG_INFO(...)` — 情報ログ出力

#### 関数

| 関数 | シグネチャ | 説明 |
|------|-----------|------|
| `mm_set_abort_func` | `void mm_set_abort_func(void (*func)(void))` | アボート時に呼ばれるコールバックを登録する |
| `mmf2_abort` | `void mmf2_abort(void)` | フレームワークのアボート処理を実行する |

---

### 1.8 メディアタイム管理 (`mmf2_mediatime_8735b.h`)

8735B プラットフォーム専用のシステム時刻 API。

| 関数 | シグネチャ | 説明 |
|------|-----------|------|
| `mm_read_mediatime_us` | `uint64_t mm_read_mediatime_us(void)` | メディアタイムを µs 単位で取得する |
| `mm_read_mediatime_ms` | `uint32_t mm_read_mediatime_ms(void)` | メディアタイムを ms 単位で取得する |
| `mm_read_mediatime_us_fromisr` | `uint64_t mm_read_mediatime_us_fromisr(void)` | ISR コンテキストから µs 単位で取得する |
| `mm_read_mediatime_ms_fromisr` | `uint32_t mm_read_mediatime_ms_fromisr(void)` | ISR コンテキストから ms 単位で取得する |
| `mm_set_mediatime_in_us` | `void mm_set_mediatime_in_us(uint64_t mediatime_in_us)` | メディアタイムを µs 単位で設定する |
| `mm_set_mediatime_in_ms` | `void mm_set_mediatime_in_ms(uint32_t mediatime_in_ms)` | メディアタイムを ms 単位で設定する |

---

### 1.9 エンコーダメモリプール (`memory_encoder.h`)

動画エンコーダ用の固定ブロックメモリプール管理。

#### 構造体 `encoder_buffer`

| フィールド | 型 | 説明 |
|-----------|-----|------|
| `memory_lock` | `_mutex` | ミューテックス |
| `pool` | `uint8_t *` | メモリプール先頭 |
| `pool_table` | `uint8_t *` | ブロック使用状態テーブル |
| `max_pool_blk` | `int32_t` | 最大ブロック数 |
| `blk_size` | `int32_t` | 1 ブロックのサイズ (バイト) |
| `mem_size` | `int32_t` | 合計プールサイズ (バイト) |

#### 関数

| 関数 | シグネチャ | 説明 |
|------|-----------|------|
| `memory_init` | `void *memory_init(int size, int block_size)` | メモリプールを初期化して返す |
| `memory_deinit` | `void memory_deinit(void *ctx)` | メモリプールを解放する |
| `memory_alloc` | `void *memory_alloc(void *ctx, uint32_t size)` | プールからメモリを確保する |
| `memory_realloc` | `uint8_t *memory_realloc(void *ctx, uint8_t *addr, uint32_t size)` | 確保済みメモリをリサイズする |
| `memory_free` | `void memory_free(void *ctx, uint8_t *addr)` | プールにメモリを返却する |
| `dump_info` | `void dump_info(void *ctx)` | プール状態をデバッグ出力する |

---

## 2. メディアモジュール

### 2.1 映像ソースモジュール (`module_video.h`)

ISP カメラまたはファイルからの映像を取得し、H.264 / JPEG などのフォーマットで出力する。

#### モジュールインスタンス

```c
extern mm_module_t video_module;
```

#### 制御コマンド

| コマンド | 値 | 説明 |
|---------|-----|------|
| `CMD_VIDEO_SET_PARAMS` | `MM_MODULE_CMD(0x00)` | パラメータ一括設定 |
| `CMD_VIDEO_GET_PARAMS` | `MM_MODULE_CMD(0x01)` | パラメータ一括取得 |
| `CMD_VIDEO_SET_HEIGHT` | `MM_MODULE_CMD(0x02)` | 映像高さ設定 |
| `CMD_VIDEO_SET_WIDTH` | `MM_MODULE_CMD(0x03)` | 映像幅設定 |
| `CMD_VIDEO_BITRATE` | `MM_MODULE_CMD(0x04)` | ビットレート設定 |
| `CMD_VIDEO_FPS` | `MM_MODULE_CMD(0x05)` | フレームレート設定 |
| `CMD_VIDEO_GOP` | `MM_MODULE_CMD(0x06)` | GOP 設定 |
| `CMD_VIDEO_MEMORY_SIZE` | `MM_MODULE_CMD(0x07)` | メモリサイズ設定 |
| `CMD_VIDEO_BLOCK_SIZE` | `MM_MODULE_CMD(0x08)` | ブロックサイズ設定 |
| `CMD_VIDEO_MAX_FRAME_SIZE` | `MM_MODULE_CMD(0x09)` | 最大フレームサイズ設定 |
| `CMD_VIDEO_RCMODE` | `MM_MODULE_CMD(0x0a)` | レート制御モード設定 |
| `CMD_VIDEO_FORCE_IFRAME` | `MM_MODULE_CMD(0x0e)` | 強制 I フレーム生成 |
| `CMD_VIDEO_STREAMID` | `MM_MODULE_CMD(0x10)` | ストリーム ID 設定 |
| `CMD_VIDEO_FORMAT` | `MM_MODULE_CMD(0x11)` | 映像フォーマット設定 |
| `CMD_VIDEO_SNAPSHOT` | `MM_MODULE_CMD(0x13)` | スナップショット取得 |
| `CMD_VIDEO_SNAPSHOT_CB` | `MM_MODULE_CMD(0x14)` | スナップショットコールバック設定 |
| `CMD_VIDEO_APPLY` | `MM_MODULE_CMD(0x20)` | 設定を適用する |
| `CMD_VIDEO_UPDATE` | `MM_MODULE_CMD(0x21)` | 設定を更新する |
| `CMD_VIDEO_STREAM_STOP` | `MM_MODULE_CMD(0x23)` | ストリーム停止 |
| `CMD_VIDEO_SET_TIMESTAMP_OFFSET` | `MM_MODULE_CMD(0x25)` | タイムスタンプオフセット設定 |
| `CMD_VIDEO_SET_SENSOR_ID` | `MM_MODULE_CMD(0x28)` | センサー ID 設定 |
| `CMD_VIDEO_MD_SET_ROI` | `MM_MODULE_CMD(0x31)` | モーション検知 ROI 設定 |
| `CMD_VIDEO_MD_SET_SENSITIVITY` | `MM_MODULE_CMD(0x32)` | モーション検知感度設定 |
| `CMD_VIDEO_MD_START` | `MM_MODULE_CMD(0x33)` | モーション検知開始 |
| `CMD_VIDEO_MD_STOP` | `MM_MODULE_CMD(0x34)` | モーション検知停止 |
| `CMD_VIDEO_SET_PRIVATE_MASK` | `MM_MODULE_CMD(0x37)` | プライバシーマスク設定 |
| `CMD_VIDEO_BPS_STBL_CTRL_EN` | `MM_MODULE_CMD(0x40)` | BPS 安定化制御の有効化 |
| `CMD_VIDEO_GET_CURRENT_BITRATE` | `MM_MODULE_CMD(0x44)` | 現在のビットレート取得 |
| `CMD_VIDEO_SET_MAX_QP` | `MM_MODULE_CMD(0x47)` | 最大 QP 設定 |
| `CMD_VIDEO_SET_SPS_PPS_INFO` | `MM_MODULE_CMD(0x48)` | SPS/PPS 情報設定 |
| `CMD_VIDEO_GET_SPS_PPS_INFO` | `MM_MODULE_CMD(0x49)` | SPS/PPS 情報取得 |

#### グローバル関数

| 関数 | シグネチャ | 説明 |
|------|-----------|------|
| `video_voe_presetting` | `int video_voe_presetting(int v1_enable, int v1_w, int v1_h, int v1_bps, int v1_snapshot, ...)` | VOE ヒープを最大 4 チャンネル分プリ設定する |
| `video_voe_presetting_by_params` | `int video_voe_presetting_by_params(const void *v1_params, int v1_jpg_only_snapshot, ...)` | パラメータ構造体を用いた VOE プリ設定 |
| `video_extra_voe_presetting` | `int video_extra_voe_presetting(int originl_heapsize, int vext_enable, int vext_w, int vext_h, int vext_bps, int vext_snapshot)` | 追加 VOE チャンネルのプリ設定 |
| `video_voe_release` | `void video_voe_release(void)` | VOE リソースを解放する |
| `video_set_sensor_id` | `void video_set_sensor_id(int SensorName)` | センサー ID を設定する |
| `video_setup_sensor` | `void video_setup_sensor(void *sensor_setup_cb)` | センサーセットアップコールバックを登録する |
| `video_show_fps` | `void video_show_fps(int enable)` | FPS 表示の有効/無効を切り替える |
| `video_get_cb_fps` | `int video_get_cb_fps(int chn)` | チャンネルのコールバック FPS を取得する |
| `video_set_fps_dropframe_mode` | `void video_set_fps_dropframe_mode(int drop_frame)` | FPS ドロップフレームモードを設定する |

---

### 2.2 音声 I/O モジュール (`module_audio.h`)

内蔵 ADC/DAC (AMIC / DMIC) を使った音声入出力。AEC・NS・AGC・VAD などの ASP 処理をサポート。

#### モジュールインスタンス

```c
extern mm_module_t audio_module;
extern audio_params_t default_audio_params;
extern TX_cfg_t default_tx_asp_params;
extern RX_cfg_t default_rx_asp_params;
```

#### マイクタイプ列挙 (`audio_mic_type`)

| 値 | 定数 | 説明 |
|----|------|------|
| `0` | `USE_AUDIO_AMIC` | アナログマイク (AMIC) |
| `1` | `USE_AUDIO_LEFT_DMIC` | 左チャンネル DMIC |
| `2` | `USE_AUDIO_RIGHT_DMIC` | 右チャンネル DMIC |
| `3` | `USE_AUDIO_STEREO_DMIC` | ステレオ DMIC |

#### 制御コマンド

| コマンド | 値 | 説明 |
|---------|-----|------|
| `CMD_AUDIO_SET_PARAMS` | `MM_MODULE_CMD(0x00)` | パラメータ一括設定 |
| `CMD_AUDIO_GET_PARAMS` | `MM_MODULE_CMD(0x01)` | パラメータ一括取得 |
| `CMD_AUDIO_SET_SAMPLERATE` | `MM_MODULE_CMD(0x02)` | サンプルレート設定 |
| `CMD_AUDIO_SET_ADC_GAIN` | `MM_MODULE_CMD(0x06)` | ADC ゲイン設定 |
| `CMD_AUDIO_SET_DAC_GAIN` | `MM_MODULE_CMD(0x07)` | DAC ゲイン設定 |
| `CMD_AUDIO_SET_NS_ENABLE` | `MM_MODULE_CMD(0x10)` | ノイズサプレス有効化 |
| `CMD_AUDIO_SET_AEC_ENABLE` | `MM_MODULE_CMD(0x11)` | エコーキャンセル有効化 |
| `CMD_AUDIO_SET_AGC_ENABLE` | `MM_MODULE_CMD(0x12)` | 自動ゲイン制御有効化 |
| `CMD_AUDIO_SET_VAD_ENABLE` | `MM_MODULE_CMD(0x13)` | 音声活動検出有効化 |
| `CMD_AUDIO_RUN_NS` | `MM_MODULE_CMD(0x14)` | NS 処理の実行 |
| `CMD_AUDIO_RUN_AEC` | `MM_MODULE_CMD(0x15)` | AEC 処理の実行 |
| `CMD_AUDIO_RUN_AGC` | `MM_MODULE_CMD(0x16)` | AGC 処理の実行 |
| `CMD_AUDIO_RUN_VAD` | `MM_MODULE_CMD(0x17)` | VAD 処理の実行 |
| `CMD_AUDIO_SET_MIC_ENABLE` | `MM_MODULE_CMD(0x19)` | マイク有効化 |
| `CMD_AUDIO_SET_SPK_ENABLE` | `MM_MODULE_CMD(0x1A)` | スピーカー有効化 |
| `CMD_AUDIO_SET_AVSYNC_TIMESTAMP` | `MM_MODULE_CMD(0x1F)` | AV 同期タイムスタンプ設定 |
| `CMD_AUDIO_SET_RXASP_PARAM` | `MM_MODULE_CMD(0x24)` | 受信 ASP パラメータ設定 |
| `CMD_AUDIO_SET_TXASP_PARAM` | `MM_MODULE_CMD(0x25)` | 送信 ASP パラメータ設定 |
| `CMD_AUDIO_APPLY` | `MM_MODULE_CMD(0x30)` | 設定を適用する |

#### パラメータ構造体 `audio_params_t`

| フィールド | 型 | 説明 |
|-----------|-----|------|
| `sample_rate` | `audio_sr` | サンプルレート (例: `ASR_8KHZ`) |
| `word_length` | `audio_wl` | ワード長 (例: `WL_16BIT`) |
| `mic_gain` | `audio_mic_gain` | マイクゲイン (例: `MIC_40DB`) |
| `dmic_l_gain` | `audio_dmic_gain` | 左 DMIC ゲイン |
| `dmic_r_gain` | `audio_dmic_gain` | 右 DMIC ゲイン |
| `channel` | `int` | チャンネル数 (デフォルト: 1) |
| `use_mic_type` | `uint8_t` | マイクタイプ (0: AMIC, 1: DMIC) |
| `mic_bias` | `int` | マイクバイアス (0:0.9V, 1:0.86V, 2:0.75V) |
| `hpf_set` | `int` | ハイパスフィルタ設定 (0〜7) |
| `mic_l_eq[5]` | `eq_cof_t` | 左マイク EQ 係数 |
| `mic_r_eq[5]` | `eq_cof_t` | 右マイク EQ 係数 |
| `spk_l_eq[5]` | `eq_cof_t` | スピーカー EQ 係数 |
| `ADC_gain` | `int` | ADC デジタルゲイン |
| `DAC_gain` | `int` | DAC デジタルゲイン |
| `enable_record` | `int` | 録音の有効化 |
| `avsync_en` | `uint8_t` | AV 同期の有効化 |

---

### 2.3 RTSP2 ストリーミングモジュール (`module_rtsp2.h`)

RTSP サーバー経由で映像・音声をネットワーク配信する。

#### モジュールインスタンス

```c
extern mm_module_t rtsp2_module;
```

#### 制御コマンド

| コマンド | 値 | 説明 |
|---------|-----|------|
| `CMD_RTSP2_SET_PARAMS` | `MM_MODULE_CMD(0x00)` | パラメータ一括設定 |
| `CMD_RTSP2_GET_PARAMS` | `MM_MODULE_CMD(0x01)` | パラメータ一括取得 |
| `CMD_RTSP2_SET_STREAMMING` | `MM_MODULE_CMD(0x02)` | ストリーミング ON/OFF |
| `CMD_RTSP2_SELECT_STREAM` | `MM_MODULE_CMD(0x03)` | ストリーム選択 |
| `CMD_RTSP2_SET_FRAMERATE` | `MM_MODULE_CMD(0x04)` | フレームレート設定 |
| `CMD_RTSP2_SET_BITRATE` | `MM_MODULE_CMD(0x05)` | ビットレート設定 |
| `CMD_RTSP2_SET_SAMPLERATE` | `MM_MODULE_CMD(0x06)` | 音声サンプルレート設定 |
| `CMD_RTSP2_SET_CHANNEL` | `MM_MODULE_CMD(0x07)` | 音声チャンネル設定 |
| `CMD_RTSP2_SET_SPS` | `MM_MODULE_CMD(0x08)` | SPS 設定 |
| `CMD_RTSP2_SET_PPS` | `MM_MODULE_CMD(0x09)` | PPS 設定 |
| `CMD_RTSP2_SET_CODEC` | `MM_MODULE_CMD(0x0b)` | コーデック設定 |
| `CMD_RTSP2_SET_PORT` | `MM_MODULE_CMD(0x0f)` | RTSP ポート番号設定 |
| `CMD_RTSP2_SET_START_CB` | `MM_MODULE_CMD(0x10)` | ストリーム開始コールバック設定 |
| `CMD_RTSP2_SET_STOP_CB` | `MM_MODULE_CMD(0x11)` | ストリーム停止コールバック設定 |
| `CMD_RTSP2_SET_PAUSE_CB` | `MM_MODULE_CMD(0x12)` | ストリーム一時停止コールバック設定 |
| `CMD_RTSP2_SET_CUSTOM_CB` | `MM_MODULE_CMD(0x13)` | カスタムコールバック設定 |
| `CMD_RTSP2_SET_DROP_TIME` | `MM_MODULE_CMD(0x14)` | ドロップタイム設定 |
| `CMD_RTSP2_SET_BLOCK_TYPE` | `MM_MODULE_CMD(0x15)` | ブロックタイプ設定 (NON_BLOCK / BLOCK) |
| `CMD_RTSP2_SET_SYNC_MODE` | `MM_MODULE_CMD(0x16)` | 同期モード設定 |
| `CMD_RTSP2_SET_URL` | `MM_MODULE_CMD(0x17)` | RTSP URL 設定 |
| `CMD_RTSP2_SET_INTERFACE` | `MM_MODULE_CMD(0x18)` | ネットワークインターフェース設定 (`RTSP_WIFI_STA` / `RTSP_ETHERNET`) |
| `CMD_RTSP2_SET_APPLY` | `MM_MODULE_CMD(0x0d)` | 設定を適用する |

#### パラメータ構造体 `rtsp2_params_t`

| フィールド | 型 | 説明 |
|-----------|-----|------|
| `type` | `uint32_t` | ストリームタイプ (映像 / 音声) |
| `u.v.codec_id` | `uint32_t` | 映像コーデック ID |
| `u.v.fps` | `uint32_t` | フレームレート |
| `u.v.bps` | `uint32_t` | ビットレート (bps) |
| `u.v.sps` / `u.v.pps` | `char *` | SPS/PPS データ |
| `u.a.codec_id` | `uint32_t` | 音声コーデック ID |
| `u.a.channel` | `uint32_t` | チャンネル数 |
| `u.a.samplerate` | `uint32_t` | サンプルレート |
| `u.a.max_average_bitrate` | `uint32_t` | Opus 用最大平均ビットレート |
| `u.a.frame_size` | `uint32_t` | Opus 用フレームサイズ |

---

### 2.4 RTP 受信モジュール (`module_rtp.h`)

RTP パケットを UDP で受信する。

#### モジュールインスタンス

```c
extern mm_module_t rtp_module;
```

#### 制御コマンド

| コマンド | 値 | 説明 |
|---------|-----|------|
| `CMD_RTP_SET_PARAMS` | `MM_MODULE_CMD(0x00)` | パラメータ一括設定 |
| `CMD_RTP_GET_PARAMS` | `MM_MODULE_CMD(0x01)` | パラメータ一括取得 |
| `CMD_RTP_STREAMING` | `MM_MODULE_CMD(0x02)` | ストリーミング制御 |
| `CMD_RTP_APPLY` | `MM_MODULE_CMD(0x20)` | 設定を適用する |

#### パラメータ構造体 `rtp_params_t`

| フィールド | 型 | 説明 |
|-----------|-----|------|
| `valid_pt` | `uint32_t` | 受け付ける RTP payload type (0xFFFFFFFF = 全許可) |
| `port` | `uint32_t` | 受信 UDP ポート番号 (デフォルト: 16384) |
| `frame_size` | `uint32_t` | フレームサイズ |
| `cache_depth` | `uint32_t` | キャッシュ深度 |

---

### 2.5 AAC エンコーダモジュール (`module_aac.h`)

PCM 音声データを AAC フォーマットにエンコードする。

#### モジュールインスタンス

```c
extern mm_module_t aac_module;
```

#### 転送タイプ列挙 (`AAC_TRANSPORT_TYPE`)

| 定数 | 説明 |
|------|------|
| `AAC_TYPE_RAW` | ADTS ヘッダなし RAW |
| `AAC_TYPE_ADTS` | ADTS ヘッダ付き |

#### オーディオオブジェクトタイプ列挙 (`AAC_AOT_TYPE`)

| 定数 | 説明 |
|------|------|
| `AAC_AOT_LC` | AAC-LC |
| `AAC_AOT_SBR` | HE-AAC v1 (LC + SBR) |
| `AAC_AOT_PS` | HE-AAC v2 (LC + SBR + PS) |
| `AAC_AOT_ER_LD` | Error Resilient Low Delay |
| `AAC_AOT_ER_ELD` | Enhanced Low Delay |

#### 制御コマンド

| コマンド | 値 | 説明 |
|---------|-----|------|
| `CMD_AAC_SET_PARAMS` | `MM_MODULE_CMD(0x00)` | パラメータ一括設定 |
| `CMD_AAC_GET_PARAMS` | `MM_MODULE_CMD(0x01)` | パラメータ一括取得 |
| `CMD_AAC_SAMPLERATE` | `MM_MODULE_CMD(0x02)` | サンプルレート設定 |
| `CMD_AAC_CHANNEL` | `MM_MODULE_CMD(0x03)` | チャンネル数設定 |
| `CMD_AAC_BITLENGTH` | `MM_MODULE_CMD(0x04)` | ビット長設定 |
| `CMD_AAC_MEMORY_SIZE` | `MM_MODULE_CMD(0x07)` | メモリサイズ設定 |
| `CMD_AAC_BLOCK_SIZE` | `MM_MODULE_CMD(0x08)` | ブロックサイズ設定 |
| `CMD_AAC_MAX_FRAME_SIZE` | `MM_MODULE_CMD(0x09)` | 最大フレームサイズ設定 |
| `CMD_AAC_INIT_MEM_POOL` | `MM_MODULE_CMD(0x0a)` | メモリプール初期化 |
| `CMD_AAC_RESET` | `MM_MODULE_CMD(0x0b)` | リセット |
| `CMD_AAC_STOP` | `MM_MODULE_CMD(0x0c)` | 停止 |
| `CMD_AAC_APPLY` | `MM_MODULE_CMD(0x20)` | 設定を適用する |

#### パラメータ構造体 `aac_params_t`

| フィールド | 型 | 説明 |
|-----------|-----|------|
| `trans_type` | `AAC_TRANSPORT_TYPE` | 転送タイプ |
| `object_type` | `AAC_AOT_TYPE` | オーディオオブジェクトタイプ |
| `sample_rate` | `uint32_t` | サンプルレート (例: 8000) |
| `channel` | `uint32_t` | チャンネル数 (例: 1) |
| `bitrate` | `uint32_t` | ビットレート |
| `mem_total_size` | `uint32_t` | メモリプール合計サイズ |
| `mem_block_size` | `uint32_t` | メモリブロックサイズ |
| `mem_frame_size` | `uint32_t` | フレームバッファサイズ |

---

### 2.6 AAC デコーダモジュール (`module_aad.h`)

AAC ストリームを PCM にデコードする。

#### モジュールインスタンス

```c
extern mm_module_t aad_module;
```

#### 転送タイプ列挙 (`AAD_TRANSPORT_TYPE`)

| 定数 | 値 | 説明 |
|------|-----|------|
| `AAD_TYPE_RAW` | `0` | AU-header なし RAW |
| `AAD_TYPE_ADTS` | `2` | ADTS ヘッダ付き |
| `AAD_TYPE_RTP_RAW` | `3` | RTP パケット由来 AU-header 付き |

#### 制御コマンド

| コマンド | 値 | 説明 |
|---------|-----|------|
| `CMD_AAD_SET_PARAMS` | `MM_MODULE_CMD(0x00)` | パラメータ一括設定 |
| `CMD_AAD_GET_PARAMS` | `MM_MODULE_CMD(0x01)` | パラメータ一括取得 |
| `CMD_AAD_SAMPLERATE` | `MM_MODULE_CMD(0x02)` | サンプルレート設定 |
| `CMD_AAD_CHANNEL` | `MM_MODULE_CMD(0x03)` | チャンネル数設定 |
| `CMD_AAD_TRANSPORT_TYPE` | `MM_MODULE_CMD(0x04)` | 転送タイプ設定 |
| `CMD_AAD_RESET` | `MM_MODULE_CMD(0x05)` | リセット |
| `CMD_AAD_STOP` | `MM_MODULE_CMD(0x06)` | 停止 |
| `CMD_AAD_APPLY` | `MM_MODULE_CMD(0x20)` | 設定を適用する |

---

### 2.7 G.711 コーデックモジュール (`module_g711.h`)

G.711 PCMA (A-law) / PCMU (µ-law) のエンコード・デコードを行う。

#### モジュールインスタンス

```c
extern mm_module_t g711_module;
```

#### コーデックモード定数

| 定数 | 値 | 説明 |
|------|----|------|
| `G711_ENCODE` | `0` | エンコードモード |
| `G711_DECODE` | `1` | デコードモード |

#### 制御コマンド

| コマンド | 値 | 説明 |
|---------|-----|------|
| `CMD_G711_SET_PARAMS` | `MM_MODULE_CMD(0x00)` | パラメータ一括設定 |
| `CMD_G711_GET_PARAMS` | `MM_MODULE_CMD(0x01)` | パラメータ一括取得 |
| `CMD_G711_CODECID` | `MM_MODULE_CMD(0x02)` | コーデック ID 設定 (`AV_CODEC_ID_PCMA` / `AV_CODEC_ID_PCMU`) |
| `CMD_G711_LENGTH` | `MM_MODULE_CMD(0x03)` | 出力バッファ長設定 |
| `CMD_G711_CODECMODE` | `MM_MODULE_CMD(0x04)` | コーデックモード設定 |
| `CMD_G711_APPLY` | `MM_MODULE_CMD(0x20)` | 設定を適用する |

---

### 2.8 I2S 音声インターフェースモジュール (`module_i2s.h`)

外部 I2S デバイスとの音声データ送受信を行う。

#### モジュールインスタンス

```c
extern mm_module_t i2s_module;
extern i2s_params_t default_i2s_params;
```

#### 関連列挙型

**`i2s_ws_trig_edge`** — WS トリガエッジ

| 定数 | 説明 |
|------|------|
| `WS_NEGATIVE_EDGE` | ネガティブエッジ |
| `WS_POSITIVE_EDGE` | ポジティブエッジ |

**`i2s_channel_type`** — チャンネルタイプ

| 定数 | 説明 |
|------|------|
| `I2S_LEFT_CHANNEL` | 左チャンネル |
| `I2S_RIGHT_CHANNEL` | 右チャンネル |
| `I2S_STEREO_CHANNEL` | ステレオ |

**`i2s_direction_type`** — 通信方向

| 定数 | 説明 |
|------|------|
| `I2S_RX_ONLY` | 受信のみ |
| `I2S_TX_ONLY` | 送信のみ |
| `I2S_TRX_BOTH` | 送受信両方 |

#### 制御コマンド

| コマンド | 値 | 説明 |
|---------|-----|------|
| `CMD_I2S_SET_PARAMS` | `MM_MODULE_CMD(0x00)` | パラメータ一括設定 |
| `CMD_I2S_GET_PARAMS` | `MM_MODULE_CMD(0x01)` | パラメータ一括取得 |
| `CMD_I2S_SET_SAMPLERATE` | `MM_MODULE_CMD(0x02)` | サンプルレート設定 |
| `CMD_I2S_SET_TRX` | `MM_MODULE_CMD(0x03)` | 送受信方向設定 |
| `CMD_I2S_SET_RX` | `MM_MODULE_CMD(0x04)` | 受信設定 |
| `CMD_I2S_SET_TX` | `MM_MODULE_CMD(0x05)` | 送信設定 |
| `CMD_I2S_SET_FORMAT` | `MM_MODULE_CMD(0x08)` | フォーマット設定 |
| `CMD_I2S_SET_ROLE` | `MM_MODULE_CMD(0x09)` | マスター/スレーブ設定 |
| `CMD_I2S_SET_DATA_EDGE` | `MM_MODULE_CMD(0x0A)` | データエッジ設定 |
| `CMD_I2S_SET_WS_EDGE` | `MM_MODULE_CMD(0x0B)` | WS エッジ設定 |
| `CMD_I2S_SET_TIMESTAMP_OFFSET` | `MM_MODULE_CMD(0x0C)` | タイムスタンプオフセット設定 |
| `CMD_I2S_APPLY` | `MM_MODULE_CMD(0x20)` | 設定を適用する |

---

### 2.9 Opus エンコーダモジュール (`module_opusc.h`)

PCM 音声を Opus フォーマットにエンコードする。

#### モジュールインスタンス

```c
extern mm_module_t opusc_module;
```

#### 制御コマンド

| コマンド | 値 | 説明 |
|---------|-----|------|
| `CMD_OPUSC_SET_PARAMS` | `MM_MODULE_CMD(0x00)` | パラメータ一括設定 |
| `CMD_OPUSC_GET_PARAMS` | `MM_MODULE_CMD(0x01)` | パラメータ一括取得 |
| `CMD_OPUSC_SAMPLERATE` | `MM_MODULE_CMD(0x02)` | サンプルレート設定 |
| `CMD_OPUSC_CHANNEL` | `MM_MODULE_CMD(0x03)` | チャンネル数設定 |
| `CMD_OPUSC_BITLENGTH` | `MM_MODULE_CMD(0x04)` | ビット長設定 |
| `CMD_OPUSC_RESET` | `MM_MODULE_CMD(0x0b)` | リセット |
| `CMD_OPUSC_STOP` | `MM_MODULE_CMD(0x0c)` | 停止 |
| `CMD_OPUSC_SET_PADDING_SIZE` | `MM_MODULE_CMD(0x0d)` | パディングサイズ設定 |
| `CMD_OPUSC_APPLY` | `MM_MODULE_CMD(0x20)` | 設定を適用する |

#### パラメータ構造体 `opusc_params_t`

| フィールド | 型 | 説明 |
|-----------|-----|------|
| `sample_rate` | `uint32_t` | サンプルレート (例: 8000) |
| `channel` | `uint32_t` | チャンネル数 |
| `bit_length` | `uint32_t` | ビット長 (例: 16) |
| `complexity` | `uint32_t` | エンコード複雑度 |
| `use_framesize` | `uint32_t` | フレームサイズ |
| `bitrate` | `uint32_t` | ビットレート (デフォルト: 25000) |
| `enable_vbr` | `uint32_t` | VBR 有効フラグ |
| `vbr_constraint` | `uint32_t` | VBR 制約フラグ |
| `packetLossPercentage` | `uint32_t` | パケットロス率 |
| `opus_application` | `uint32_t` | アプリケーション種別 |

---

### 2.10 Opus デコーダモジュール (`module_opusd.h`)

Opus ストリームを PCM にデコードする。

#### モジュールインスタンス

```c
extern mm_module_t opusd_module;
```

#### 制御コマンド

| コマンド | 値 | 説明 |
|---------|-----|------|
| `CMD_OPUSD_SET_PARAMS` | `MM_MODULE_CMD(0x00)` | パラメータ一括設定 |
| `CMD_OPUSD_GET_PARAMS` | `MM_MODULE_CMD(0x01)` | パラメータ一括取得 |
| `CMD_OPUSD_SAMPLERATE` | `MM_MODULE_CMD(0x02)` | サンプルレート設定 |
| `CMD_OPUSD_CHANNEL` | `MM_MODULE_CMD(0x03)` | チャンネル数設定 |
| `CMD_OPUSD_STREAM_TYPE` | `MM_MODULE_CMD(0x04)` | ストリームタイプ設定 |
| `CMD_OPUSD_RESET` | `MM_MODULE_CMD(0x05)` | リセット |
| `CMD_OPUSD_STOP` | `MM_MODULE_CMD(0x06)` | 停止 |
| `CMD_OPUSD_APPLY` | `MM_MODULE_CMD(0x20)` | 設定を適用する |

---

### 2.11 MP4 録画モジュール (`module_mp4.h`)

映像・音声ストリームを SD カードに MP4 形式で録画する。

#### モジュールインスタンス

```c
extern mm_module_t mp4_module;
```

#### 制御コマンド

| コマンド | 値 | 説明 |
|---------|-----|------|
| `CMD_MP4_SET_PARAMS` | `MM_MODULE_CMD(0x00)` | パラメータ一括設定 |
| `CMD_MP4_GET_PARAMS` | `MM_MODULE_CMD(0x01)` | パラメータ一括取得 |
| `CMD_MP4_SET_HEIGHT` | `MM_MODULE_CMD(0x02)` | 映像高さ設定 |
| `CMD_MP4_SET_WIDTH` | `MM_MODULE_CMD(0x03)` | 映像幅設定 |
| `CMD_MP4_SET_FPS` | `MM_MODULE_CMD(0x04)` | フレームレート設定 |
| `CMD_MP4_SET_RECORD_LENGTH` | `MM_MODULE_CMD(0x08)` | 録画時間設定 |
| `CMD_MP4_GET_RECORD_LENGTH` | `MM_MODULE_CMD(0x09)` | 録画時間取得 |
| `CMD_MP4_SET_RECORD_TYPE` | `MM_MODULE_CMD(0x0a)` | 録画タイプ設定 |
| `CMD_MP4_SET_RECORD_FILE_NAME` | `MM_MODULE_CMD(0x0c)` | ファイル名設定 |
| `CMD_MP4_START` | `MM_MODULE_CMD(0x10)` | 録画開始 |
| `CMD_MP4_STOP` | `MM_MODULE_CMD(0x11)` | 録画停止 |
| `CMD_MP4_STOP_IMMEDIATELY` | `MM_MODULE_CMD(0x12)` | 録画即時停止 |
| `CMD_MP4_GET_STATUS` | `MM_MODULE_CMD(0x13)` | 録画状態取得 |
| `CMD_MP4_LOOP_MODE` | `MM_MODULE_CMD(0x17)` | ループ録画モード設定 |
| `CMD_MP4_SET_STOP_CB` | `MM_MODULE_CMD(0x15)` | 各ループ終了コールバック設定 |
| `CMD_MP4_SET_END_CB` | `MM_MODULE_CMD(0x16)` | 最終ループ終了コールバック設定 |
| `CMD_MP4_SET_ERROR_CB` | `MM_MODULE_CMD(0x18)` | エラーコールバック設定 |
| `CMD_MP4_SET_UDAT_CALLBACK` | `MM_MODULE_CMD(0x1d)` | ユーザーデータコールバック設定 |
| `CMD_MP4_SET_TIMELAPSE_PARAMS` | `MM_MODULE_CMD(0x1e)` | タイムラプスパラメータ設定 |

---

### 2.12 フラグメント MP4 モジュール (`module_fmp4.h`)

映像・音声データをフラグメント MP4 (fMP4) 形式でファイルに書き出す。

#### モジュールインスタンス

```c
extern mm_module_t fmp4_module;
```

#### 制御コマンド

| コマンド | 値 | 説明 |
|---------|-----|------|
| `CMD_FMP4_SET_WIDTH` | `MM_MODULE_CMD(0x00)` | 映像幅設定 |
| `CMD_FMP4_SET_HEIGHT` | `MM_MODULE_CMD(0x01)` | 映像高さ設定 |
| `CMD_FMP4_SET_FILENAME` | `MM_MODULE_CMD(0x02)` | ファイル名設定 |
| `CMD_FMP4_FILE_OPEN` | `MM_MODULE_CMD(0x03)` | ファイルオープン |
| `CMD_FMP4_FILE_CLOSE` | `MM_MODULE_CMD(0x04)` | ファイルクローズ |
| `CMD_FMP4_APPLY` | `MM_MODULE_CMD(0x05)` | 設定を適用する |

---

### 2.13 MP4 デマルチプレクサモジュール (`module_demuxer.h`)

SD カード上の MP4 ファイルを映像・音声ストリームに分解して出力する。

#### モジュールインスタンス

```c
extern mm_module_t demuxer_module;
```

#### 状態定数

| 定数 | 値 | 説明 |
|------|----|------|
| `DEMUXER_IDLE` | `0x00` | アイドル |
| `DEMUXER_OPEN` | `0x01` | オープン済み |
| `DEMUXER_START` | `0x02` | 再生中 |
| `DEMUXER_PAUSE` | `0x04` | 一時停止中 |
| `DEMUXER_END` | `0x08` | 再生終了 |
| `DEMUXER_STOP` | `0x10` | 停止 |

#### ストリームタイプ定数

| 定数 | 値 | 説明 |
|------|----|------|
| `STREAM_AUDIO` | `0x01` | 音声ストリーム |
| `STREAM_VIDEO` | `0x10` | 映像ストリーム |
| `STREAM_ALL` | `0x11` | 映像・音声両方 |

#### 制御コマンド

| コマンド | 値 | 説明 |
|---------|-----|------|
| `CMD_DEMUXER_SET_PARAMS` | `MM_MODULE_CMD(0x00)` | パラメータ一括設定 |
| `CMD_DEMUXER_SET_FILE_NAME` | `MM_MODULE_CMD(0x03)` | ファイル名設定 |
| `CMD_DEMUXER_OPEN` | `MM_MODULE_CMD(0x07)` | ファイルオープン |
| `CMD_DEMUXER_CLOSE` | `MM_MODULE_CMD(0x08)` | ファイルクローズ |
| `CMD_DEMUXER_STREAM_PAUSE` | `MM_MODULE_CMD(0x09)` | ストリーム一時停止 |
| `CMD_DEMUXER_STREAM_RESUME` | `MM_MODULE_CMD(0x10)` | ストリーム再開 |
| `CMD_DEMUXER_STREAM_START` | `MM_MODULE_CMD(0x11)` | ストリーム開始 |
| `CMD_DEMUXER_STOP` | `MM_MODULE_CMD(0x12)` | 停止 |
| `CMD_DEMUXER_GET_STATUS` | `MM_MODULE_CMD(0x13)` | 状態取得 |
| `CMD_DEMUXER_APPLY` | `MM_MODULE_CMD(0x20)` | 設定を適用する |

---

### 2.14 キューバッファモジュール (`module_queue.h`)

映像・音声の非同期バッファリングに使用するキュー。

#### モジュールインスタンス

```c
extern mm_module_t queue_module;
```

#### 制御コマンド

| コマンド | 値 | 説明 |
|---------|-----|------|
| `CMD_QUEUE_SET_VQUEUE_LEN` | `MM_MODULE_CMD(0x00)` | 映像キュー長設定 |
| `CMD_QUEUE_SET_AQUEUE_LEN` | `MM_MODULE_CMD(0x01)` | 音声キュー長設定 |

---

### 2.15 配列データソースモジュール (`module_array.h`)

メモリ上の配列データを映像・音声ソースとしてパイプラインに流す。テスト・再生用途。

#### モジュールインスタンス

```c
extern mm_module_t array_module;
```

#### モード定数

| 定数 | 値 | 説明 |
|------|----|------|
| `ARRAY_MODE_ONCE` | `0` | 1 回再生 |
| `ARRAY_MODE_LOOP` | `1` | ループ再生 |

#### 制御コマンド

| コマンド | 値 | 説明 |
|---------|-----|------|
| `CMD_ARRAY_SET_PARAMS` | `MM_MODULE_CMD(0x00)` | パラメータ一括設定 |
| `CMD_ARRAY_GET_PARAMS` | `MM_MODULE_CMD(0x01)` | パラメータ一括取得 |
| `CMD_ARRAY_SET_ARRAY` | `MM_MODULE_CMD(0x02)` | 配列データ設定 |
| `CMD_ARRAY_SET_MODE` | `MM_MODULE_CMD(0x03)` | 再生モード設定 |
| `CMD_ARRAY_GET_STATE` | `MM_MODULE_CMD(0x04)` | 状態取得 |
| `CMD_ARRAY_STREAMING` | `MM_MODULE_CMD(0x05)` | ストリーミング制御 |
| `CMD_ARRAY_APPLY` | `MM_MODULE_CMD(0x20)` | 設定を適用する |

---

### 2.16 ファイルローダーモジュール (`module_fileloader.h`)

SD カード・TFTP・FTP から画像・音声ファイルを読み込み、パイプラインに供給する。

#### モジュールインスタンス

```c
extern mm_module_t fileloader_module;
```

#### 読み込みモード列挙 (`read_mode_t`)

| 定数 | 説明 |
|------|------|
| `SEQUENCE_MODE` | 番号順にファイルを読み込む |
| `FILELIST_MODE` | テキストリストに記載のファイルを順に読み込む |

#### メディアアクセスモード定数

| 定数 | 値 | 説明 |
|------|----|------|
| `MA_SD` | `0` | SD カード |
| `MA_TFTP` | `1` | TFTP |
| `MA_FTP` | `2` | FTP |

#### 制御コマンド

| コマンド | 値 | 説明 |
|---------|-----|------|
| `CMD_FILELOADER_SET_PARAMS` | `MM_MODULE_CMD(0x00)` | パラメータ一括設定 |
| `CMD_FILELOADER_SET_TEST_FILE_PATH` | `MM_MODULE_CMD(0x02)` | テストファイルパス設定 |
| `CMD_FILELOADER_SET_FILE_NUM` | `MM_MODULE_CMD(0x03)` | テストファイル数設定 |
| `CMD_FILELOADER_SET_DECODE_PROCESS` | `MM_MODULE_CMD(0x04)` | デコード前処理関数設定 |
| `CMD_FILELOADER_SET_READ_MODE` | `MM_MODULE_CMD(0x05)` | 読み込みモード設定 |
| `CMD_FILELOADER_SET_FILELIST_NAME` | `MM_MODULE_CMD(0x06)` | ファイルリスト名設定 |
| `CMD_FILELOADER_SET_TFTP_MODE` | `MM_MODULE_CMD(0x10)` | TFTP モード設定 |
| `CMD_FILELOADER_SET_FTP_MODE` | `MM_MODULE_CMD(0x11)` | FTP モード設定 |
| `CMD_FILELOADER_SET_REMOTE_IP` | `MM_MODULE_CMD(0x18)` | リモート IP アドレス設定 |
| `CMD_FILELOADER_SET_REMOTE_PORT` | `MM_MODULE_CMD(0x19)` | リモートポート設定 |
| `CMD_FILELOADER_SET_REMOTE_USER` | `MM_MODULE_CMD(0x1a)` | リモートユーザー設定 |
| `CMD_FILELOADER_SET_REMOTE_PASS` | `MM_MODULE_CMD(0x1b)` | リモートパスワード設定 |
| `CMD_FILELOADER_SET_REMOTE_DIR` | `MM_MODULE_CMD(0x1c)` | リモートディレクトリ設定 |
| `CMD_FILELOADER_GET_LOADED_FILE_COUNT` | `MM_MODULE_CMD(0x1f)` | 読み込み済みファイル数取得 |
| `CMD_FILELOADER_APPLY` | `MM_MODULE_CMD(0x20)` | 設定を適用する |

---

### 2.17 ファイルセーバーモジュール (`module_filesaver.h`)

パイプラインから受け取ったデータをファイルに保存する。

#### モジュールインスタンス

```c
extern mm_module_t filesaver_module;
```

#### 制御コマンド

| コマンド | 値 | 説明 |
|---------|-----|------|
| `CMD_FILESAVER_SET_PARAMS` | `MM_MODULE_CMD(0x00)` | パラメータ一括設定 |
| `CMD_FILESAVER_GET_PARAMS` | `MM_MODULE_CMD(0x01)` | パラメータ一括取得 |
| `CMD_FILESAVER_SET_SAVE_FILE_PATH` | `MM_MODULE_CMD(0x02)` | 保存ファイルパス設定 |
| `CMD_FILESAVER_SET_TYPE_HANDLER` | `MM_MODULE_CMD(0x10)` | タイプ別ハンドラ関数設定 (`type_handler_t`) |
| `CMD_FILESAVER_APPLY` | `MM_MODULE_CMD(0x20)` | 設定を適用する |

---

### 2.18 モーション検知モジュール (`module_md.h`)

ISP から取得した輝度データを用いてフレーム間差分によるモーション検知を行う。

#### モジュールインスタンス

```c
extern mm_module_t md_module;
```

#### ステータス定数

| 定数 | 値 | 説明 |
|------|----|------|
| `MD_STATUS_START` | `0x00` | 動作中 |
| `MD_STATUS_STOP` | `0x01` | 停止中 |
| `MD_SET_STOP` | `0x02` | 停止要求 |

#### 制御コマンド

| コマンド | 値 | 説明 |
|---------|-----|------|
| `CMD_MD_SET_PARAMS` | `MM_MODULE_CMD(0x00)` | パラメータ設定 |
| `CMD_MD_SET_MD_CONFIG` | `MM_MODULE_CMD(0x01)` | MD 設定 |
| `CMD_MD_GET_MD_CONFIG` | `MM_MODULE_CMD(0x02)` | MD 設定取得 |
| `CMD_MD_SET_MD_MASK` | `MM_MODULE_CMD(0x03)` | 検知マスク設定 |
| `CMD_MD_GET_MD_MASK` | `MM_MODULE_CMD(0x04)` | 検知マスク取得 |
| `CMD_MD_GET_MD_RESULT` | `MM_MODULE_CMD(0x05)` | 検知結果取得 |
| `CMD_MD_SET_OUTPUT` | `MM_MODULE_CMD(0x06)` | 出力有効化 |
| `CMD_MD_SET_DISPPOST` | `MM_MODULE_CMD(0x07)` | 表示後処理コールバック設定 |
| `CMD_MD_SET_TRIG_BLK` | `MM_MODULE_CMD(0x08)` | トリガブロック設定 |
| `CMD_MD_EN_AE_STABLE` | `MM_MODULE_CMD(0x09)` | AE 安定化の有効化 |
| `CMD_MD_SET_ADAPT_THR_MODE` | `MM_MODULE_CMD(0x0A)` | 適応閾値モード設定 |
| `CMD_MD_SET_TIME_FILTER_INTERVAL` | `MM_MODULE_CMD(0x0B)` | 時間フィルタ間隔設定 |
| `CMD_MD_SET_DETECT_INTERVAL` | `MM_MODULE_CMD(0x0C)` | 検知間隔設定 |
| `CMD_MD_SET_BGMODE` | `MM_MODULE_CMD(0x0D)` | 背景モード設定 |
| `CMD_MD_SET_STATUS` | `MM_MODULE_CMD(0x0E)` | ステータス設定 |
| `CMD_MD_SET_MD_SENSITIVITY` | `MM_MODULE_CMD(0x10)` | 検知感度設定 |
| `CMD_MD_GET_MD_SENSITIVITY` | `MM_MODULE_CMD(0x11)` | 検知感度取得 |

---

### 2.19 拡張画像処理モジュール (`module_eip.h`)

モーション検知 (MD) と自動 WDR (Wide Dynamic Range) を統合して処理する。

#### モジュールインスタンス

```c
extern mm_module_t eip_module;
```

#### 制御コマンド

| コマンド | 値 | 説明 |
|---------|-----|------|
| `CMD_EIP_SET_PARAMS` | `MM_MODULE_CMD(0x00)` | パラメータ設定 |
| `CMD_EIP_SET_CONFIG` | `MM_MODULE_CMD(0x01)` | EIP 設定 |
| `CMD_EIP_GET_CONFIG` | `MM_MODULE_CMD(0x02)` | EIP 設定取得 |
| `CMD_EIP_SET_STATUS` | `MM_MODULE_CMD(0x03)` | ステータス設定 |
| `CMD_EIP_GET_STATIS_INFO` | `MM_MODULE_CMD(0x04)` | 統計情報取得 |
| `CMD_EIP_AE_STABLE_EN` | `MM_MODULE_CMD(0x06)` | AE 安定化の有効化 |
| `CMD_EIP_SET_MD_EN` | `MM_MODULE_CMD(0x10)` | MD 有効化 |
| `CMD_EIP_SET_MD_CONFIG` | `MM_MODULE_CMD(0x11)` | MD 設定 |
| `CMD_EIP_SET_MD_SENSITIVITY` | `MM_MODULE_CMD(0x15)` | MD 感度設定 |
| `CMD_EIP_GET_MD_RESULT` | `MM_MODULE_CMD(0x17)` | MD 結果取得 |
| `CMD_EIP_SET_AUTO_WDR_EN` | `MM_MODULE_CMD(0x20)` | 自動 WDR 有効化 |
| `CMD_EIP_SET_AUTO_WDR_CONFIG` | `MM_MODULE_CMD(0x21)` | 自動 WDR 設定 |
| `CMD_EIP_GET_AUTO_WDR_CONFIG` | `MM_MODULE_CMD(0x22)` | 自動 WDR 設定取得 |

#### `eip_config_t` 構造体

| フィールド | 型 | 説明 |
|-----------|-----|------|
| `en_ae_stable` | `int` | AE 安定化の有効フラグ |
| `en_md` | `int` | モーション検知の有効フラグ |
| `en_auto_wdr` | `int` | 自動 WDR の有効フラグ |

---

### 2.20 VIP ニューラルネット推論モジュール (`module_vipnn.h`)

VIP (Vision Intelligence Processor) を用いた NN モデルによる画像・音声推論を行う。

#### モジュールインスタンス

```c
extern mm_module_t vipnn_module;
```

#### 制御コマンド

| コマンド | 値 | 説明 |
|---------|-----|------|
| `CMD_VIPNN_SET_MODEL` | `MM_MODULE_CMD(0x00)` | モデル設定 |
| `CMD_VIPNN_SET_IN_PARAMS` | `MM_MODULE_CMD(0x01)` | 入力パラメータ設定 |
| `CMD_VIPNN_SET_DISPPOST` | `MM_MODULE_CMD(0x03)` | 表示後処理コールバック設定 |
| `CMD_VIPNN_GET_STATUS` | `MM_MODULE_CMD(0x04)` | ステータス取得 |
| `CMD_VIPNN_SET_CONFIDENCE_THRES` | `MM_MODULE_CMD(0x06)` | 信頼度閾値設定 |
| `CMD_VIPNN_SET_NMS_THRES` | `MM_MODULE_CMD(0x07)` | NMS 閾値設定 |
| `CMD_VIPNN_SET_DESIRED_CLASS` | `MM_MODULE_CMD(0x08)` | 検出クラス設定 |
| `CMD_VIPNN_SET_OUTPUT` | `MM_MODULE_CMD(0x15)` | 出力有効化 |
| `CMD_VIPNN_SET_OUTPUT_TYPE` | `MM_MODULE_CMD(0x16)` | 出力タイプ設定 (`VIPNN_NORMAL_OUTPUT` / `VIPNN_RAW_OUTPUT`) |
| `CMD_VIPNN_SET_CASCADE` | `MM_MODULE_CMD(0x17)` | カスケードモード設定 |
| `CMD_VIPNN_SET_RES_SIZE` | `MM_MODULE_CMD(0x18)` | 結果構造体サイズ設定 |
| `CMD_VIPNN_SET_RES_MAX_CNT` | `MM_MODULE_CMD(0x19)` | 最大結果数設定 |
| `CMD_VIPNN_SET_USR_OUTPUT_BUF` | `MM_MODULE_CMD(0x1B)` | ユーザー定義出力バッファ設定 |
| `CMD_VIPNN_APPLY` | `MM_MODULE_CMD(0x20)` | 設定を適用する |

#### モデル定義構造体 `nnmodel_t`

| フィールド | 型 | 説明 |
|-----------|-----|------|
| `nb` | `nn_get_nb_t` | ネットワークバイナリ取得コールバック |
| `nb_size` | `nn_get_nb_size_t` | ネットワークバイナリサイズ取得コールバック |
| `preprocess` | `nn_preprocess_t` | 前処理コールバック |
| `postprocess` | `nn_postprocess_t` | 後処理コールバック |
| `freemodel` | `nn_free_model_t` | モデル解放コールバック |
| `model_src` | `int` | モデルソース (`MODEL_SRC_MEM` / `MODEL_SRC_FILE`) |
| `set_init_info` | `nn_set_init_info_t` | 初期化情報設定コールバック |
| `set_confidence_thresh` | `nn_set_confidence_thresh_t` | 信頼度閾値設定コールバック |
| `set_nms_thresh` | `nn_set_nms_thresh_t` | NMS 閾値設定コールバック |
| `set_desired_class` | `nn_set_desired_class_t` | 検出クラス設定コールバック |
| `release` | `nn_release_t` | リソース解放コールバック |
| `name` | `const char *` | モデル名 |

#### 主な結果構造体

| 構造体 | 説明 |
|--------|------|
| `objdetect_res_t` | 物体検出結果 (クラス・スコア・バウンディングボックス) |
| `facedetect_res_t` | 顔検出結果 (バウンディングボックス・ランドマーク) |
| `face_feature_res_t` | 顔特徴量 (128 次元) |
| `yamnet_res_t` / `classification_res_t` | 音声分類結果 |
| `palmdetect_res_t` | 手のひら検出結果 |
| `handland_res_t` | 手のランドマーク (21 点 3D) |

---

### 2.21 顔認識モジュール (`module_facerecog.h`)

`module_vipnn` と連携し、顔の登録・照合を行う。

#### モジュールインスタンス

```c
extern mm_module_t facerecog_module;
```

#### 動作モード (`frc_mode_t`)

| 定数 | 説明 |
|------|------|
| `FRC_RECOGNITION` | 認識モード |
| `FRC_REGISTER` | 登録モード |

#### 制御コマンド

| コマンド | 値 | 説明 |
|---------|-----|------|
| `CMD_FRC_SET_PARAMS` | `MM_MODULE_CMD(0x00)` | パラメータ設定 |
| `CMD_FRC_SET_THRES100` | `MM_MODULE_CMD(0x01)` | 類似度閾値設定 (0〜100) |
| `CMD_FRC_SET_OSD_DRAW` | `MM_MODULE_CMD(0x04)` | OSD 描画コールバック設定 |
| `CMD_FRC_REGISTER_MODE` | `MM_MODULE_CMD(0x10)` | 登録モードに切り替え |
| `CMD_FRC_RECOGNITION_MODE` | `MM_MODULE_CMD(0x11)` | 認識モードに切り替え |
| `CMD_FRC_LOAD_FEATURES` | `MM_MODULE_CMD(0x12)` | 顔特徴量をロード |
| `CMD_FRC_SAVE_FEATURES` | `MM_MODULE_CMD(0x13)` | 顔特徴量をセーブ |
| `CMD_FRC_RESET_FEATURES` | `MM_MODULE_CMD(0x14)` | 顔特徴量をリセット |

---

### 2.22 HTTP ファイルサーバーモジュール (`module_httpfs.h`)

SD カード上のファイルを HTTP 経由で配信する。

#### モジュールインスタンス

```c
extern mm_module_t httpfs_module;
```

#### 制御コマンド

| コマンド | 値 | 説明 |
|---------|-----|------|
| `CMD_HTTPFS_SET_PARAMS` | `MM_MODULE_CMD(0x00)` | パラメータ一括設定 |
| `CMD_HTTPFS_GET_PARAMS` | `MM_MODULE_CMD(0x01)` | パラメータ一括取得 |
| `CMD_HTTPFS_APPLY` | `MM_MODULE_CMD(0x20)` | 設定を適用する |
| `CMD_HTTPFS_STREAM_STOP` | `MM_MODULE_CMD(0x23)` | ストリーム停止 |
| `CMD_HTTPFS_SET_RESPONSE_CB` | `MM_MODULE_CMD(0x14)` | HTTP レスポンスコールバック設定 |

#### パラメータ構造体 `httpfs_params_t`

| フィールド | 型 | 説明 |
|-----------|-----|------|
| `fileext[4]` | `char` | 配信対象の拡張子 |
| `filedir[32]` | `char` | ファイルディレクトリ |
| `request_string[128]` | `char` | HTTP リクエスト文字列 |
| `fatfs_buf_size` | `uint32_t` | FATFS バッファサイズ |

---

### 2.23 USB UVC デバイスモジュール (`module_uvcd.h`)

USB Video Class (UVC) デバイスとして映像を USB ホストに出力する。

#### モジュールインスタンス

```c
extern mm_module_t uvcd_module;
```

#### 制御コマンド

| コマンド | 値 | 説明 |
|---------|-----|------|
| `CMD_UVCD_CALLBACK_SET` | `MM_MODULE_CMD(0x00)` | コールバック設定 |
| `CMD_UVCD_CALLBACK_GET` | `MM_MODULE_CMD(0x01)` | コールバック取得 |
| `CMD_UVCD_STOP` | `MM_MODULE_CMD(0x02)` | 停止 |

#### `uvc_format` 構造体

| フィールド | 型 | 説明 |
|-----------|-----|------|
| `width` | `int` | 映像幅 |
| `height` | `int` | 映像高さ |
| `format` | `int` | 映像フォーマット |
| `fps` | `int` | フレームレート |
| `isp_format` | `int` | ISP フォーマット (1:YUV420, 2:YUV422, 3:Bayer) |
| `ldc` | `int` | レンズ歪み補正 (0:無効, 1:有効) |
| `bayer_type` | `int` | Bayer タイプ (1〜4: 各 ISP 処理ステージ) |

---

### 2.24 WebSocket ビューアモジュール (`module_web_viewer.h`)

WebSocket を使ってブラウザ向けにリアルタイム映像配信を行う。

#### モジュールインスタンス

```c
extern mm_module_t websocket_viewer_module;
```

#### 制御コマンド

| コマンド | 値 | 説明 |
|---------|-----|------|
| `CMD_WEB_VIEWER_APPLY` | `MM_MODULE_CMD(0x00)` | WebSocket ビューア起動 |
| `CMD_WEB_VIEWER_SET_BUF` | `MM_MODULE_CMD(0x01)` | バッファポインタ設定 |
| `CMD_WEB_VIEWER_SET_LEN` | `MM_MODULE_CMD(0x02)` | バッファ長設定 |

---

### 2.25 データ複製モジュール (`module_dup.h`)

入力データをそのままコピーして出力する。パイプライン内のデータ複製用。

#### モジュールインスタンス

```c
extern mm_module_t dup_module;
```

#### 制御コマンド

| コマンド | 値 | 説明 |
|---------|-----|------|
| `CMD_DUP_SET_PARAMS` | `MM_MODULE_CMD(0x00)` | パラメータ一括設定 |
| `CMD_DUP_GET_PARAMS` | `MM_MODULE_CMD(0x01)` | パラメータ一括取得 |
| `CMD_DUP_CODECID` | `MM_MODULE_CMD(0x02)` | コーデック ID 設定 |
| `CMD_DUP_LENGTH` | `MM_MODULE_CMD(0x03)` | 出力バッファ長設定 |
| `CMD_DUP_APPLY` | `MM_MODULE_CMD(0x20)` | 設定を適用する |

---

## 3. パイプライン構成例

### 映像 RTSP ストリーミング (SISO)

```
[video_module] --SISO--> [rtsp2_module]
```

1. `mm_module_open(&video_module)` で映像モジュールを生成
2. `mm_module_open(&rtsp2_module)` で RTSP モジュールを生成
3. `siso_create()` で SISO リンカーを生成
4. `siso_ctrl(siso, MMIC_CMD_ADD_INPUT, (uint32_t)video_ctx, ...)` で入力設定
5. `siso_ctrl(siso, MMIC_CMD_ADD_OUTPUT, (uint32_t)rtsp_ctx, ...)` で出力設定
6. `siso_start(siso)` でストリーミング開始

### 映像 + 音声 RTSP ストリーミング (MISO)

```
[video_module] --\
                  MISO--> [rtsp2_module]
[audio_module] --/
```

### 映像配信 + AI 推論 (SIMO)

```
                  /--> [rtsp2_module]
[video_module] --SIMO
                  \--> [vipnn_module]
```

---

## 4. モジュールコマンド番号体系

全モジュール共通の `MM_MODULE_CMD(x)` マクロは、ベース値 `0x80` に `x` を加算した値を生成します。

```c
#define MM_CMD_MODULE_BASE   0x80
#define MM_MODULE_CMD(x)     (MM_CMD_MODULE_BASE + (x))
```

例: `CMD_VIDEO_SET_PARAMS = MM_MODULE_CMD(0x00) = 0x80`

---

## 5. 依存関係

| コンポーネント | 説明 |
|--------------|------|
| FreeRTOS | タスク・キュー・セマフォ管理 |
| `video_api.h` | ISP / VOE 映像 API |
| `audio_api.h` | 内蔵 ADC/DAC 音声 API |
| `ASP.h` | 音声信号処理 (AEC/NS/AGC/VAD) |
| `rtsp/rtsp_api.h` | RTSP サーバー API |
| `rtsp/rtp_api.h` | RTP API |
| `aacenc_lib.h` | FDK-AAC エンコーダライブラリ |
| `aacdecoder_lib.h` | FDK-AAC デコーダライブラリ |
| `opus.h` | Opus コーデックライブラリ |
| `i2s_api.h` | I2S ハードウェア API |
| `mp4_muxer.h` | MP4 マルチプレクサ |
| `mp4_demuxer.h` | MP4 デマルチプレクサ |
| `fmp4-writer.h` | fMP4 ライタ |
| `vip_lite.h` | VIP NN 推論ライブラリ |
| `md_api.h` | モーション検知 API |
| `eip_api.h` | 拡張画像処理 API |
| `fatfs_sdcard_api.h` | SD カード FatFS API |
| `web_service.h` | WebSocket サービス |

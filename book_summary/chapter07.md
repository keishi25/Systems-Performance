# Chapter 7: Memory まとめ

## 7.1 Terminology（用語）

| 用語 | 意味 |
|---|---|
| **Main memory** | 物理メモリ。DRAMで提供される高速なデータ保存領域 |
| **Virtual memory** | メインメモリの抽象化。ほぼ無限で競合なし。仮想メモリは実際のメモリではない |
| **Resident memory** | 現在メインメモリに存在するメモリ（RSS） |
| **Anonymous memory** | ファイルシステム上にパスを持たないメモリ。プロセスのヒープなど |
| **Address space** | メモリコンテキスト。プロセスごと・カーネルごとに存在する |
| **Segment** | 特定の目的にフラグされた仮想メモリ領域（例：実行可能ページ・書き込み可能ページ） |
| **OOM** | Out of Memory。カーネルが利用可能メモリ不足を検知した状態 |
| **Page** | メモリの単位。歴史的に4KB or 8KB。現代のプロセッサは複数サイズをサポート |
| **Page fault** | オンデマンド仮想メモリ使用時に発生する無効なメモリアクセス（正常な現象） |
| **Paging** | メインメモリとストレージデバイス間のページ転送 |
| **Swapping（Linux）** | スワップデバイスへのanonymous paging（Linuxでの定義） |
| **Swap** | ページングされたanonymousデータ用のオンディスク領域 |

---

## 7.2 Concepts（概念）

### 仮想メモリの状態

仮想メモリのページは以下の4状態のいずれかにある：

| 状態 | 意味 |
|---|---|
| **A. Unallocated** | 未割り当て |
| **B. Allocated, unmapped** | 割り当て済みだが未マッピング（まだフォルトしていない） |
| **C. Allocated, in RAM** | 割り当て済みで物理メモリにマッピング済み |
| **D. Allocated, in swap** | 割り当て済みでスワップデバイスにマッピング済み |

- **RSS** (Resident Set Size) = C の合計サイズ
- **VSZ** (Virtual memory Size) = B + C + D の合計サイズ

### 7.2.2 Paging（ページング）

1962年のAtlas Computerで初めて導入。ページ単位（~4KB）でのメモリ管理。

**File System Paging（ファイルシステムページング）**
- mmap(2)やページキャッシュを使うアプリによって発生
- cleanページ（未変更）はディスクに書かずに解放できる → **"good" paging**
- dirtyページ（変更済み）は書き戻しが必要

**Anonymous Paging（Swapping）**
- プロセスのヒープ・スタックなどprivateなデータが対象
- 必ずスワップデバイスへの読み書きが発生 → **"bad" paging**
- スワップアウトしたページへのアクセスはディスクI/Oをブロック

### 7.2.4 Overcommit（オーバーコミット）

- Linuxは物理メモリ＋スワップの合計を超えるメモリ割り当てを許可
- `malloc(3)` などが通常は失敗する状況でも成功する
- Demand pagingとアプリの実際使用量の少なさを前提に成立
- 設定は `vm.overcommit_memory` チューナブルで制御
- オーバーコミットが問題になった場合: **OOM killer** が発動

### 7.2.5 Process Swapping

- プロセス全体をスワップデバイスに移動（古典的Unixの手法）
- 現代ではメモリが非常に少ない場合のみ発生

### 7.2.9 Shared Memory（共有メモリ）

- システムライブラリの read-only instruction text を複数プロセスで共有
- 計測の難しさ: RSSにはshared memoryが含まれるため合計するとシステムの実メモリを超える
- **PSS（Proportional Set Size）**: privateメモリ + 共有メモリ÷利用プロセス数
  - `pmap(1)` コマンドで確認可能

### 7.2.10 Working Set Size（WSS）

- **WSS**: プロセスが作業のために頻繁に使うメインメモリ量
- パフォーマンスの指針:
  - WSS < CPUキャッシュ → 非常に高速
  - WSS > メインメモリ → スワップが発生、大幅な性能劣化
- 計測が難しい（ツールはRSSを表示、WSSは別途推定が必要）

### 7.2.11 Word Size（ワードサイズ）

- **32-bit**: アドレス空間が最大4GB（Linuxカーネル予約で~1GBが減る）
- **64-bit**: アドレス空間が大幅に広くなりカーネル予約は問題にならない

---

## 7.3 Architecture（アーキテクチャ）

### Hardware

#### UMA vs NUMA

```
UMA（Uniform Memory Access）        NUMA（Non-Uniform Memory Access）
                                    
┌──────────────────────┐           ┌──────┐      ┌──────┐
│   CPU 1    CPU 2     │           │ DRAM │ CPU1 │ CPU2 │ DRAM
│      ↕         ↕     │           │  A   │←────→│←────→│  B
│   System Bus         │           └──────┘1 hop └──────┘2 hop
│   DRAM A ←→ DRAM B   │
└──────────────────────┘
全CPUが均等レイテンシ              ローカル(1hop)とリモート(2hop)で異なるレイテンシ
```

| アーキテクチャ | 特徴 |
|---|---|
| **UMA** | 全CPUが同じレイテンシでメモリアクセス |
| **NUMA** | CPU+メモリのノード構成。ローカルメモリ（1hop）が速く、リモートメモリ（2hop+）は遅い |

- OSはメモリローカリティを考慮してタスクを配置（NUMAアウェア）

#### Buses（バス）
- **Shared system bus**: 複数CPUが共有するシステムバス（旧来の構成）
- **Dedicated memory bus**: CPUごとの専用メモリバス（NUMA構成で使用）

### Software

#### プロセスの仮想メモリアドレス空間

```
0x0        ┌─────────────┐
           │  Executable  │
           │    Heap      │ ↓ (grows)
           │              │
           │  Libraries   │
           │    Stack     │ ↑ (grows)
           │    Kernel    │
0xFFFFFFFF └─────────────┘
           (x86例)
```

#### Heap Growth（ヒープの成長）
- `free(3)` はOSにメモリを返さない（将来の割り当てのために保持）
- そのため、プロセスのRSSは基本的に増加のみ
- メモリをOSに返す方法:
  - **Re-exec**: `execve(2)` で空のアドレス空間から再スタート
  - **Memory mapping**: `mmap(2)` / `munmap(2)` でOSにメモリを返却
- glibcは `malloc_trim(3)` で `free(3)` 時に自動的に上部の未使用ヒープを解放

#### Allocators（アロケータ）
- **ユーザーレベル**: malloc/free（glibc）、tcmalloc、jemalloc など
- **カーネルレベル**: スラブアロケータ（Linuxカーネルの内部アロケーション）

---

## 7.4 Methodology（方法論）

### 7.4.2 USE Method for Memory

| 指標 | 内容 | 確認方法 |
|---|---|---|
| **Utilization** | 使用中メモリ量と空き量（物理・仮想） | `free`, `vmstat`, `top` |
| **Saturation** | ページスキャン・ページング・スワップ・OOM killer | `vmstat` (si/so列), `dmesg` |
| **Errors** | OOM killer発動、ECCメモリエラー | `dmesg \| grep "Out of memory"` |

- **飽和（Saturation）から先に確認する**（memoryの問題の明確なサイン）
- 物理メモリ使用率はツールによってファイルキャッシュの扱いが異なるため注意

```bash
# Utilization確認
free -h
vmstat 1

# Saturation確認（si/soが継続的に非ゼロならスワッピング中）
vmstat 1

# Errors確認
dmesg | grep -i "out of memory"
dmesg | grep -i "oom"
```

### 7.4.8 Resource Controls

- OS提供のきめ細かいメモリ制御（cgroups等）
- プロセス・プロセスグループ単位でメインメモリ・仮想メモリの上限設定可能

### 7.4.9 Micro-Benchmarking

- メインメモリの速度やCPUキャッシュのサイズ特性を計測
- システム間の違いの把握に有用

### 7.4.10 Memory Shrinking（WSSの推定）

- **WSS推定の実験的手法**: 利用可能メモリを段階的に減らしながらパフォーマンスを計測
- 性能が急激に落ちてスワップが大幅増加した時点 → WSSがメモリに収まらなくなった点
- **本番環境では非推奨**（意図的にパフォーマンスを悪化させる）
- 実験的ツール: `wss(8)`

---

## 7.5 Observability Tools（観測ツール）

**Table 7.4: Linux memory observability tools**

| セクション | ツール | 説明 |
|---|---|---|
| 7.5.1 | `vmstat` | 仮想・物理メモリ統計 |
| 7.5.2 | `PSI` | メモリ圧力ストール情報 |
| ... | `top` | プロセスごとのメモリ使用量 |
| ... | `pmap` | プロセスのメモリマップ（PSS等） |
| ... | `numastat` | NUMAメモリ統計 |
| 7.5.7 | `ps` | プロセスのメモリ統計 |
| ... | `perf/BCC/bpftrace` | メモリ割り当てのトレース |

### vmstat（7.5.1）

```bash
$ vmstat -Sm 1
procs ------memory------- ---swap-- -----io---- -system-- ----cpu----
 r  b swpd  free  buff cache  si  so   bi   bo   in  cs us sy id wa
 4  0    0 35280  114 13761   0   0    0    5    2   1  0  0 100  0
```

| 列 | 意味 |
|---|---|
| **swpd** | スワップアウト済みメモリ量 |
| **free** | 空きメモリ量 |
| **buff** | バッファキャッシュのメモリ |
| **cache** | ページキャッシュのメモリ |
| **si** | スワップイン（ページイン）中のメモリ |
| **so** | スワップアウト（ページアウト）中のメモリ |

- **si/so が継続的に非ゼロ** → メモリ圧迫状態でスワッピング発生中
- `vmstat -a 1` : inactive/active のページキャッシュ内訳表示
- `vmstat -s` : サマリーリスト表示

### PSI（7.5.2）

- Linux 4.20で追加
- メモリ圧力ストール情報（Pressure Stall Information）
- 圧力の有無だけでなく、直近5分間の変化傾向も表示

### ps（7.5.7）

```bash
$ ps aux
USER   PID %CPU %MEM    VSZ   RSS TTY  STAT START  TIME COMMAND
nginx 13419  0.0  0.5  5176  1796 ?     Ss  Mar27  0:00 /usr/local/sbin/nginx
```

| 列 | 意味 |
|---|---|
| **%MEM** | システム全体のメモリに対する物理使用率 |
| **RSS** | Resident Set Size（KB）- 共有ライブラリ含む |
| **VSZ** | Virtual memory Size（KB） |

```bash
# SVR4形式で特定列だけ表示
# ps -eo pid,pmem,vsz,rss,comm
```

注意: RSSにはshared memoryが含まれるため、全プロセスのRSS合計 > 実物理メモリになることがある

---

## 7.6 Tuning（チューニング）

最重要のチューニング目標: **アプリケーションをメインメモリ内に保ち、ページングとスワッピングが頻繁に発生しないようにする**

### 7.6.1 Tunable Parameters（チューナブルパラメータ）

**Table 7.7: Example Linux memory tunables**

| パラメータ | デフォルト | 説明 |
|---|---|---|
| `vm.dirty_background_bytes` | 0 | pdflushバックグラウンドライトバックをトリガーするdirtyメモリ量 |
| `vm.dirty_background_ratio` | 10 | pdflushをトリガーするシステムメモリのパーセンテージ |
| `vm.dirty_bytes` | 0 | 書き込みプロセス自身がライトバックを開始するdirtyメモリ量 |
| `vm.swappiness` | 60 | スワッピングの積極性（0=最小、100=最大） |
| `vm.overcommit_memory` | 0 | オーバーコミット動作モード |

```bash
# チューナブルパラメータの確認・変更
sysctl vm.dirty_background_ratio
sysctl -w vm.swappiness=10
```

### cgroup によるリソース制御

| cgroupパラメータ | 説明 |
|---|---|
| `memory.memsw.limit_in_bytes` | 最大メモリ＋スワップ（バイト） |
| `memory.kmem.limit_in_bytes` | 最大カーネルメモリ（バイト） |
| `memory.tcp.limit_in_bytes` | 最大TCPバッファメモリ（バイト） |
| `memory.swappiness` | cgroupごとのスワッピング積極性（vm.swappiness相当） |
| `memory.oom_control` | 0=OOM killer有効、1=OOM killer無効 |

- システム全体の制限: `/etc/security/limits.conf`

---

## まとめ：Chapter 7 の要点

| セクション | キーポイント |
|---|---|
| **Paging** | File system paging（good）はclean pageを解放するだけ。Anonymous paging（bad）はディスクI/Oが必要 |
| **Virtual memory states** | A=未割当, B=割当済み未マップ, C=RAM上, D=スワップ上。RSS=C、VSZ=B+C+D |
| **Overcommit** | Linuxはmalloc成功を保証するが、実際に使うとOOM killerが発動することも |
| **NUMA** | ローカルメモリ（1hop）とリモートメモリ（2hop+）でレイテンシが異なる。メモリローカリティが重要 |
| **USE Method** | 飽和（si/soの非ゼロ、OOM killer）から確認する |
| **vmstat si/so** | 継続的に非ゼロ → メモリ圧迫・スワッピング発生中 |
| **Tuning目標** | アプリをRAM内に収める。vm.swappinessやcgroupで制御 |



Page 316のFigure 7.6から読み取った内容を基にお答えします。

OOM killer以外のメモリ解放手段は、空きメモリが減るにつれて段階的に発動します：

Linux のメモリ解放フロー（OOM killer以外）

Available Memory
     HIGH
      │
      ▼
┌─────────────────────────────────────────────┐
│ 1. Page Cache の解放（ファイルシステムページング）      │
│    clean ページ → そのまま解放（書き込み不要）          │
│    dirty ページ → ディスクへ writeback してから解放      │
└─────────────────────────────────────────────┘
      │
      ▼  vm.swappiness でバランスを調整
┌─────────────────────────────────────────────┐
│ 2. Swapping（kswapd: ページアウトデーモン）           │
│    最近使われていない anonymous ページ                │
│    （ヒープ・スタック）をスワップデバイスに退避          │
└─────────────────────────────────────────────┘
      │
      ▼  低メモリ閾値を下回ったとき
┌─────────────────────────────────────────────┐
│ 3. Reaping（= Shrinking）                        │
│    カーネルモジュールとスラブアロケータが               │
│    即座に解放できるメモリをまとめて返却                 │
│    例）未使用のスラブキャッシュ（inode キャッシュ等）    │
└─────────────────────────────────────────────┘
      │
     LOW → OOM Killer
各手段の詳細
手段	主体	対象メモリ	補足
Page Cache 解放	カーネル	ファイルシステムページキャッシュ	clean ページはI/Oなしで即解放。dirty はwriteback必要
Swapping	kswapd	Anonymous pages（ヒープ・スタック）	スワップデバイス/ファイルへ退避。vm.swappiness で積極性を調整
Reaping	スラブアロケータ	カーネル内部キャッシュ（inode, dentry等）	shrinking とも呼ぶ。OOM killerより先に動く最後の砦
vm.swappiness の役割
Page Cache解放（手段1）とSwapping（手段2）のどちらを優先するかを制御するパラメータ：


# 現在の値確認
sysctl vm.swappiness   # デフォルト: 60

# 0 に近い → Page Cache をなるべく残し、アプリをスワップしにくくする
# 100 に近い → 積極的にアプリメモリをスワップ
sysctl -w vm.swappiness=10   # アプリをスワップしにくくする例
手段ごとのパフォーマンスへの影響
手段	影響度	理由
Page Cache解放（clean）	低	ディスクI/Oなし
Page Cache解放（dirty）	中	writeback が発生
Swapping	高	アプリのページイン時にディスクI/Oでブロック
Reaping	低〜中	キャッシュ再構築コストはあるが直接ブロックなし
Swappingが始まった時点でパフォーマンスへの深刻な影響が出始めるため、vmstat の si/so 列を監視することが重要です。

メモ
cpuのアーキテクチャ復習

x86-64やARM64は、cpuのアーキテクチャ
アーキテクチャとは、cpuへの命令の仕様
appleは、ARM64ベースで独自のアーキテクチャ設計
cpu、メモリを独自で開発

大きな点は、ARMはスマホで使われている。後発商品。
スナップドラゴンやapple Aシリーズ
下記進化の過程

2020年以前の常識：
  x86-64 > ARM64（性能面）
  ARM64  > x86-64（省電力面）

2020年 Apple M1 登場後：
  性能でも ARM64 が x86-64 を超えた

  MacBook Pro M1 vs Intel MacBook Pro
    CPU 性能: M1 が約 2〜3倍 速い
    バッテリー: M1 が約 2倍 長持ち
    発熱: M1 が大幅に少ない

M1について
SoC = System on a Chip
    = システム全体を 1チップに集積

従来の PC：
  CPU      → 別チップ
  GPU      → 別チップ（グラフィックボード）
  RAM      → 別チップ（メモリスロット）
  Wi-Fi    → 別チップ
  → バラバラのチップをマザーボードで繋ぐ

M1（SoC）：
  全部が 1チップの中に入っている

よく聞く、Unified memoryについて
CPUとGPUはメモリを共有できるため、従来と違い、インターコネクトのオーバーヘッドが少ない。

x86-64 CPU の周辺は
バラバラの規格・バラバラのメーカーが担当

CPU           Intel / AMD が設計
              ↕
PCIe          PCIe 規格委員会が策定
              ↕
GPU           NVIDIA / AMD が独自設計
              ↕
RAM（DRAM）   JEDEC 規格（DDR5 等）
              ↕
チップセット   Intel / AMD が設計
              ↕
マザーボード   ASUS / MSI 等が設計


M1（SoC）が定義するのは「システム全体」

┌─────────────────────────────────────────┐
│              M1 の守備範囲               │
│                                         │
│  ・CPU コアの設計（ARM64 ベース）         │
│  ・GPU コアの設計（Apple 独自）           │
│  ・CPU-GPU 間のやり取りの仕組み           │
│  ・Unified Memory の構成                 │
│  ・Neural Engine の設計                  │
│  ・内部ファブリック（バス）の設計          │
│  ・Media Engine の設計                   │
│  ・Secure Enclave の設計                 │
│  ・各コンポーネント間の帯域・レイテンシ    │
│                                         │
└─────────────────────────────────────────┘


そのほか大きな違い
apple siliconだけページサイズが16kb
効率がい。性能向上・省電力

x86は、後方互換性のため変更できない。
すべてのドライバ、OS、アプリケーションは、4kb前提で作られているため、動かなくなる。

TLB（Translation Lookaside Buffer）とは
一言で言うと
仮想アドレス → 物理アドレスの変換結果をキャッシュするハードウェア。

なぜ TLB が必要か
まず前提を整理：


プロセスは「仮想アドレス」でメモリを参照する
CPU は「物理アドレス」でメモリにアクセスする

→ 毎回「仮想 → 物理」の変換が必要
変換の仕組み：


仮想アドレス
    ↓
ページテーブル（RAM 上にある変換表）を参照
    ↓
物理アドレス
    ↓
データにアクセス

問題：


ページテーブルは RAM 上にある

つまり：
  データにアクセスするために
  まずページテーブルを RAM から読む（1回目）
  その後データを RAM から読む（2回目）

→ 毎回 RAM アクセスが 2倍 になる
→ 遅い

TLB が解決する

TLB = ページテーブルの変換結果を CPU 内にキャッシュ

変換結果を覚えておく：
  仮想アドレス 0x7fff0000 → 物理アドレス 0x1a2b0000
  仮想アドレス 0x7fff1000 → 物理アドレス 0x3c4d0000

  CPU
┌─────────────────────────────┐
│  レジスタ                    │ ← 最速
│  ────────────────────────── │
│  L1 Cache  ┌────────┐      │
│            │  TLB   │      │ ← L1 と同じくらい速い
│  L2 Cache  └────────┘      │ ← CPU 内部にある
│  L3 Cache                   │
└─────────────────────────────┘
          ↕
┌─────────────────────────────┐
│  RAM（ページテーブルはここ）  │ ← TLB ミス時に参照
└─────────────────────────────┘

ページサイズと TLB の関係

ここで前の会話と繋がる：

4KB ページ（x86）で 1GB を管理：
  1GB ÷ 4KB = 262,144 ページ
  → TLB に 262,144 エントリ必要
  → TLB は容量が小さい（数百〜数千エントリ）
  → エントリが溢れる → TLB ミス多発

16KB ページ（Apple Silicon）で 1GB を管理：
  1GB ÷ 16KB = 65,536 ページ
  → TLB エントリが 1/4 で済む
  → TLB ミスが減る → 速い

  
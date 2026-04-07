# Chapter 7: Memory 練習問題 解答

---

## 問1. 用語に関する問題

### Q: What is a page of memory?（メモリのページとは何か？）

OSとCPUが使用するメモリの管理単位。歴史的に 4KB または 8KB。現代のプロセッサは複数のページサイズをサポートしており、Linux では通常 4KB（標準）と 2MB/1GB（Huge Pages）が使える。

OSはページ単位でメモリの割り当て・解放・スワップを管理する。

---

### Q: What is resident memory?（レジデントメモリとは何か？）

現在、物理メモリ（RAM）上に実際に存在しているメモリ。**RSS（Resident Set Size）** とも呼ばれる。

- `ps` の `RSS` 列、`top` の `RES` 列で確認できる
- 共有ライブラリのメモリも含まれる
- スワップアウトされた部分は含まれない

---

### Q: What is virtual memory?（仮想メモリとは何か？）

物理メモリの抽象化。プロセスにとってほぼ無限かつ競合のないメモリ空間を提供する仕組み。

特徴：
- 実際の物理メモリではなく、アドレス空間の「見かけ上のメモリ」
- 複数プロセスが互いの領域を侵犯しないよう分離（アドレス空間の分離）
- 物理メモリ以上の容量を仮想的に利用可能（スワップと組み合わせて）
- MMUとページテーブルによって仮想アドレスを物理アドレスに変換

---

### Q: Linux 用語として、paging と swapping の違いは何か？

| 用語 | Linux での意味 |
|---|---|
| **Paging** | メインメモリとストレージデバイス間のページ転送全般。File system pagingとanonymous pagingの両方を含む |
| **Swapping** | Linuxでは特に **anonymous paging**（プロセスのヒープ・スタックのスワップデバイスへのページアウト）を指す |

> 注意: 伝統的な Unix の "swapping" = プロセス全体をスワップする意味だったが、Linux ではこの狭い意味で使う

---

## 問2. 概念に関する問題

### Q: What is the purpose of demand paging?（デマンドページングの目的は何か？）

**物理メモリの割り当てを、実際にそのページが初めてアクセスされる瞬間まで遅らせる**仕組み。

目的：
1. **メモリ節約**: 実際には使われないページのために物理メモリを事前確保しない
2. **大きなプログラムの実行**: 物理メモリより大きなプログラムも、必要な部分だけロードして実行可能
3. **起動の高速化**: プログラム全体をメモリに読み込まずに実行開始できる

仕組み：
- ページが割り当て済み（Allocated）だがまだ物理メモリにマップされていない状態（状態B）
- 初めてアクセスしたときに **page fault** が発生し、その時点で物理ページを割り当てる

---

### Q: Describe memory utilization and saturation.（メモリの使用率と飽和を説明せよ）

**Utilization（使用率）**
- どれだけのメモリが使用中か（物理・仮想メモリの両方を確認する）
- 物理メモリの使用率はツールによって報告方法が異なる（ファイルキャッシュを「使用中」に含めるかどうか）
- `free` の `available` 列は「即座に解放できるキャッシュ」も含めた実際の空き量

```bash
free -h
# または
vmstat -Sm 1  # swpd, free, buff, cache 列
```

**Saturation（飽和）**
- ページスキャン・ページング・スワッピング・OOM killer が発動している程度
- vmstat の `si`（swap in）`so`（swap out）列が継続的に非ゼロ → メモリ圧迫・スワッピング発生中
- OOM killer が発動 → `dmesg` に "Out of memory" が記録される

```bash
vmstat 1              # si/so列を監視
dmesg | grep -i "out of memory"
```

---

### Q: What is the purpose of the MMU and the TLB?（MMU と TLB の目的は何か？）

**MMU（Memory Management Unit）**
- 仮想アドレスを物理アドレスに変換するハードウェア
- ページテーブルを参照して変換を行う
- プロセス間のメモリ空間の分離（保護）も担う

**TLB（Translation Lookaside Buffer）**
- ページテーブルの変換結果をキャッシュするハードウェア
- TLBヒット → 高速に変換完了
- TLBミス → ページテーブルウォーク（低速）が発生
- **Huge Pages（2MB/1GB）** を使うとTLBエントリ1つでカバーするアドレス範囲が広がり、TLBミスを大幅に削減できる

```
CPU → 仮想アドレス → TLB確認 → [ヒット] 物理アドレス取得
                              → [ミス]  ページテーブルウォーク → 物理アドレス取得（低速）
```

---

### Q: What is the role of the page-out daemon?（ページアウトデーモンの役割は何か？）

Linuxでは **kswapd** がページアウトデーモンの役割を担う。

役割：
- **最近使用されていないページを探してフリーリストに追加する**
- アプリケーションメモリ（anonymous pages）や、ファイルシステムのページキャッシュを対象に
- スワップデバイスまたはスワップファイルにページアウト
- **バックグラウンドで常時動作**し、空きメモリが一定量を下回らないよう維持する
- `vm.swappiness`（デフォルト60）によって、ページキャッシュを解放するか、アプリケーションメモリをスワップするかのバランスを調整

---

### Q: What is the role of the OOM killer?（OOM killer の役割は何か？）

**Out-of-Memory killer**。他のメモリ解放手段が尽きたときの最終手段。

役割：
- `select_bad_process()` で「犠牲にするプロセス」を選択
- `oom_kill_process()` でそのプロセスをkill
- ログは `/var/log/messages` または `dmesg` に `"Out of memory: Kill process"` として記録される

選択基準：
- 大量のメモリを消費しているプロセスが優先的に選ばれる（スコアリング方式）

```bash
# OOM killer の発動履歴確認
dmesg | grep -i "out of memory"
dmesg | grep -i "oom"
grep -i "out of memory" /var/log/messages
```

---

## 問3. より深い問題

### Q: anonymous paging とは何か？なぜファイルシステムページングより重要なのか？

**Anonymous paging とは**

プロセスのプライベートデータ（ヒープ・スタック）のスワップデバイスへのページング。「anonymous」の名の通り、ファイルシステム上のパスを持たないメモリが対象。Linux ではこれを **swapping** と呼ぶ。

**なぜ file system paging より重要か**

| 比較点 | File System Paging | Anonymous Paging |
|---|---|---|
| 別名 | "good" paging | "bad" paging |
| clean ページの扱い | ディスクに書かず解放するだけ（コピーはディスク上に既存） | 必ずスワップデバイスへの書き出しが必要 |
| I/O の発生 | dirty ページのみ | 常に発生 |
| ページイン時 | ディスクから読み直す（ファイルのキャッシュ） | スワップデバイスからの読み出し（ブロッキング） |
| パフォーマンスへの影響 | 比較的軽微 | **ディスクI/Oをブロック**し直接レイテンシに影響 |

→ anonymous paging（swapping）はアプリケーションが直接ブロックされるため、**パフォーマンスへの影響が深刻**。

---

### Q: フリーメモリが枯渇したとき、Linuxカーネルが踏む手順を説明せよ

Figure 7.6「Linux memory availability management」によると、利用可能メモリが減るにつれて以下の順に対応する：

```
Available Memory: High
         |
         ▼
[1] Free List からの割り当て（通常動作）
         |
         ▼  メモリ減少
[2] Page Cache の解放（vm.swappiness に基づき、
    ファイルキャッシュの解放 ↔ Swappingのバランスを調整）
         |
[3] Swapping（page-out daemon: kswapd）
    最近使われていないページをスワップデバイスに退避
         |
[4] Reaping（低メモリ閾値を下回ったとき）
    カーネルモジュールとスラブアロケータが即座に解放できるメモリを返却
    ＝ "shrinking"
         |
         ▼  Available Memory: Low
[5] OOM Killer（最終手段）
    select_bad_process() → oom_kill_process()
    犠牲プロセスをkillしてメモリを解放
```

`vm.swappiness`（デフォルト60）は[2]と[3]のバランスを制御する。高い値はより積極的にアプリメモリをスワップする。

---

### Q: スラブベースのアロケーションのパフォーマンス上の利点を説明せよ

**スラブアロケータ（Slab Allocator）** はBonwick (1994) がUnixカーネル向けに設計したオブジェクトキャッシング型のカーネルメモリアロケータ。

**パフォーマンス上の利点：**

| 利点 | 説明 |
|---|---|
| **断片化の削減** | 同サイズのオブジェクトを「スラブ」にグループ化。外部断片化を防ぐ |
| **高速な割り当て・解放** | 同じ型のオブジェクト（inodeなど）をキャッシュし再利用。mallocのような汎用アロケータより高速 |
| **CPUキャッシュの温存** | オブジェクトを初期化済みの状態でキャッシュするため、コンストラクタ/デストラクタのコストを繰り返し払わない |
| **メモリ再利用の効率化** | free後もオブジェクトをキャッシュ内に保持し、次の同型割り当てで即返却 |
| **Reaping対応** | メモリ圧迫時にslab cacheをすぐに解放できる（shrinking）。OOM killerより先に対応可能 |

Linux では現在 **SLUB**（The unqueued slab allocator）が使われており、さらなるCPUスケーラビリティを実現している。

---

## 問4. 手順の開発

### USE Method チェックリスト（メモリリソース）

| 指標 | コマンド | 解釈 |
|---|---|---|
| **Utilization（物理）** | `free -h` | `available` 列が少ない → 実質的に空きが少ない |
| **Utilization（仮想）** | `vmstat -Sm 1` の `swpd` 列 | 非ゼロ = スワップが使われている |
| **Saturation（スワッピング）** | `vmstat 1` の `si` / `so` 列 | 継続的に非ゼロ → メモリ圧迫でスワッピング発生中 |
| **Saturation（OOM killer）** | `dmesg \| grep -i "out of memory"` | 出力あり = OOM killerが過去に発動 |
| **Errors（ハードウェア）** | `dmesg \| grep -i "ecc\|mce\|hardware error"` | ECCエラー・MCEはメモリハードウェア障害 |
| **Errors（割り当て失敗）** | `dmesg \| grep -i "cannot allocate"` | カーネルのメモリ割り当て失敗 |

```bash
# 一括確認スクリプト例
echo "=== Memory Utilization ==="
free -h

echo "=== vmstat (5秒間) ==="
vmstat -Sm 1 5

echo "=== OOM killer history ==="
dmesg | grep -i "out of memory" | tail -5

echo "=== Hardware errors ==="
dmesg | grep -i "ecc\|machine check" | tail -5
```

---

### Workload Characterization チェックリスト（メモリリソース）

| 指標 | コマンド | 説明 |
|---|---|---|
| システム全体のメモリ概要 | `free -h` | Total / Used / Free / Cache の内訳 |
| 仮想・スワップ状況 | `vmstat -Sm 1` | swpd・si・so の推移 |
| ページキャッシュの inactive/active | `vmstat -a 1` | アクティブなキャッシュ量 |
| メモリ消費上位プロセス（RSS） | `ps aux --sort=-%mem \| head -20` | どのプロセスが最も物理メモリを使っているか |
| プロセスごとのVSZ・RSS | `ps -eo pid,pmem,vsz,rss,comm --sort=-rss \| head -20` | 詳細なメモリサイズ内訳 |
| メモリ圧力トレンド（PSI） | `cat /proc/pressure/memory` | 直近の圧力（some/full）の傾向 |
| NUMAメモリ分布 | `numastat -m` | NUMAノードごとのメモリ配置（マルチソケット環境） |

```bash
# 代表的なワークロード特性確認コマンド
echo "=== Top memory consumers ==="
ps aux --sort=-%mem | head -15

echo "=== Memory summary ==="
vmstat -s | head -20

echo "=== NUMA stats (if applicable) ==="
numastat -m 2>/dev/null || echo "numastat not available"

echo "=== PSI memory pressure ==="
cat /proc/pressure/memory 2>/dev/null || echo "PSI not supported"
```

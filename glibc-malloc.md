# glibc malloc — あなたの手元で動く実装

Linux で C プログラムをコンパイルして `malloc` を呼ぶと、
たいていの場合この **glibc malloc**（実装名 **ptmalloc2**）が動く。
[ヒープを覗く章](inside-malloc.md)で観察したのも、Ruby の
「`MALLOC_ARENA_MAX=2` でメモリが減る」のも、すべてこの実装の話だ。
ptmalloc2 は、前章の [dlmalloc](dlmalloc.md) に
**マルチスレッド対応（アリーナ）**と**スレッドキャッシュ（tcache）**を
足したものである。だから本章は「dlmalloc に何を足したか」を軸に読むのが速い。

## アリーナ — 並列性のための複数ヒープ

dlmalloc は単一ヒープ＋単一ロックで、
[マルチスレッドの章](multithread.md)で見たロック直列化に陥る。
ptmalloc2 の答えが[アリーナ](#index:アリーナ)（arena）である。
アリーナとは「独立したヒープ＋専用ロック」の一式で、
スレッドを別々のアリーナに振り分けて競合を減らす。

- 最初のアリーナ（**メインアリーナ**）は `brk` ベースのヒープを使う。
- スレッドが `malloc` 時にメインアリーナのロックを取れなかったら、
  別の（または新しい）**非メインアリーナ**を割り当てて使う。
  非メインアリーナは `mmap` で確保した独立領域上に作られる。
- 各スレッドは一度結びついたアリーナを使い続ける（thread-local に記憶）。

アリーナ数には上限があり、既定で **CPU コア数 × 8**（64 ビット）である。
ここで有名な `MALLOC_ARENA_MAX` 環境変数が効く。
アリーナを増やすと並列性は上がるが、**各アリーナが独立に在庫を抱える**ので、
[マルチスレッドの章](multithread.md)の blowup と同種のメモリ増加が起きる。

> [!IMPORTANT]
> 「Rails アプリで `MALLOC_ARENA_MAX=2` を設定するとメモリ使用量が減る」
> という定番の対処の正体がこれだ。多数のスレッドがアリーナを増やし、
> それぞれが断片化した在庫を持つことで RSS が膨らむ。
> アリーナ数を 2 に絞れば在庫の総量が減り、メモリが安定する——
> 並列性能と引き換えにメモリを節約する調整である。
> 同じ理由で `--with-jemalloc` 版 Ruby も使われる（[jemalloc の章](jemalloc.md)）。

実際に観察してみよう。8 スレッドがそれぞれ `malloc(4096)` を 64 個確保し、半分だけ `free` して在庫を抱えたところで `malloc_stats()` を呼ぶ（全スレッドが確保し終えるのをバリアで待ってから表示する）。

```c
void *worker(void *_) {
    void *p[64];
    for (int i = 0; i < 64; i++) p[i] = malloc(4096);
    for (int i = 0; i < 64; i += 2) free(p[i]); /* 半分だけ返す */
    /* …バリアで待機… */
}
/* main: 8 スレッド起動 → 全員の確保完了を待って malloc_stats() */
```

既定と `MALLOC_ARENA_MAX=2` で、`malloc_stats()` の末尾 `Total` を比べる（glibc 2.39, 16 コア機。数値は環境により変わる）。

```
$ ./arena_demo 2>&1 | grep -A2 'Total'
Total (incl. mmap):
system bytes     =    2265088     ← 既定: アリーナ 9 個

$ MALLOC_ARENA_MAX=2 ./arena_demo 2>&1 | grep -A2 'Total'
Total (incl. mmap):
system bytes     =    1343488     ← アリーナ 2 個
```

同じ作業でも、アリーナが 9 個に分かれると OS から仕入れた総量（system bytes）は約 2.27 MB、2 個に絞ると約 1.34 MB。**各アリーナが独立に在庫を持つ**ぶんだけ総量が増える、というボックスの主張がそのまま数字に出ている。

## tcache — スレッドごとの速いパス

ptmalloc2 はもうひとつ、glibc 2.26（2017 年）で大きな改良を入れた。
[tcache](#index:tcache)（per-thread cache）である。
[マルチスレッドの章](multithread.md)のスレッドキャッシュの実装で、
アリーナのロックすら取らずに済む最速パスを提供する。

- 各スレッドが、サイズクラス（64 種、最大 1032 バイト程度）ごとに
  小さな単方向リストを**ロックなし**で持つ。1 クラスあたり既定 7 個まで。
- `malloc` はまず tcache を見て、あれば pop して即返す。
  ここにヒットする限り、アリーナのロックも fast bin も触らない。
- `free` は、サイズが合えばまず tcache へ push する。
- tcache が一杯（7 個）なら、その先は従来の fast bin / アリーナへ回る。

tcache の導入で、glibc malloc のシングルスレッド・マルチスレッド両方の
性能が大きく向上した。[ヒープを覗く章](inside-malloc.md)で
「2 個目の `malloc(1000)` はシステムコールを起こさない」と見たが、
小さいサイズなら今や「2 個目はロックすら取らない」のである。

tcache は単方向リストの **LIFO**（後入れ先出し）なので、`free` した直後に同サイズを `malloc` すると、たった今返したのと同じ番地が戻ってくる。9 個確保して全部 `free` し、取り直すと「7 個まで」の境界も見える。

```c
void *a = malloc(64); free(a);
void *b = malloc(64);            /* tcache から pop */
/* a == b になる */

void *p[9];
for (int i = 0; i < 9; i++) p[i] = malloc(64);
for (int i = 0; i < 9; i++) free(p[i]);   /* 9 個 free */
for (int i = 0; i < 9; i++) printf("%p ", malloc(64));  /* 取り直す */
```

```
free(a) 直後の malloc: a=0x...d72a0 b=0x...d72a0  -> 同一（tcache から pop）
free 順:   p0=...300 p1=...350 p2=...3a0 p3=...3f0 p4=...440 p5=...490 p6=...4e0 p7=...530 p8=...580
再 malloc: ...4e0 ...490 ...440 ...3f0 ...3a0 ...350 ...300   ...580 ...530
```

再 `malloc` の最初の 7 個は `p[6]→p[0]` と**逆順**に戻る——後から `free` した順に積まれた tcache を LIFO で pop しているからだ。8・9 個目（`...580 ...530`）は tcache が満杯（7 個）になった後の `free` なので fast bin 側へ回り、出てくる順序が違う。文章の「1 クラスあたり既定 7 個まで」が、番地の戻り順としてそのまま見えている。

tcache の在庫数や上限サイズは `mallopt` / 環境変数
（`glibc.malloc.tcache_count` など、`GLIBC_TUNABLES` 経由）で調整できる。

## 全体像 — 確保が通る道

dlmalloc の指定適合（[dlmalloc の章](dlmalloc.md)）に
tcache とアリーナが加わった、ptmalloc2 の `malloc` の流れはこうなる。

```mermaid
graph TD
    M["malloc(n)"] --> T{"tcache に空き?"}
    T -->|あり| TR["pop して返す<br/>(ロックなし・最速)"]
    T -->|なし| L["アリーナのロックを取る"]
    L --> F{"fast bin / small bin<br/>に一致?"}
    F -->|あり| FR["取り出して返す"]
    F -->|なし| U["unsorted bin を走査<br/>(仕分けつき)"]
    U --> B["large bin で best-fit<br/>(binmap)"]
    B --> TOP["top チャンクから切り出す"]
    TOP --> OS["brk / mmap で OS から仕入れる"]
```

上から順に試し、ヒットしたところで止まる。
大多数の確保は最上段の tcache で終わる。
この「速いパスを上に、正確なパスを下に」という多段構造は、
ptmalloc2 に限らず本書のすべてのアロケータに共通する骨格である。

## 観察と診断

glibc malloc の内部構造をさらに深く追うなら、公式 wiki の Malloc Internals 解説
[](#cite:glibcwiki)が最も信頼できる一次資料である。
ここでは、その内部を外から観察するための API を整理しておく
（[ヒープを覗く章](inside-malloc.md)・[付録](tools.md)でも使う道具だ）。

- `malloc_stats()`: アリーナごとの確保量・空き量を stderr に表示。
- `malloc_info(0, stream)`: より詳しい統計を XML で出力。
  アリーナ数、bin ごとの空き、mmap 領域などが分かる。
- `mallinfo2()`: 集計値を構造体で返す（古い `mallinfo()` は
  32 ビットあふれの問題があり、2.33+ の `mallinfo2()` が推奨）。
- 環境変数 `MALLOC_CHECK_=3`: 二重解放やオーバーフローを
  検出したら中断する簡易チェックモード（本番では使わない）。
- `M_PERTURB`（`mallopt`）: 確保領域を既知のバイトで埋め、
  「初期化忘れ」を顕在化させる。

「`free` したのに RSS が減らない」を診断するときは、
`malloc_stats()` で「アリーナは空いている（in use が小さい）が
system bytes が大きい」状態を確認し、`malloc_trim(0)` で
末尾返却を試みる、という手順になる
（[ヒープを覗く章](inside-malloc.md)の実験 3 を思い出してほしい）。

## セキュリティ強化の歴史

glibc malloc は、[セキュリティの章](security.md)で見たヒープ攻撃に対して、
長年かけて検査を足してきた。代表的なものを挙げる。

- **チャンクサイズの健全性検査**: `free` 時に「次のチャンクのサイズが
  妥当か」「アラインメントが合っているか」を確認し、
  壊れていれば `malloc(): corrupted ...` で中断する。
- **fast bin / tcache の二重解放検査**: 同じチャンクが
  リスト先頭に二度入るのを検出する（tcache には専用の
  `key` フィールドがあり、解放済みマークに使う）。
- **unlink の検査**: 双方向リストから外すとき
  `fd->bk == p && bk->fd == p` を確認し、古典的な unlink 攻撃を防ぐ。
- **safe-linking**（glibc 2.32+）: fast bin / tcache の単方向ポインタを、
  **その格納位置のアドレスをページ粒度で右シフトした値**（`pos >> 12`）と
  XOR して格納する。格納位置を知らない攻撃者がポインタを生で
  書き換えても復号で破綻し、副次的にアラインメント検査にもなる緩和策である。

この二系統の検査は、実際に二重 `free` を起こすと別々のメッセージで現れる。`GLIBC_TUNABLES` で tcache を切ると、同じバグが通る経路が変わるのが分かる。

```c
void *p = malloc(64);
free(p);
free(p);   /* 二重解放 */
```

```
$ ./dfree
free(): double free detected in tcache 2          ← tcache の key で検出

$ GLIBC_TUNABLES=glibc.malloc.tcache_count=0 ./dfree
double free or corruption (fasttop)               ← tcache 無効 → fast bin の検査で検出
```

既定では `free` がまず tcache に入るので、`key` フィールドによる二重解放検査が `tcache 2` で止める。`tcache_count=0` で tcache を無効化すると同じチャンクが fast bin へ回り、今度は fast bin 側の検査（`fasttop`）が捕まえる。本文で挙げた二つの検査が、それぞれどの経路を守っているかが見える。

safe-linking が本当に「`pos >> 12` と XOR して格納している」のかは、`free` 済みチャンクの先頭 8 バイト（next ポインタの格納位置）を読めば確かめられる。2 個確保して `b`→`a` の順に `free` すると、`a` の next は本来 `b` を指すはずだ。

```c
void *a = malloc(64), *b = malloc(64);
free(b); free(a);                       /* tcache: a->next は b を指す */
uint64_t stored  = *(uint64_t *)a;      /* 格納されている生の値 */
uint64_t decoded = stored ^ ((uint64_t)a >> 12);  /* safe-linking 復号 */
```

```
a            = 0x5f4ed56182a0
b            = 0x5f4ed56182f0
a に格納された生の値 = 0x00005f4b218cd4e8   ← b とは全くの別物
pos>>12 で復号した値 = 0x00005f4ed56182f0   ← b と一致
```

生で読むと `0x5f4b218cd4e8` というデタラメな値だが、`a` 自身のアドレスを 12 ビット右シフトした値と XOR すると、ぴたり `b`（`0x5f4ed56182f0`）に戻る。攻撃者が next を生のポインタに書き換えても、復号時に格納位置とのつじつまが合わず破綻する——これが safe-linking の効き目である（アドレスは ASLR で毎回変わるが、この復号関係は常に成り立つ）。

これらは攻撃を完全には防げないが、攻撃の難度を上げる
「多層防御」の積み重ねだ。glibc malloc は速度最優先の汎用実装なので、
[セキュリティの章](security.md)の DieHarder や hardened_malloc のような
**強い**防御は持たない。強い保証が要るなら専用アロケータに差し替える、
という棲み分けになっている（[その他のアロケータの章](other-allocators.md)）。

## まとめ

glibc malloc（ptmalloc2）は、[dlmalloc](dlmalloc.md) の
チャンク・bin・指定適合という土台に、
**アリーナ**（複数ヒープでロック競合を分散）と
**tcache**（スレッドごとのロックなし速いパス）を足した実装である。
`MALLOC_ARENA_MAX` の挙動も、`free` 後に RSS が減らない現象も、
この設計から説明できる。汎用性と互換性を最優先するため
セキュリティ強化は緩和策の積み重ねにとどまる。
次章では、同じ問題を「最初からマルチスレッド前提」で設計し直した
jemalloc を見る。

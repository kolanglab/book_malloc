# アリーナとサイズクラスを精緻化する jemalloc

**jemalloc** は、Jason Evans が 2005 年に FreeBSD の libc 用に書き、
のちに Facebook（現 Meta）で大規模サーバ向けに磨かれたアロケータである
[](#cite:evans2006)。FreeBSD の標準 malloc であり、
Firefox が長く採用し、Rust が初期に既定アロケータとして使い、
Redis や Cassandra など多くのサーバソフトが採用してきた。
設計思想は「最初からマルチスレッド前提」「断片化を統計で抑え込む」
「徹底的に観測可能にする」である。
[glibc malloc](glibc-malloc.md) が dlmalloc に並行性を後付けしたのに対し、
jemalloc は並行性を土台から設計している点が対照的だ。

## アリーナとサイズクラス

jemalloc も[アリーナ](#index:アリーナ)を使うが、glibc とは割り当て方が違う。
スレッドは生成時に**ラウンドロビン**で固定数のアリーナへ振り分けられる
（既定でコア数の数倍）。glibc のように「ロックが取れなかったら増やす」
動的な増殖をしないので、アリーナ数が予測可能で、
[マルチスレッドの章](multithread.md)の blowup を制御しやすい。
各スレッドには **tcache** があり、ここはどのアロケータとも同じ
「ロックなしの速いパス」だ。

サイズクラス設計が jemalloc の見どころである。
[サイズクラスの章](segregated-lists.md)で生成したような
「各 2 倍区間を 4 分割」した細かい系列を使い、
オブジェクトを大きさで大別する。

具体的な値を見てみよう。以下は現行の jemalloc 5.x、最も一般的な
64 ビット、4 KiB ページ、16 バイト quantum の既定構成で、サイズクラスは
次のように並ぶ（4.x 以前は区分も境界も異なる。後述の NOTE を参照）。

- `8`：最小クラス。
- `16, 32, 48, 64, 80, 96, 112, 128`：quantum（16 バイト）刻み。
- `160, 192, 224, 256` / `320, 384, 448, 512` / ……：2 倍区間ごとに 4 分割。
  刻みが常に区間の 1/4 なので、内部断片化は最大でも約 20%。
- `14336`（14 KiB）：small の最大クラス。
- `16384`（16 KiB）以上：スラブではなくページ単位の **large**
  （16 KiB, 20 KiB, …）に切り替わる。

実際に丸めてみると、この「約 20%」がそのまま現れる。`malloc_usable_size` は要求サイズが切り上げられた先のクラスサイズを返すので、両者の差が内部断片化だ（jemalloc 5.3.0、64 ビット、16 バイト quantum）。

```c
extern size_t malloc_usable_size(void *);
size_t reqs[] = {1, 9, 100, 129, 200, 257, 1000, 1025};
for (size_t i = 0; i < sizeof(reqs)/sizeof(reqs[0]); i++) {
  void *p = malloc(reqs[i]);
  size_t u = malloc_usable_size(p);
  printf("request=%5zu -> usable=%5zu  (内部断片化 %4.1f%%)\n",
         reqs[i], u, 100.0 * (u - reqs[i]) / u);
  free(p);
}
```

```text
request=    1 -> usable=    8  (内部断片化 87.5%)
request=    9 -> usable=   16  (内部断片化 43.8%)
request=  100 -> usable=  112  (内部断片化 10.7%)
request=  129 -> usable=  160  (内部断片化 19.4%)
request=  200 -> usable=  224  (内部断片化 10.7%)
request=  257 -> usable=  320  (内部断片化 19.7%)
request= 1000 -> usable= 1024  (内部断片化  2.3%)
request= 1025 -> usable= 1280  (内部断片化 19.9%)
```

小さいクラスでは比率が跳ね上がるが（`1`→`8` で 87.5%）、これは絶対量が小さいので実害は少ない。問題は中〜大サイズで、`257`→`320` や `1025`→`1280` のように**区間の入口を 1 バイト超えた要求**が最悪ケースになり、いずれも約 20% に収まる。4 分割の刻みがこの上限を保証している。

これらの small クラスは、サイズクラスごとの**スラブ**（[スラブの章](buddy-slab.md)で
学んだ、固定サイズを敷き詰めた連続領域。jemalloc はこれを slab と呼ぶ）で
管理され、ビットマップで空きスロットを追う。ヘッダをオブジェクトごとに持たず、
**メタデータをデータと分離**して別領域に置くのも特徴で、これは局所性と
セキュリティの両方に効く。

> [!NOTE]
> 上に挙げた具体値は 5.x のものだ。**大別の区分と境界は
> jemalloc 5.0（2017-06-13）で大きく変わった**。
>
> - **4.x まで（3 区分）**：small（8 B〜3584 B、< 1 ページ）、
>   large（4 KiB〜、ページ刻み）、
>   huge（4 MiB〜、chunk 刻みで全スレッド共有の別構造）。
> - **5.0 以降（2 区分）**：large と huge が統合され、small（8 B〜14 KiB）と
>   large（16 KiB〜 `PTRDIFF_MAX` 未満）になった。
> - **管理単位**：固定 chunk（既定 4 MiB）から可変の
>   **extent（エクステント）** へ移行。後述の low-address reuse や decay は
>   この extent 単位の話だ。
> - **なぜ変わったか**：これらは別々の整理ではなく、ひとつの再設計の帰結だ[](#cite:jemalloc500)。
>   4.x の huge は**全スレッド共有の赤黒木＋単一 mutex**で管理され、ここが
>   スケーラビリティの競合点だった。しかもこの global mutex のせいで chunk を
>   大きく（4 MiB）保つ圧力がかかっていた（本来 256 KiB 程度でも足りる）。
>   管理を**アリーナごとの extent** に移すと global な huge 経路が消え、
>   large と huge を 1 区分に畳めた。固定で自然境界の chunk は**仮想メモリの
>   外部断片化**を生み huge page とも相性が悪かったが、可変長 extent は
>   メタデータを別所に置けるためこれを抑える。同時に、比率指定の
>   `lg_dirty_mult` を廃して dirty→muzzy→clean の二段 decay（秒→ミリ秒解像度）に
>   置き換え、バックグラウンドスレッドでの purge を可能にした。
> - **実機の値**：man ページ（jemalloc.net）か
>   `mallctl` の `arenas.nbins` / `arenas.bin.<i>.size` /
>   `arenas.lextent.<i>.size` で確認できる。

## 低アドレス優先と decay による断片化対策

jemalloc は断片化対策に明確な方針を持つ。

ひとつは **low-address reuse**、つまり[フリーリストの章](free-list.md)で
良策とされたアドレス順 first-fit の精神だ。
スラブやエクステント（連続領域の管理単位）を選ぶとき、
**なるべく低いアドレスを優先**して再利用する。
こうすると使用中の領域がアドレス空間の低い側に寄り、
高い側を OS に返しやすくなる。「ヒープを下に詰める」発想である。

もうひとつが **dirty page decay** である。
`free` で空いたページをすぐ OS へ返すと、再確保のたびに
ページフォルトが起きて遅い。かといって持ち続けると RSS が膨らむ。
jemalloc は「空いてから時間が経つほど返す確率を上げる」
**時間減衰（decay）**で折り合いをつける。
未使用ページを `madvise`（[OS とのインターフェースの章](os-interface.md)、
`MADV_DONTNEED` か `MADV_FREE`）で返すまでの猶予を、
`dirty_decay_ms` / `muzzy_decay_ms` で調整できる。
「`free` してもしばらくは RSS が減らないが、放っておくと減る」
という jemalloc 特有の挙動は、この decay の表れだ。

この `madvise` は `strace` で直接見える。8 KiB を 5000 個確保して解放するだけのプログラムを、decay 即時（`dirty_decay_ms:0`）と既定とで比べる（秒数は環境により変わる）。

```console
$ MALLOC_CONF=dirty_decay_ms:0,muzzy_decay_ms:0 \
    strace -f -e trace=madvise -c ./prog
% time     seconds  usecs/call     calls    errors syscall
------ ----------- ----------- --------- --------- --------
100.00    0.015563           3      4990           madvise

$ strace -f -e trace=madvise -c ./prog   # 既定（10秒猶予）
  0.00    0.000000           0         2           madvise
```

即時返却では `free` のたびにページを返すので madvise が約 5000 回（ほぼ全部 `MADV_DONTNEED`）も飛ぶのに対し、既定ではプロセス実行中は 2 回しか出ない。猶予のあいだ抱えているからだ。decay を攻めると RSS は減るが、この**システムコールと再フォルトのコスト**を CPU で払うことになる。`dirty_decay_ms` のトレードオフはここに表れる。

これは実測すると一目瞭然だ。4 KiB を 2 万個確保してから全部 `free` し、`/proc/self/statm` の RSS と jemalloc の統計（`stats.active` = 使用中、`stats.resident` = OS から見て常駐）を比べる（jemalloc 5.3.0。値は環境により変わる）。

```text
=== 既定 (dirty_decay_ms=10秒) ===
確保直後: RSS=87744KB  active=80116KB  resident=87268KB
free直後: RSS=87816KB  active=  196KB  resident=87268KB

=== MALLOC_CONF=dirty_decay_ms:0 (即時返却) ===
確保直後: RSS=87652KB  active=80116KB  resident=87268KB
free直後: RSS= 7796KB  active=  196KB  resident= 7348KB
```

既定では `free` 直後に `active` は 80 MB → 0.2 MB へ落ちるのに、**RSS は 87 MB のまま**動かない。空きページを decay の猶予（既定 10 秒）のあいだ抱えているからだ。`dirty_decay_ms:0` にすると同じ `free` で RSS が即座に 8 MB まで落ちる。「`free` したのに RSS が減らない」と驚いたときは、まずこの decay を疑えばよい。

## MALLOC_CONF による観測とチューニング

jemalloc の大きな魅力が、**圧倒的な観測可能性**である。
`malloc_stats_print()` を呼ぶと、アリーナごと、サイズクラスごとの
確保量や断片化、decay の状況が事細かに出力される。
本番サーバのメモリ挙動をこれで日常的に監視できる。

実際の出力の一部を見てみよう。Ruby を `LD_PRELOAD` で jemalloc に差し替え、`MALLOC_CONF=stats_print:true`（終了時に統計を吐く）を付けて起動する。

```console
$ MALLOC_CONF=stats_print:true \
    LD_PRELOAD=/usr/lib/x86_64-linux-gnu/libjemalloc.so.2 ruby app.rb
```

冒頭の実行時オプションでまず構成が分かる（`narenas:64` と `dirty_decay_ms:10000` の固定アリーナと decay 既定値）。続く `bins:` 表が圧巻で、**サイズクラスごと**に確保量、スラブ数、充填率まで並ぶ（列を抜粋）。次は処理系の起動時点での `bins:` 表だ。

```text
  size ind   allocated   nmalloc   curregs  curslabs  regs   util
     8   0        1600       200       200        1   512  0.390
    16   1        6400       400       400        2   256  0.781
    32   2       16000       500       500        4   128  0.976
    64   4        4096        64        64        1    64  1
   160   9       16000       100       100        1   128  0.781
   256  12        4096        16        16        1    16  1
```

`util`（スラブ充填率）は、そのサイズクラスのスラブがどれだけ詰まっているかを示す。1 に近いほど無駄がなく、低ければスカスカのスラブが RSS を食っているサインだ。本番では `nmalloc`/`ndalloc` の差で増え続けるクラスを探し、リークや肥大の当たりをつける。これが章で言う「圧倒的な観測可能性」の実体である。

設定は環境変数 `MALLOC_CONF`（またはビルト時設定）で行う。
代表的な項目を挙げる。

- `background_thread:true`：専用バックグラウンドスレッドで
  decay（ページ返却）を非同期に行い、確保パスを軽くする。
- `dirty_decay_ms:N` / `muzzy_decay_ms:N`：ページ返却の積極度。
  小さくすると RSS は減るが CPU を食う。
- `narenas:N`：アリーナ数。メモリと並列性のトレードオフ。
- `prof:true`：**ヒーププロファイリング**を有効化。
  「どのコールスタックがどれだけ確保したか」を集計し、
  `jeprof` で可視化できる。メモリリークや肥大の調査に強力だ
  （[ツールの付録](tools.md)）。
- `thp:always`：スラブにヒュージページを積極利用
  （[局所性の章](locality.md)）。

```console
# 実行時にアロケータ挙動を変える（再コンパイル不要）
$ MALLOC_CONF=narenas:2,dirty_decay_ms:1000,background_thread:true \
  LD_PRELOAD=/usr/lib/x86_64-linux-gnu/libjemalloc.so.2 ./server
```

このチューニングの自由度が、jemalloc が「サーバ運用者に
選ばれる」理由のひとつである。glibc malloc の `mallopt` より
はるかに細かく、しかも本番で安全に効かせられる。

## Ruby と Rust での採用

jemalloc は言語処理系の世界でも存在感が大きい。

CRuby は `./configure --with-jemalloc` で jemalloc 版をビルドでき、
Rails アプリのメモリ削減と安定化の定番手段として使われてきた。
効く理由は[マルチスレッドの章](multithread.md)と
[glibc malloc の章](glibc-malloc.md)で見たとおりだ。
glibc のアリーナ増殖による断片化を、jemalloc の
「固定アリーナ＋低アドレス再利用＋decay」が抑える。
`MALLOC_CONF` で decay を効かせれば、長時間稼働しても
RSS が肥大しにくい。`LD_PRELOAD` でも `--with-jemalloc` でも、
[ヒープを覗く章](inside-malloc.md)で見たとおりプロセス全体に効く。

Rust は 2018 年頃まで jemalloc を既定アロケータにしていた
（その後、バイナリサイズと「OS 標準を尊重する」方針から
システムアロケータが既定になり、jemalloc は
`jemallocator` クレートで明示的に選ぶ形になった）。
高並列なサーバを書く Rust プログラマが、今でも
jemalloc や後述の mimalloc を選ぶのは普通のことだ。

> [!NOTE]
> jemalloc の upstream 開発は 2025 年にいったん終了し（リポジトリがアーカイブされた）、
> 開発の主軸が Meta 内のフォークへ移った。ところが 2026 年 3 月、Meta は jemalloc への
> 再投資を表明してアーカイブを解除し、開発を再開している。いずれにせよ FreeBSD 標準として、
> また膨大な既存システムで現役であり続けており、設計から学べることは何も減っていない。
> 「固定アリーナ、細かいサイズクラス、decay、深い観測可能性」という
> jemalloc の語彙は、アロケータを語るうえでの共通言語になっている。

## まとめ

jemalloc は、最初からマルチスレッドを前提に設計されたアロケータで、
ラウンドロビンの固定アリーナ、細かいサイズクラスとスラブ、
メタデータの分離、低アドレス再利用と時間減衰によるページ返却、
そして MALLOC_CONF による深いチューニングと観測可能性を特徴とする。
glibc malloc の弱点（アリーナ増殖による断片化）を構造的に抑えるため、
Ruby/Rails のメモリ対策や高並列サーバで広く選ばれてきた。
次章では、同じ「サーバ向け」でも Google の本番環境で鍛えられ、
ヒュージページ対応を牽引する TCMalloc を見る。

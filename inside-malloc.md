# ヒープの中身を覗いてみる

アルゴリズムの勉強に入る前に、実物を観察しよう。
この章では、特別なツール（`printf` と `strace` だけ）で
glibc malloc の挙動を外側から推理する。
「ヘッダはどこにあるか」「サイズはどう丸められるか」「いつ OS と取引するか」が、
実験で確かめられることを体感するのが目的だ。
ここで掴んだ感覚は、第 II 部のアルゴリズム、第 IV 部の実装を読む際の足場になる。

## 実験 1: アドレスの間隔からヘッダを推理する

まず、小さな確保を連続して行い、返ってくるアドレスを眺める。

```c
#include <stdio.h>
#include <stdlib.h>

int main(void) {
    for (int i = 0; i < 5; i++) {
        char *const p = malloc(24);
        printf("malloc(24) = %p\n", (void *)p);
    }
    return 0;
}
```

Linux/x86-64 + glibc での実行結果の一例（アドレスは環境ごとに変わる）:

```
malloc(24) = 0x55b9d22a02a0
malloc(24) = 0x55b9d22a02c0
malloc(24) = 0x55b9d22a02e0
malloc(24) = 0x55b9d22a0300
malloc(24) = 0x55b9d22a0320
```

24 バイトしか頼んでいないのに、アドレスは **0x20 = 32 バイト**間隔で並ぶ。
差分の 8 バイトはどこに消えたのか。これが**チャンクヘッダ**である。
glibc malloc は確保単位を[チャンク](#index:チャンク)（chunk）と呼び、
ユーザに渡す領域の直前 8 バイトに、チャンク自身のサイズとフラグを記録している。
24 + 8 = 32 バイトが 16 の倍数にちょうど収まるので、この間隔になる。

> [!NOTE]
> 「ヘッダ = 8 バイト」は導入用の単純化である。実際のチャンクには
> `size` の前にもう 1 ワード（`prev_size`）があるが、これは**直前の
> チャンクが空きのときだけ**意味を持ち、直前が使用中の間はその領域を
> 直前チャンクのユーザデータに転用してよい。逆に言えば、使用中チャンクは
> 「次のチャンクの `prev_size` 領域」まで自分のデータに使えるので、
> 実効的なヘッダの負担は 1 ワード（8 バイト）で済む。
> `malloc(24)` が 32 バイトのチャンクに収まるのは、この転用のおかげだ。
> 正確なレイアウトは [dlmalloc の章](dlmalloc.md)のチャンク図で見る。

`malloc_usable_size` で「実際に使える」サイズを聞くと、丸めの実態が見える。

```c
#include <malloc.h>   /* glibc 拡張 */
for (size_t n = 0; n <= 40; n += 8)
    printf("malloc(%2zu) -> usable %zu\n", n, malloc_usable_size(malloc(n)));
```

```
malloc( 0) -> usable 24
malloc( 8) -> usable 24
malloc(16) -> usable 24
malloc(24) -> usable 24
malloc(32) -> usable 40
malloc(40) -> usable 40
```

最小でも 24 バイト使え、以降は 16 バイト刻みで増える
（チャンク全体では 32, 48, 64, … バイト）。
`malloc(25)` は実質 40 バイト要求と同じ、ということだ。
この「刻み」がなぜ必要か、刻みの設計が何をトレードオフするかは、
[サイズクラスの章](segregated-lists.md)の主題である。

> [!CAUTION]
> ヘッダが「ユーザ領域の直前」にあるということは、
> **負のインデックスへの書き込み 1 つでアロケータの管理情報が壊れる**ということだ。
> `p[-1] = 0;` のようなバグは、その場では何も起きず、
> 後続の `malloc`/`free` で「malloc(): corrupted top size」のような
> 不可解なエラーを引き起こす。エラーを出した場所に原因はない。

百聞は一見にしかず、実際に壊してみよう。`malloc(16)` の領域を 64 バイト分はみ出して書き、隣のチャンクヘッダを潰す。

```c
char *p = malloc(16);
for (int i = 0; i < 64; i++) p[i] = 'A';  /* 16 バイトを大きくはみ出す */
void *q = malloc(16);                      /* ← ここで破綻が露見 */
```

```console
$ gcc -O0 corrupt.c && ./a.out
malloc(): corrupted top size
Aborted (core dumped)
```

`p` への書き込み自体は静かに成功する。abort するのは、無関係に見える次の `malloc`（壊れた size を信じてヒープを辿った瞬間）だ。スタックトレースを見ても `for` ループは出てこない。これがヒープ破壊バグのデバッグを難しくする正体である。

## 実験 2: strace でカーネルとの取引を見る

次に、アロケータがいつ OS からメモリを仕入れるかを `strace` で観察する。

```c
#include <stdlib.h>
int main(void) {
    void *a = malloc(1000);        /* 小さい確保 */
    void *b = malloc(1000);        /* もう一つ */
    void *c = malloc(1 << 20);     /* 1 MiB の大きい確保 */
    free(c); free(b); free(a);
    return 0;
}
```

```console
$ gcc -O0 demo.c && strace -e brk,mmap,munmap ./a.out
...（プログラム起動時のローダによる mmap が並ぶ）...
brk(NULL)                       = 0x55ad34f96000
brk(0x55ad34fb7000)             = 0x55ad34fb7000
mmap(NULL, 1052672, PROT_READ|PROT_WRITE,
     MAP_PRIVATE|MAP_ANONYMOUS, -1, 0) = 0x7f1d2c4e1000
munmap(0x7f1d2c4e1000, 1052672) = 0
+++ exited with 0 +++
```

読み取れることが 3 つある。

1. 最初の `malloc(1000)` で `brk` が一度だけ呼ばれ、ヒープが
   **132 KiB**（0x21000）伸びた。1000 バイトのために 132 KiB 仕入れる、
   まとめ買いである。2 個目の `malloc(1000)` ではシステムコールが起きない。
   在庫から切り分けるだけだからだ。
2. 1 MiB の確保は `brk` のヒープを使わず、**個別の `mmap`** になった
   （前章で説明した `M_MMAP_THRESHOLD` 越えの直行ルート）。
   サイズが 1052672 = 1 MiB + 4 KiB なのは、16 バイトのチャンクヘッダを足して
   4 KiB ページに切り上げた結果、ちょうど 1 ページ分（4 KiB、計 257 ページ）増えたためである。
3. その `free(c)` は即座に `munmap` になった。一方、小さい方の
   `free(b)`/`free(a)` では何も起きない。チャンクはアロケータの
   フリーリスト（[次の部](part-algorithms.md)の主題！)に戻っただけだ。

このしきい値は固定ではない。glibc tunable `MALLOC_MMAP_THRESHOLD_` で動かせる。200 KiB の確保を例にとると、既定ではしきい値（既定 128 KiB）を超えるので mmap 直行になる。

```console
$ strace -e mmap,munmap ./thr     # malloc(200*1024); free();
mmap(NULL, 208896, PROT_READ|PROT_WRITE, MAP_PRIVATE|MAP_ANONYMOUS, -1, 0) = 0x73e34414e000
munmap(0x73e34414e000, 208896)          = 0
```

しきい値を 1 MiB に上げると、同じ 200 KiB が今度は `brk` のヒープから切り出され、`mmap` は消える。

```console
$ MALLOC_MMAP_THRESHOLD_=1048576 strace -e mmap,munmap ./thr
（208896 の mmap/munmap は無し → brk 経由）
```

サイズが 208896 = 200 KiB + 16 バイトのヘッダを 4 KiB ページに切り上げた値（計 51 ページ）であることも、実験 2 と同じ理屈だ。「大きいかどうか」の境目はアロケータの方針にすぎず、環境変数 1 つで挙動が裏返る。`free` が `munmap` になるか何も起きないかも、この境目で決まっている。

「malloc はシステムコールではない」という[最初の章](introduction.md)の主張が、
そのまま画面に映っている。

## 実験 3: free してもメモリ使用量が減らない、を再現する

「大量に `free` したのに `top` の RSS が減らない」は、
malloc に関する質問の定番中の定番だ。再現してみよう。

```c
#include <stdio.h>
#include <stdlib.h>

static void show_rss(const char *const label) {
    char buf[256];
    FILE *const f = fopen("/proc/self/status", "r");
    while (fgets(buf, sizeof(buf), f))
        if (buf[0] == 'V' && buf[1] == 'm' && buf[2] == 'R')  /* VmRSS */
            printf("%s: %s", label, buf);
    fclose(f);
}

int main(void) {
    enum { N = 100000 };
    static char *p[N];
    for (int i = 0; i < N; i++) { p[i] = malloc(100); p[i][0] = 1; }
    show_rss("100000 x malloc(100) 後");
    for (int i = 0; i < N; i++) free(p[i]);
    show_rss("全部 free した後      ");
    return 0;
}
```

実行例:

```
100000 x malloc(100) 後: VmRSS:     13580 kB
全部 free した後      : VmRSS:     13580 kB
```

全部 `free` したのに RSS は 1 バイトも減らない。
理由はこの 2 章で学んだことの組み合わせで説明できる。
(1) 100 バイト級のチャンクはアロケータの在庫（フリーリスト）に戻るだけで、
OS への返却は行われない。次の `malloc` で再利用するためだ。
(2) glibc のメインヒープは `brk` ベースなので、末尾以外は構造的に返せない。
試しに `free` ループの後に `malloc_trim(0);` を呼ぶと、
このプログラムでは RSS が数 MB 減るのが観察できる（ヒープがきれいに空くため。
実アプリでは生き残りが散在して、こうはいかないことが多い。それが
[断片化の章](fragmentation.md)のテーマだ）。

> [!TIP]
> 「在庫」をどれくらい持っているかは `malloc_stats()`（stderr に出力）や
> `malloc_info()` で覗ける。jemalloc なら `malloc_stats_print()`、
> TCMalloc なら `MallocExtension` と、主要アロケータはみな統計 API を持つ。
> 測定 API の使い方は[付録のツール章](tools.md)にまとめた。

## 実験 4: アロケータを差し替えてみる

malloc はただのライブラリ関数なので、**動的リンクの仕組みで丸ごと差し替えられる**。
`LD_PRELOAD` 環境変数で先にロードしたライブラリのシンボルが優先される
（シンボル解決の割り込み、interposition）ことを利用する。

```console
$ sudo apt install libjemalloc2
$ LD_PRELOAD=/usr/lib/x86_64-linux-gnu/libjemalloc.so.2 ./a.out
$ # Ruby だってそのまま差し替わる
$ LD_PRELOAD=/usr/lib/x86_64-linux-gnu/libjemalloc.so.2 ruby app.rb
```

再コンパイルなしでプロセス全体（C 拡張も含む）のアロケータが替わる。
同じプログラムでアロケータだけ替えて RSS や実行時間を比べる実験は、
本書を読み進めるうえで最良の演習になる。
CRuby には `./configure --with-jemalloc` というビルドオプションもあり、
Rails アプリのメモリ削減手法として広く使われてきた
（なぜ効くのかは[マルチスレッドの章](multithread.md)と [jemalloc の章](jemalloc.md)で
説明できるようになる）。

差し替えが効いているかは、実験 1 の `malloc_usable_size` を `LD_PRELOAD` 付きで走らせるだけで分かる。丸め方(サイズクラス)がアロケータごとに違うからだ。

| `malloc(n)` | glibc usable | jemalloc usable |
|---:|---:|---:|
| 16 | 24 | 16 |
| 17 | 24 | 32 |
| 33 | 40 | 48 |
| 100 | 104 | 112 |
| 128 | 136 | 128 |
| 1000 | 1000 | 1024 |

```console
$ LD_PRELOAD=/usr/lib/x86_64-linux-gnu/libjemalloc.so.2 ./a.out
```

同じ `malloc(128)` でも glibc は 136、jemalloc はぴったり 128 を返す。glibc は「8 バイトヘッダ＋16 刻み」、jemalloc は別系統のサイズクラス表に従っているからだ(その設計は [jemalloc の章](jemalloc.md)で見る)。どちらが得かはアプリの確保サイズ分布しだいで、まさにこれが差し替えて測る価値である。

## 観察から分かったこと

4 つの実験から、glibc malloc について次の事実が分かった。

| 観察 | 背後にある仕組み | 詳しく学ぶ章 |
|------|----------------|------------|
| アドレスが 16/32 バイト刻み | チャンクヘッダ＋アラインメント | [free-list](free-list.md), [glibc-malloc](glibc-malloc.md) |
| 小さい確保でシステムコールが起きない | まとめ買いと在庫（フリーリスト） | [free-list](free-list.md) |
| 大きい確保は mmap 直行・即返却 | mmap しきい値の二段構え | [os-interface](os-interface.md), [glibc-malloc](glibc-malloc.md) |
| free しても RSS が減らない | 在庫保持＋ brk の構造的制約 | [fragmentation](fragmentation.md) |
| LD_PRELOAD で丸ごと差し替え可能 | malloc は普通のライブラリ関数 | [第 IV 部](part-libraries.md) |

ここからは、「在庫管理」の中身（返ってきたメモリをどう記録し、
次の要求にどれを充てるか）を体系的に学ぶ。[第 II 部](part-algorithms.md)へ進もう。

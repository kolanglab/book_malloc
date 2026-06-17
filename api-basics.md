# malloc API の仕様

`malloc` は誰でも知っている関数だが、その仕様を正確に言える人は意外と少ない。
「`malloc(0)` は何を返すか」「`free` を 2 回呼ぶとどうなるか」、
そして「`realloc` が失敗したら元のポインタはどうなるか」。
この章では C 標準（および POSIX、glibc 拡張）が定める**契約**を一つずつ確認する。
アロケータを実装・解析する側に回ると、この契約こそが設計の自由度を決めるからだ。

## 4 つの基本関数

C 標準ライブラリの動的メモリ管理は、次の 4 関数からなる（`<stdlib.h>`）。

```c
void *malloc(size_t size);
void free(void *ptr);
void *calloc(size_t nmemb, size_t size);
void *realloc(void *ptr, size_t size);
```

**`malloc(size)`** は `size` バイト以上の領域を確保し、その先頭を指すポインタを返す。
失敗すると `NULL` を返す（POSIX では加えて `errno` に `ENOMEM` を設定する）。
返された領域の中身は不定である。0 で埋まっていると思ってはいけない
（実際、後で見るように「たまたま 0 のことが多い」のがタチが悪い）。

**`free(ptr)`** は `malloc` 系関数が返したポインタを受け取り、領域を解放する。
`free(NULL)` は何もしない、と標準で保証されている。これは覚えておくと
エラー処理パスでの `if (p) free(p);` という冗長な分岐を消せる。

**`calloc(nmemb, size)`** は `nmemb × size` バイトを確保し、**0 で初期化して**返す。
乗算のオーバーフローを検査する義務が実装側にあるのが、`malloc` との大きな違いだ。

```c
/* 危険: n * sizeof(int) が size_t を桁あふれすると小さい領域が返る */
int *a = malloc(n * sizeof(int));

/* 安全: あふれる場合 calloc は NULL を返す */
int *b = calloc(n, sizeof(int));
```

`n` が攻撃者の制御下にあるとき、前者は「小さく確保して大きく書き込む」
ヒープオーバーフロー（[セキュリティの章](security.md)参照）の入口になる。

実際に桁あふれさせると差がはっきり出る。`SIZE_MAX / 4 + 4` 個の `int`（各 4 バイト）を要求してみよう。

```c
size_t n = (SIZE_MAX / 4) + 4;           /* n * 4 が size_t を一周する */
printf("n*4 (wrapped) = %zu bytes\n", n * 4);
void *m = malloc(n * 4);                  /* 一周した小さい値で確保 */
void *c = calloc(n, 4);                   /* 同じ要求を calloc で */
printf("malloc(n*4)  = %p\n", m);
printf("calloc(n, 4) = %p\n", c);
```

```
n*4 (wrapped) = 12 bytes
malloc(n*4)  = 0x5ff31cb032b0   <- わずか 12 バイトを確保して成功
calloc(n, 4) = (nil)            <- オーバーフローを検出して NULL
```

`malloc` 版は「巨大配列を確保したつもり」で 12 バイトしか持っていない。ここに `n` 要素ぶん書き込めば、確保領域の外へ一気にあふれる。`calloc` は乗算の桁あふれを検査して `NULL` を返すので、この罠を踏まない（glibc 2.39 で確認）。

**`realloc(ptr, size)`** は確保済み領域のサイズを変更する。
中身は新旧サイズの小さい方まで保存される。
「その場で広げられればポインタはそのまま、無理なら新領域にコピーして移動」
という動作になるため、戻り値が `ptr` と異なりうる。ここに定番のバグがある。

```c
p = realloc(p, new_size);   /* 失敗すると NULL が返り、元の p が迷子に（リーク） */

/* 正しい書き方 */
void *const q = realloc(p, new_size);
if (q == NULL) { /* p はまだ有効。エラー処理してから free(p) */ }
else           { p = q; }
```

`realloc` が失敗（`NULL` を返す）しても**元の領域は解放されない**、
というのが契約のポイントである。

## サイズ 0 と二重解放の境界条件

仕様の隅に、知らないと事故になる規定がいくつかある。

**`malloc(0)`** の結果は実装定義で、「`NULL` を返す」か
「`free` できる一意な非 NULL ポインタを返す」かのどちらかである。
glibc は後者で、最小サイズのチャンクを実際に確保して返す。
したがって `malloc(0)` の戻り値が `NULL` かどうかでエラー判定をすると、
環境によって挙動が変わってしまう。

「一意な」というのは飾りではない。glibc では `malloc(0)` を 2 回呼ぶと、別々のアドレスが返る。

```c
void *a = malloc(0), *b = malloc(0);
printf("a=%p b=%p (a==b? %d)\n", a, b, a == b);
printf("usable(a)=%zu\n", malloc_usable_size(a));
free(a); free(b);
```

```
a=0x55bfa54922a0 b=0x55bfa54922c0 (a==b? 0)
usable(a)=24
```

アドレスは環境により変わるが、`a != b` であること、そして `malloc_usable_size` が 0 ではなく最小チャンク（この環境で 24 バイト）を示すことは安定している。つまり `malloc(0)` の戻り値は「`free` 必須の、れっきとした確保済み領域」であって、解放を忘れればリークする。

**`realloc(ptr, 0)`** はさらに危うい。C17 までは実装定義
（glibc は `free(ptr)` 相当＋`NULL` または最小確保）だったが、
C23 では未定義動作に格上げされた。移植性が必要なら
`realloc` にサイズ 0 を渡すのは避け、`free` を明示的に呼ぶべきだ。

**二重解放（double free）**と**解放済み領域の使用（use-after-free）**は未定義動作である。
「未定義」とは「クラッシュするかもしれない」ではなく
「何が起きてもよい」という意味で、実際には攻撃者がヒープの内部構造を
乗っ取る足がかりになる（[セキュリティの章](security.md)で仕組みごと説明する）。
`free` 後にポインタへ `NULL` を代入する習慣は、二重解放を
「`free(NULL)` は無害」という規定に変換する古典的な防御である。

「未定義動作」と言われてもピンと来ないので、glibc で実際に二重解放してみる。

```c
void *p = malloc(32);
free(p);
free(p);   /* double free */
```

```console
$ gcc -O0 -Wall df.c   # コンパイル時点で警告が出る
df.c:5:5: warning: pointer ‘p’ used after ‘free’ [-Wuse-after-free]
    5 |     free(p);   /* double free */
      |     ^~~~~~~
$ ./a.out
free(): double free detected in tcache 2
Aborted (core dumped)
```

glibc は直近に解放したチャンクを覚えており（tcache）、同じものをもう一度 `free` すると `double free detected in tcache 2` でその場でアボートする。これは glibc が親切に検査してくれている場合の話で、規格上は「何が起きてもよい」。別のアロケータや別の解放順序では、何事もなく続行してずっと後で壊れることもある。だからこそ `free` 後の `NULL` 代入が効く。

> [!WARNING]
> `malloc` が返した領域の**外**への読み書きは、たとえ 1 バイトでも未定義動作である。
> 多くのアロケータは管理情報（ヘッダ）を確保領域の直前に置くため、
> 「1 バイトはみ出すだけ」でアロケータの管理構造が壊れる。
> 壊れた管理構造によるクラッシュは**ずっと後**の無関係な `malloc`/`free` で起きるので、
> 原因究明が非常に難しい。[ツールの付録](tools.md)で対策を扱う。

## なぜ 16 バイト境界なのか

`malloc` の戻り値は、「あらゆる基本型に対して適切にアラインされている」ことが
保証される（C11 でいう `max_align_t` のアラインメント）。
x86-64 の Linux/glibc では 16 バイト境界である。
`long double` や、SSE 命令で使う 16 バイトベクタ型がこの境界を要求するためだ。

この保証はアロケータの設計に直接響く。どんなに小さい要求でも
戻り値は 16 の倍数のアドレスでなければならないので、
**確保サイズは事実上 16 バイト刻み**になり、`malloc(1)` でも
内部的には数十バイトを消費する（正確な値は[ヒープを覗く章](inside-malloc.md)で実測する）。
この「丸めによる無駄」は内部断片化と呼ばれ、
[断片化の章](fragmentation.md)の主題の一つである。

より大きなアラインメントが必要なとき（ページ境界、キャッシュライン境界など）のために、
専用の API がある。

```c
#include <stdlib.h>
void *aligned_alloc(size_t alignment, size_t size);          /* C11 */
int   posix_memalign(void **memptr, size_t alignment, size_t size); /* POSIX */
```

`aligned_alloc` は C11 で標準化された。`alignment` は 2 の冪、かつ
`size` は `alignment` の倍数でなければならない（glibc は後者を検査しないが、
規格上は守るべきである）。`posix_memalign` は結果をポインタ引数で返し、
戻り値はエラー番号という、一風変わったインターフェースを持つ
（`errno` を変更しないことが保証されている）。
どちらで確保した領域も、普通に `free` で解放できる。

この「戻り値がエラー番号」という規約は実際に動かすと腑に落ちる。

```c
void *p = NULL;
errno = 0;
int rc = posix_memalign(&p, 4096, 100);   /* 成功時 rc==0、p に結果 */
printf("ok:      rc=%d errno=%d p=%p  p%%4096=%zu\n", rc, errno, p, (size_t)p % 4096);

void *q = NULL;
int rc2 = posix_memalign(&q, 24, 100);     /* 24 は 2 の冪でない -> EINVAL */
printf("invalid: rc=%d (EINVAL=%d) q=%p\n", rc2, EINVAL, q);
```

```
ok:      rc=0 errno=0 p=0x60aaa1075000  p%4096=0
invalid: rc=22 (EINVAL=22) q=(nil)
```

成功すれば `0` を返してポインタ引数に結果を書き、失敗すればエラー番号（ここでは `EINVAL` = 22）を**戻り値として**返す。`errno` には触れない。`malloc` のように戻り値で成否を見る癖のまま `if (posix_memalign(...))` と書くと、成功（0）が偽になって判定が逆転しやすいので注意したい。

## 標準の外側にある拡張 API

実務でよく見る非標準 API も押さえておこう。移植性はないが、
アロケータの内部を理解する手がかりとして本書でも使う。

- `malloc_usable_size(ptr)`（glibc）: そのポインタの**実際に使える**サイズを返す。
  要求サイズより大きいことが多く、丸めの実態が観察できる。
- `malloc_trim(pad)`（glibc）: ヒープ末尾などの未使用メモリを OS に返すよう試みる。
- `malloc_stats()` / `malloc_info(0, stream)`（glibc）: 内部統計を表示する。
  [glibc malloc の章](glibc-malloc.md)で読み方を説明する。
- `mallopt(param, value)`（glibc）: `M_MMAP_THRESHOLD` などの内部しきい値を変更する。
- `reallocarray(ptr, nmemb, size)`（OpenBSD 発祥、glibc 2.26+）:
  オーバーフロー検査つき `realloc`。`calloc` の検査を `realloc` にも、という発想だ。
- `strdup(s)`（POSIX、C23 で標準化）: `malloc` + `strcpy` の合成。
  「`malloc` した側と `free` する側が別の関数になる」API 設計の代表例でもある。

また、C++ の `new`/`delete` は多くの処理系で内部的に `malloc`/`free` を呼ぶ。
したがって本書の内容は C++ プログラムにもほぼそのまま当てはまるし、
jemalloc などへの差し替え（[jemalloc の章](jemalloc.md)）も `new` ごと効く。

## 所有権という見方

API の最後に、仕様ではなく規律の話をしておきたい。
`malloc`/`free` の最大の難点は、「どのポインタを・誰が・いつ `free` するか」を
コンパイラが一切検査してくれないことである。
この「誰が解放する責任を持つか」を**所有権**（ownership）と呼ぶ。

```c
/* この関数の契約はどっち？
   (a) 戻り値の所有権は呼び出し側に移る（呼び出し側が free する）
   (b) 内部バッファを貸しているだけ（free してはいけない） */
char *get_name(void);
```

C ではこの契約をコメントと命名規則でしか表現できない。
所有権をコンパイル時に検査する Rust や、所有権の管理自体を消し去る GC 言語は、
いずれもこの問題への言語側からの回答である。
逆に言えば、**malloc API の使いにくさこそが、メモリ管理研究と
言語設計を駆動してきた**。Ruby 処理系の中を見ても、`xmalloc` のラッパで
確保量を集計して GC と連動させる、確保したメモリをオブジェクトの寿命に
ひもづける（`RSTRING` の中身は `String` オブジェクトが「所有」する）など、
所有権を処理系の規約として管理する工夫が詰まっている。

> [!NOTE]
> カスタムアロケータ（領域ごと一括解放する arena/region アロケータなど）は
> 所有権管理を単純化する古典的な技法だが、Berger らの測定によれば
> 「速さ」を理由にした自作アロケータの大半は汎用アロケータに勝っておらず、
> 一括解放できる region 型だけが明確に有利だった[](#cite:berger2002)。
> 「malloc が遅いから自作する」前に、この論文を読む価値がある。

## 契約のまとめ

`malloc` API の契約を整理した。ポイントは、
(1) 失敗は `NULL` で通知され、`realloc` の失敗時も元領域は生きていること、
(2) サイズ 0 や二重解放など、境界条件の多くが実装定義か未定義であること、
(3) アラインメント保証が確保サイズの最小単位を決めること、
(4) 所有権の管理は完全にプログラマの責任であること、の 4 つだ。

ところで、`malloc` はこのメモリをそもそもどこから持ってくるのだろうか。
次の [OS とのインターフェースの章](os-interface.md)で、卸売業者（カーネル）との
取引を見に行こう。

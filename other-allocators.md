# その他のアロケータたち

主要 5 実装（[dlmalloc](dlmalloc.md)・[glibc](glibc-malloc.md)・
[jemalloc](jemalloc.md)・[TCMalloc](tcmalloc.md)・[mimalloc](mimalloc.md)）の
ほかにも、目的に特化した現役アロケータは数多い。
この章では、それらを「何を最優先したか」という軸で概観する。
1 つの正解はなく、**ワークロードと制約が違えば最適なアロケータも違う**。
本書全体の主張を、実例の多様さで締めくくりたい。

## 小ささを取る musl の malloc

**musl** は、glibc に代わる軽量な C 標準ライブラリで、
Alpine Linux など小さなコンテナイメージで広く使われる。
その malloc は、コードの小ささと素直さを重視してきた。
初期実装（dlmalloc 系）は単純だが、マルチスレッドでの断片化に
弱点があった。そこで作者の Rich Felker は 2020 年、
**mallocng**（next-generation malloc）に置き換えた[](#cite:felker2020)。

mallocng は、サイズクラスごとに**グループ**（同サイズ枠の集合）を持ち、
メタデータをユーザデータから分離し、強い整合性検査を入れることで、
小さなコードのまま断片化耐性とセキュリティを底上げした。
「巨大で速い」より「小さく堅い」を選ぶ設計で、
コンテナや組み込み Linux のようにフットプリントが効く環境に向く。
glibc malloc しか知らない人が musl 環境で
「同じプログラムなのにメモリ挙動が違う」と驚くのは、この違いのためだ。

> [!NOTE]
> 「挙動が違う」の中身を一つ挙げると、**スレッドごとの arena** の有無がある。glibc malloc はマルチスレッドのロック競合を避けるため arena を CPU 数に応じて自動増殖させ（既定で最大 `8 * nproc` 個）、その分だけ仮想メモリと断片化が膨らみうる。musl の mallocng は arena 増殖という戦略を取らず、メタデータをユーザデータから分離した素直な構成のままスレッド競合をさばく。だから同じ多スレッドプログラムでも、Alpine（musl）と通常の glibc イメージで RSS や `/proc/<pid>/maps` の様子が変わる。glibc 側は `MALLOC_ARENA_MAX=1` を与えると arena を 1 個に固定でき、musl に近い挙動で比較できる。

## 安全を取る Scudo と hardened_malloc

[セキュリティの章](security.md)の防御を、
本番投入できる速度で実装した実用アロケータがいくつかある。

**Scudo** は LLVM プロジェクトのアロケータで[](#cite:llvmscudo)、
Android のシステムアロケータとして全面採用されている。
チャンクヘッダのチェックサム、確保のランダム化、
解放チャンクの隔離（quarantine）、サイズクラス分離などで、
ヒープオーバーフローや use-after-free を緩和する。
スマートフォン全体を守る必要から、強い防御と実用的な速度の両立が
設計目標になっている。

**hardened_malloc** は、GrapheneOS の Daniel Micay による
セキュリティ最優先のアロケータである[](#cite:micay2024)。
[セキュリティの章](security.md)で見た DieHarder 系の原則
（メタデータの完全分離、確保の強いランダム化、
ガードページ、解放の遅延と検証）を妥協なく実装する。
速度よりも攻撃耐性を取る明確な立場で、
攻撃対象になりやすい環境（OS、ブラウザ、機微なサービス）向けだ。
[セキュリティの章](security.md)の理論が、そのまま製品になった例として読める。

## ブラウザの要求に応える PartitionAlloc

Web ブラウザは、攻撃者が JavaScript で確保パターンを
自在に操れる、極めて敵対的な環境である（[セキュリティの章](security.md)）。
Chromium の **PartitionAlloc**[](#cite:partitionalloc) は、
この要求に応えるアロケータだ。

中心概念は **partition**（区画）で、
**用途や型ごとにヒープを物理的に分ける**。
たとえば「DOM ノード用」「文字列バッファ用」を別パーティションにすれば、
あるオブジェクトの use-after-free で別種のオブジェクトを
掴ませる攻撃（type confusion）が成立しにくくなる。
[セキュリティの章](security.md)の type-after-type 的な発想を、
区画の分離で実現しているわけだ。
これに加え、サイズクラス・スレッドキャッシュ・
強い整合性検査といった、本書で見てきた部品を
ブラウザの安全要件に合わせて組み合わせている。

## 用途に特化した region、obstack、Hoard

汎用 malloc とは別の系譜として、**一括解放**を前提にした
アロケータがある。[API の章](api-basics.md)で触れた region（arena）型だ。
GNU の **obstack**（オブジェクトスタック）はその古典で、
スタック状に積んで一気に巻き戻す。
コンパイラの 1 パスや 1 リクエストの処理のように
「フェーズの終わりに全部捨てる」用途で、`free` の管理を消し去り、
Berger らが「明確に速い唯一のカスタムアロケータ類型」と
位置づけたもの[](#cite:berger2002)である。

obstack は glibc に同梱されており、その挙動は数行で確かめられる。`obstack_copy0` で文字列を次々に積み、返ってきたポインタの差を見ると、各オブジェクトが連続領域に詰めて置かれていることがわかる（`free` は一度も呼ばない）。

```c
#include <obstack.h>
#include <stdio.h>
#include <stdlib.h>
#define obstack_chunk_alloc malloc
#define obstack_chunk_free  free
int main(void) {
    struct obstack ob; obstack_init(&ob);
    char *a = obstack_copy0(&ob, "foo", 3);     /* "foo\0"    */
    char *b = obstack_copy0(&ob, "barbaz", 6);  /* "barbaz\0" */
    char *c = obstack_copy0(&ob, "qux", 3);
    printf("b - a = %ld\n", (long)(b - a));
    printf("c - b = %ld\n", (long)(c - b));
    obstack_free(&ob, NULL);   /* フェーズの終わりに一括で巻き戻す */
    return 0;
}
```

```
b - a = 16
c - b = 16
```

隣り合うオブジェクトの間隔は確保サイズ（4、7 バイト）そのものではなく、既定アライメントに丸めた 16 バイトになっている。確保はポインタを進めるだけ、解放は最後の `obstack_free` 一回で chunk ごとまとめて返す。この「ポインタを進めるだけ／個別 `free` なし」が、region 型が汎用 malloc より明確に速い理由である。

研究の起点として外せないのが **Hoard**[](#cite:berger2000) だ。
[マルチスレッドの章](multithread.md)で詳しく見たとおり、
スーパーブロック単位でスレッドに割り当てて偽共有を防ぎ、
blowup を定数倍に抑える設計で、
jemalloc・TCMalloc・mimalloc すべての先祖にあたる。
今日のマルチスレッドアロケータの教科書的原型として、
歴史的価値とともに今も参照される。なお Berger らは、こうしたアロケータを再利用可能な部品の組み合わせとして構築する枠組み（Heap Layers）も提案しており[](#cite:berger2001)、本書が部品ごとに分解してきた見方の理論的な裏づけにもなっている。

## メッセージパッシングで解く snmalloc

[マルチスレッドの章](multithread.md)で触れた
**snmalloc**[](#cite:lietar2019) は、Microsoft Research 発の、
遠隔解放をメッセージパッシングで扱うアロケータである。
生産者・消費者型（あるスレッドが確保し別スレッドが解放する）の
ワークロードに特化し、解放を所有スレッドへの
バッチ化メッセージとして送る。
mimalloc の `thread_free`（[mimalloc の章](mimalloc.md)）と
問題意識は同じだが、「メッセージキュー」という抽象で
正面から設計した点が独特で、特定の通信パターンで高い性能を出す。

## OS が提供するヒープ（Windows と macOS）

最後に、OS 標準のヒープにも触れておく。
[OS とのインターフェースの章](os-interface.md)で見たとおり、
Windows ではユーザ向けヒープ API `HeapAlloc`/`HeapFree`（プロセスヒープ）が
あり、C の `malloc` はその上に乗る。Windows のヒープは
[セキュリティの章](security.md)的な強化（LFH=Low Fragmentation Heap、
各種の整合性検査）を OS 側で積んできた歴史を持つ。
macOS の標準アロケータ（libmalloc の `magazine_malloc`）も、
スレッドごとの magazine（[スラブの章](buddy-slab.md)の Bonwick 由来
[](#cite:bonwick2001)）を中心にした設計で、
本書で学んだ語彙がそのまま通じる。
プラットフォームが変わっても、部品は共通なのだ。

## 一枚の地図としての本書のまとめ

ここまで見てきたアロケータを、優先順位の軸で並べると、
本書全体が一枚の地図になる。

| アロケータ | 最優先したもの | 効く場面 |
|-----------|--------------|---------|
| [dlmalloc](dlmalloc.md) | 単純さ・基準形 | 学習、単一スレッド、組み込み |
| [glibc malloc](glibc-malloc.md) | 汎用性・互換性 | Linux の既定、あらゆる用途 |
| [jemalloc](jemalloc.md) | 断片化と観測可能性 | 高並列サーバ、Ruby/Rails |
| [TCMalloc](tcmalloc.md) | 多数スレッド・ヒュージページ | 大規模 C++ サーバ |
| [mimalloc](mimalloc.md) | 速いパスの軽さ | 汎用高速、.NET、Rust |
| musl mallocng | 小ささ・堅さ | コンテナ、組み込み |
| Scudo / hardened_malloc | 安全性 | OS、機微なサービス |
| PartitionAlloc | 型分離の安全性 | ブラウザ |
| TLSF（[realtime](realtime.md)）| 最悪時間の保証 | リアルタイム・組み込み |
| region / obstack | 一括解放の速さ | コンパイラ・1 リクエスト処理 |

どれも「速度・メモリ・安全性・予測可能性・単純さ」という
複数の価値のどこかを選び、どこかを諦めた結果である。
[最初の章](introduction.md)で述べた
「将来の確保パターンを知らないまま 3 つ（速く・無駄なく・行儀よく）を
両立する」という不可能な要求に、それぞれの流儀で答えているのだ。

malloc に唯一の正解はない。
あなたがこれから言語処理系やシステムソフトウェアを書くとき、
「自分のワークロードは何を最優先すべきか」を考え、
本書の地図の上で適切なアロケータを選び、あるいは設計できるように
なっていれば、本書の目的は果たされている。
気になったアロケータは、ぜひソースを読み、`LD_PRELOAD` で動かし、
[付録のツール](tools.md)で測ってみてほしい。


最後に、その一歩を実際にやってみよう。ビルド済みのプログラムを再コンパイルせず、`LD_PRELOAD` で jemalloc に差し替え、`MALLOC_CONF=stats_print:true` で終了時に内部統計を吐かせる。48 バイトを 1000 個確保するだけのプログラムでも、jemalloc がそれを**サイズクラス（bin）**にまとめている様子が読み取れる。

```
$ LD_PRELOAD=/usr/lib/x86_64-linux-gnu/libjemalloc.so.2 \
    MALLOC_CONF=stats_print:true ./a.out 2>&1 | grep -E 'bins:|^ +48 '
bins:           size ind    allocated      nmalloc ...  regs pgs   util ...
                  48   3        48000         1000 ...   256   3  0.976 ...
```

要求した 48 バイトはサイズクラス番号 `ind=3`（サイズ 48）に落ち、1 スラブあたり `regs=256` 個・`pgs=3` ページ（=12KiB／256 ≒ 48 バイト）という、[jemalloc の章](jemalloc.md)で見たレイアウトがそのまま観測できる。1000 個ぶんで `allocated=48000`（=1000×48 バイト）、`util=0.976` は 4 スラブ＝1024 領域中 1000 個が使われている割合だ。ソースを読み、`LD_PRELOAD` で動かし、統計を覗く。本書の地図を、こうして自分の手で確かめてほしい。
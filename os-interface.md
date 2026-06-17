# OS とのインターフェース

アロケータの設計は、その「仕入れ先」であるカーネルとの取引条件に強く規定される。
この章では、malloc の足元にある仮想メモリの仕組みと、
メモリを調達・返却するためのシステムコール群を学ぶ。
ここを押さえると、第 IV 部で各ライブラリの設計判断
（「128 KiB 以上は mmap 直行」「返却は madvise で」など）が自然に読めるようになる。

## 仮想メモリとページ

現代の OS では、プロセスが見るアドレスは**仮想アドレス**である。
カーネルと CPU の MMU（Memory Management Unit; アドレス変換を行うハードウェア）が、
仮想アドレスを物理メモリのアドレスへ対応づける。
対応づけの単位が[ページ](#index:ページ)（page）で、x86-64 Linux の標準は 4 KiB だ。

アロケータにとって重要な性質が 3 つある。

第一に、仮想アドレス空間は広大で、予約だけならタダ同然である。
64 ビット環境のユーザ空間は実質 47 ビット（128 TiB）以上あり、
「とりあえず広い範囲を確保しておき、使った分だけ物理メモリを割り当てる」
という戦略が成立する。後述の**デマンドページング**がこれを支える。

第二に、物理メモリの割り当ては「最初に触ったとき」に起きる。
`mmap` で領域を作った時点ではページテーブルに「ここは有効」と記録されるだけで、
実際に読み書きした瞬間に**ページフォルト**（page fault）が発生し、
カーネルがその場で物理ページを割り当てる（demand paging）。
さらに Linux では、書き込む前の匿名ページはすべて
カーネル内の単一の「ゼロページ」を指しており、読み出しだけなら物理メモリを消費しない。
`malloc` 直後のメモリが「たまたま 0 に見える」ことが多いのはこのためだ
（ただし再利用された領域は前の中身が残る。0 を当てにしてはいけない）。

これを数値で見てみよう。256 MiB を `mmap` し、全ページに 1 バイトずつ書き込み、最後に `MADV_DONTNEED` で物理ページを破棄する間、`/proc/self/status` の `VmRSS` を観察する（Linux x86-64、ページ 4 KiB）。

```c
char *p = mmap(NULL, 256UL<<20, PROT_READ|PROT_WRITE,
               MAP_PRIVATE|MAP_ANONYMOUS, -1, 0);
for (size_t i = 0; i < (256UL<<20); i += 4096) p[i] = 1; /* 全ページに触る */
madvise(p, 256UL<<20, MADV_DONTNEED);
```

```
before mmap   : VmRSS=1352 kB
after mmap    : VmRSS=1612 kB   (256 MiB 予約済み)
after touch   : VmRSS=263756 kB  (全ページに書き込み後)
after DONTNEED: VmRSS=1612 kB
```

`mmap` した瞬間は RSS がほとんど増えない。`VmSize`（VSZ）には 256 MiB が乗っているが、物理メモリは 1 バイトも消費していない。書き込んで初めて約 256 MiB ぶんの物理ページが割り当たり、`MADV_DONTNEED` で一気に元へ戻る。これが「確保」と「物理メモリ消費」が別イベントだ、という話の正体である（RSS の絶対値は環境により変わる）。

第三に、ページ単位でしか操作できない。保護属性の変更（`mprotect`）も
解放（`munmap`）も 4 KiB 単位である。16 バイトの `malloc` 要求と
4 KiB のページの間のギャップを埋めるのが、まさにアロケータの存在意義である。

## brk によるヒープの伸縮

Unix の伝統的なヒープは、データセグメントの末尾を示す
**プログラムブレーク**（program break）を上下させる方式である。

```c
#include <unistd.h>
int brk(void *addr);           /* break を addr に設定 */
void *sbrk(intptr_t increment); /* break を increment だけ動かし、旧値を返す */
```

プロセスのアドレス空間で、ヒープは実行ファイルのデータ領域の直後から
上位アドレスへ向かって伸びる。`sbrk(4096)` と呼べばヒープが 4 KiB 伸び、
負の値を渡せば縮む。実装が単純で、連続領域が手に入るのが利点だ。

弱点は**末尾からしか伸縮できない**ことである。ヒープの途中が空いても、
break より下にあるかぎり OS には返せない。
ヒープのてっぺんに長寿命の 1 ブロックが居座ると、その下がどれだけ空いていても
プロセスのメモリ使用量は減らない。glibc malloc がメインアレナで今も `brk` を
使いつつ、`malloc_trim` で末尾返却を試みる背景がこれだ
（[glibc malloc の章](glibc-malloc.md)）。

> [!NOTE]
> `brk`/`sbrk` は SUSv2 で LEGACY 扱いとなり POSIX.1-2001 で標準から削除された歴史的 API で、
> アプリケーションが直接呼ぶことはまずない。しかし glibc malloc の
> 最初の仕入れ先として現役であり、`strace` を取ると今でも普通に観察できる
> （[次章](inside-malloc.md)で実際にやってみる）。

## 現代の主力としての mmap

現代のアロケータの主力調達手段は `mmap` である。

```c
#include <sys/mman.h>
void *mmap(void *addr, size_t length, int prot, int flags, int fd, off_t offset);
int munmap(void *addr, size_t length);
```

本来はファイルをメモリに対応づける（メモリマップする）ための API だが、
`MAP_ANONYMOUS | MAP_PRIVATE` を指定すると「ファイルの裏付けがない、
0 初期化された専用メモリ」が手に入る。これを**匿名マッピング**と呼ぶ。

```c
void *const p = mmap(NULL, 1 << 20,            /* 1 MiB */
                     PROT_READ | PROT_WRITE,
                     MAP_PRIVATE | MAP_ANONYMOUS, -1, 0);
if (p == MAP_FAILED) { /* errno を確認 */ }
...
munmap(p, 1 << 20);
```

`brk` と比べた利点は、(1) アドレス空間のどこにでも独立した領域を作れる、
(2) `munmap` で**どの領域でも個別に** OS へ返せる、
(3) スレッドごとに別の領域を持てる（マルチスレッド対応の鍵）、の 3 点である。
欠点は、カーネル内の管理構造（VMA; virtual memory area）が領域ごとに作られるため、
細かく大量に呼ぶとカーネル側が重くなることだ。
だからアロケータは「大きめに mmap して小さく切り分ける」。

この「二段構え」は `strace` で直接見える。64 KiB と 200 KiB を 1 回ずつ `malloc` するだけのプログラムを `strace -e trace=brk,mmap,munmap` で走らせると、しきい値の前後で経路が変わる（glibc 2.39）。

```
$ strace -e trace=brk,mmap,munmap ./thresh
...
brk(0x61634f689000)                     = 0x61634f689000          ← 64 KiB はヒープを brk で伸ばす
mmap(NULL, 208896, PROT_READ|PROT_WRITE,
     MAP_PRIVATE|MAP_ANONYMOUS, -1, 0)  = 0x7eae430eb000          ← 200 KiB は個別 mmap 直行
munmap(0x7eae430eb000, 208896)          = 0                       ← free 時に即 munmap
```

64 KiB は既定しきい値（128 KiB）未満なのでヒープ（`brk`）から切り出され、`free` してもプロセスからは返らない。一方 200 KiB は `mmap` で個別に確保され、`free` した瞬間に `munmap` で OS へ返る。`mmap` 長が 208896（= 204 KiB、要求 + ヘッダをページに丸めた値）になっている点にも注目。

glibc malloc は、既定で 128 KiB（動的に調整される）以上の要求を
個別の `mmap` で満たす。大きな確保は `free` 時に `munmap` で即座に OS へ返り、
断片化も起こさない。一方それ未満はヒープから切り分ける。
このしきい値が `M_MMAP_THRESHOLD` であり、ほとんどのアロケータに
同種の「大物は直行、小物は在庫から」という二段構えがある。

## madvise による物理メモリの返却

`munmap` で返すと仮想アドレスごと消えるので、再利用するにはまた `mmap` が要る。
そこで「アドレス空間は持ったまま、物理メモリだけ返す」中間手段が `madvise` である。

```c
int madvise(void *addr, size_t length, int advice);
```

- `MADV_DONTNEED`: その範囲の物理ページを即座に破棄する。次に触ると
  ページフォルトが起き、0 初期化されたページが改めて割り当てられる。
- `MADV_FREE`（Linux 4.5+）: 「もう要らないが、急いで回収しなくてよい」と伝える。
  カーネルはメモリが逼迫するまで回収を遅延し、その間にプロセスが
  再び書き込めば回収は取り消される。`DONTNEED` より速いが、
  回収されるまで RSS（後述）が減って見えないという観測上の罠がある。

この「罠」を同じ 256 MiB の領域で見比べてみよう。書き込んで RSS を約 256 MiB まで上げたあと、片方は `MADV_DONTNEED`、もう片方は `MADV_FREE` を呼ぶ。

```
--- MADV_DONTNEED ---
touch 後                : VmRSS=263500 kB
madvise(MADV_DONTNEED) 後: VmRSS=1616 kB     ← 即座に返る
--- MADV_FREE ---
touch 後                : VmRSS=263496 kB
madvise(MADV_FREE)    後: VmRSS=263756 kB     ← まだ減って見えない
```

`MADV_DONTNEED` は呼んだ瞬間に RSS が落ちるが、`MADV_FREE` はメモリが逼迫するまで回収を遅延するため、`ps` や `/proc/<pid>/status` 上は RSS が高いままに見える（逼迫すれば減る）。jemalloc が `MADV_FREE` を使う設定だと「`free` したのに RSS が減らない」が起きるのはこれが理由で、罠というより仕様である（数値は環境により変わる）。

jemalloc や mimalloc が未使用ページを OS に返す（purge する）ときの
主役はこの `madvise` である。「`free` したのにプロセスのメモリ使用量が
減らない」というよくある疑問の答えの半分は「アロケータが在庫として
持っている」、もう半分は「`MADV_FREE` 済みで、逼迫したら減る」である。

> [!TIP]
> プロセスのメモリ使用量を見るとき、**VSZ**（仮想メモリサイズ）と
> **RSS**（Resident Set Size; 実際に物理メモリに載っている量）を区別しよう。
> mmap で予約しただけの領域は VSZ にしか現れない。
> アロケータの「在庫」は RSS に現れる。`ps`、`top`、`/proc/<pid>/status` の
> `VmSize`/`VmRSS` で確認できる。

## オーバーコミットと OOM killer

Linux の既定では、カーネルは物理メモリ＋スワップを超える量の
匿名メモリ確保を許す（**オーバーコミット**）。
デマンドページングにより、確保しただけで使わないメモリは多いので、
楽観的に許した方が全体効率がよいからだ。

その代償として、「いざ全員が本当に使い始めたら」物理メモリが尽きる。
そのときカーネルは **OOM killer** を起動し、スコアの高いプロセスを強制終了する。
つまり Linux の既定設定では、**`malloc` が `NULL` を返すより先に、
ページフォルトの瞬間にプロセスが殺される**ことがある。
「`malloc` の戻り値検査は無意味」という極論の根拠だが、
`vm.overcommit_memory=2`（厳格モード）の環境や、リソース制限
（`ulimit -v`、cgroup のメモリ上限）下では普通に `NULL` が返る。
検査はやはり書くべきである。

既定の `vm.overcommit_memory=0`（ヒューリスティック）が実際どう振る舞うか見てみよう。物理 30 GiB（空きは 2.8 GiB）＋ swap 8 GiB の環境で、単発の匿名 `mmap` をサイズを変えて試す。

```
物理 RAM 30 GiB（空き 2.8 GiB）+ swap 8 GiB の環境:
mmap(20 GiB) = OK   VmSize=20 GiB / VmRSS≈0
mmap(35 GiB) = OK   VmSize=35 GiB / VmRSS≈0
mmap(50 GiB) = FAILED (ENOMEM)
```

空きが 2.8 GiB しかないのに 20 GiB の確保が通る。触っていない（= 物理ページを要求していない）ので、カーネルは楽観的に許す。一方、RAM＋swap の総量（約 38 GiB）を一発で超える 50 GiB は、ヒューリスティックでも拒否されて `NULL`（ENOMEM）になる。つまり「`malloc` は失敗しない」は半分嘘で、要求が極端に大きければ既定でも普通に失敗する。だから戻り値検査は省けない。

## ヒュージページ

仮想→物理の変換結果は **TLB**（Translation Lookaside Buffer）にキャッシュされるが、
TLB のエントリ数は限られている。4 KiB ページで 64 GiB のヒープを使うと
1600 万ページにもなり、TLB ミスが性能を蝕む。
そこで x86-64 は 2 MiB（および 1 GiB）の[ヒュージページ](#index:ヒュージページ)を
サポートし、Linux は通常ページを自動的に 2 MiB に昇格させる
THP（Transparent Huge Pages）を持つ。

ただし 2 MiB 単位で物理メモリを掴むため、アロケータが 2 MiB の中に
小さなライブオブジェクトを散らしてしまうと、断片化のコストも 512 倍になる。
ヒュージページを意識したアロケータ設計は 2020 年代の主要トピックで、
Google の TEMERAIRE[](#cite:hunter2021) を[局所性の章](locality.md)で詳しく読む。

## Windows と他の OS

本書は Linux 中心だが、構図はどの OS でも同じである。
Windows では `VirtualAlloc`/`VirtualFree` が `mmap`/`munmap` に相当し、
**予約（reserve）とコミット（commit）が API 上も分離**している点が特徴的だ。
ユーザ向けには `HeapAlloc`（プロセスヒープ）があり、
C の `malloc` はその上に実装される（[その他のアロケータの章](other-allocators.md)）。
macOS では `mach_vm_allocate` 系の Mach API と `mmap` が併存する。
アロケータを移植層から見ると「ページ単位の確保・返却・保護変更」が
あればよいので、jemalloc や mimalloc は薄い OS 抽象層を持ち、
ほぼすべての主要 OS で動く。

## この章のまとめ

カーネルが提供するのはページ単位の粗い割り当てであり、
`brk`（末尾伸縮のみ・歴史的）と `mmap`（任意位置・個別返却可・現代の主力）が
調達手段、`munmap` と `madvise` が返却手段である。
デマンドページングとオーバーコミットにより「確保」と「物理メモリ消費」は
別のイベントであり、観測には VSZ/RSS の区別が要る。
そして TLB とヒュージページが、最新のアロケータ設計を動かす制約になっている。

次章では、ここまでの知識を持って、実際に動いている glibc malloc を観察する。

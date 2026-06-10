# 付録: メモリのデバッグと計測ツール

本書の実験では `printf` と `strace` だけを使ってきたが、
実務ではもっと強力な道具がある。この付録では、
ヒープのバグを見つける道具と、性能・メモリを測る道具を、
**何を・どういう原理で検出/計測するか**に重点を置いて紹介する。
道具の出力を正しく読むには、本書で学んだアロケータの仕組みが役に立つ。

## バグを見つける — Sanitizer と Valgrind

[セキュリティの章](security.md)で見たヒープバグ——オーバーフロー、
use-after-free、二重解放、リーク——を検出する二大ツールがある。

**AddressSanitizer（ASan）**[](#cite:serebryany2012) は、
コンパイラに `-fsanitize=address` を付けるだけで使える検出器だ。
原理は **shadow memory** と **red zone** である。
アロケータが返す各ブロックの前後に「触ってはいけない」領域
（red zone; ガード帯）を挟み、メモリ 1 バイトごとに
「アクセス可能か」を記録する影の表（shadow memory）を持つ。
ロード/ストアのたびに影を確認し、red zone や解放済み領域への
アクセスを**その瞬間に**捕まえる。
[ヒープを覗く章](inside-malloc.md)で「ヘッダを 1 バイト踏むと
後でクラッシュする」と述べた問題を、ASan は踏んだ瞬間に止めてくれる。
速度低下は 2 倍程度なのでテスト時に常用でき、
CRuby など多くの言語処理系が ASan ビルドで CI を回している
（[セキュリティの章](security.md)）。

```console
$ gcc -fsanitize=address -g bug.c && ./a.out
==12345==ERROR: AddressSanitizer: heap-use-after-free on address 0x...
  READ of size 4 at 0x... thread T0
    #0 0x... in main bug.c:8
  freed by thread T0 here:
    #1 0x... in main bug.c:6   ← どこで free したかまで分かる
```

解放済み領域への use-after-free を検出するため、ASan は
解放したブロックをすぐ再利用せず**隔離（quarantine）**に置く。
[セキュリティの章](security.md)の MarkUs[](#cite:ainsworth2020) と
同じ「隔離」の発想が、ここでは検出のために使われている。

**Valgrind**（Memcheck）[](#cite:nethercote2007) は、
**再コンパイル不要**で同種のバグを検出する。
原理が異なり、プログラムを**動的バイナリ計装**で
仮想 CPU 上で実行し、すべてのメモリアクセスとビット単位の
「初期化済みか」を追跡する。コンパイル不要・追跡が精密という
利点の代わりに、速度低下は 10〜30 倍と大きい。
「ビルドし直せないバイナリを調べる」「初期化忘れ
（uninitialized read）まで精密に追う」場面で今も価値がある。

> [!TIP]
> ASan と Valgrind は守備範囲が重なるが性格が違う。
> 開発中のコードは ASan（速い・要再コンパイル）、
> 手元にあるバイナリや初期化忘れの調査は Valgrind、と使い分けるとよい。
> 本番環境では、どちらも重すぎるので
> [セキュリティの章](security.md)の GWP-ASan[](#cite:serebryany2023)
> のようなサンプリング検知が使われる。

## リークと肥大を追う — ヒーププロファイラ

「クラッシュはしないがメモリが増え続ける」問題には、
**ヒーププロファイラ**が要る。これは
「**どのコールスタックが、どれだけのメモリを確保し、
解放せずに保持しているか**」を集計する道具だ。

[jemalloc](jemalloc.md) と [TCMalloc](tcmalloc.md) は
プロファイラを内蔵している。jemalloc なら
`MALLOC_CONF=prof:true` で有効化し、`jeprof` で可視化、
TCMalloc なら `pprof` を使う。出力は
「確保バイト数で重み付けしたコールスタックの木」で、
[局所性の章](locality.md)で触れた「どこが確保しているか」が一目で分かる。
[最新研究の章](research-frontiers.md)の LLAMA[](#cite:maas2020) が
学習に使ったのも、本質的にはこの「確保コンテキスト」の情報である。

言語処理系では、処理系自身がプロファイラを提供することも多い。
Ruby なら `ObjectSpace.dump_all` でヒープの全オブジェクトを
JSON で吐き出し、どのファイル・行で生成されたオブジェクトが
生き残っているかを集計できる（`memory_profiler` gem などが
これを使う）。これは Ruby オブジェクトの話だが、
[最初の章](introduction.md)で述べたとおり、その下では
`malloc` が動いている。両方の層を見ると全体像がつかめる。

## アロケータ自身の統計を読む

本書で繰り返し使った、アロケータ内蔵の統計 API をまとめておく。
これらは「在庫をどれだけ持っているか」「断片化しているか」を
アロケータの内側から教えてくれる。

- **glibc**: `malloc_stats()`（stderr へ要約）、
  `malloc_info(0, stream)`（XML で詳細）、`mallinfo2()`（構造体）。
  [glibc malloc の章](glibc-malloc.md)で読み方を説明した。
- **jemalloc**: `malloc_stats_print()`。アリーナ・サイズクラス・
  decay の状況まで出る（[jemalloc の章](jemalloc.md)）。
- **TCMalloc**: `MallocExtension::GetStats()` ほか
  （[TCMalloc の章](tcmalloc.md)）。
- **mimalloc**: `mi_stats_print()`。

「`free` したのに RSS が減らない」を診断する標準手順は、
(1) これらで「使用中は小さいが system/heap は大きい」ことを確認、
(2) 在庫を持っているのか（断片化）、`madvise` 待ちなのか
（[OS とのインターフェースの章](os-interface.md)の `MADV_FREE`）を切り分け、
(3) glibc なら `malloc_trim(0)`、jemalloc なら decay 設定や
`mallctl` で返却を促す、という流れになる。

## プロセス全体のメモリを外から見る

アロケータの外、OS の側から見る道具も押さえておこう。
[OS とのインターフェースの章](os-interface.md)の VSZ/RSS の区別が効く。

- `ps` / `top`: VSZ（仮想サイズ）と RSS（物理常駐）。まず全体感をつかむ。
- `/proc/<pid>/status`: `VmSize`（VSZ）、`VmRSS`（RSS）、
  `VmHWM`（RSS の最高水位）。[ヒープを覗く章](inside-malloc.md)の
  実験 3 で使ったのがこれだ。
- `/proc/<pid>/smaps`: マッピングごとの内訳。
  `AnonHugePages` 行でヒュージページ（[局所性の章](locality.md)）の
  利用を、各 `[heap]`/匿名領域の `Rss` で在庫の所在を確認できる。
- `strace -e brk,mmap,munmap,madvise`: アロケータと OS の
  取引を実況する。[ヒープを覗く章](inside-malloc.md)の実験 2 の道具。

## 計測の心得

最後に、測るときの心得を 3 つ。

第一に、**アロケータを変えて比べる**のが最も学びが多い。
`LD_PRELOAD` でプログラムを変えずに glibc / jemalloc / TCMalloc /
mimalloc を差し替え（[ヒープを覗く章](inside-malloc.md)）、
同じワークロードで RSS と実行時間を測る。
本書の各章の主張を、自分のアプリで検証できる。

第二に、**マルチスレッドは 3 軸で測る**。
[マルチスレッドの章](multithread.md)で述べたとおり、
スレッド数・確保サイズ分布・確保と解放を同/別スレッドでするか、で
結果が激変する。アロケータ付属のベンチ
（`mstress`、`larson`、`xmalloc-test` など）はこの軸を狙っている。

第三に、**最悪も測る**。平均レイテンシだけでなく、
裾（テールレイテンシ）と、長時間稼働後の断片化
（[断片化の章](fragmentation.md)の high-water の伸び）を見る。
[リアルタイムの章](realtime.md)の関心が、汎用サーバでも
テールレイテンシとして効いてくる。

これらの道具と本書の知識があれば、
「なぜこのプログラムはメモリを食うのか」「なぜ遅いのか」を、
推測でなく**観測**で答えられるようになる。それが本書の最終目標である。

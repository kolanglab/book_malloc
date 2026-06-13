# 付録: 用語集

本書に登場する主な用語を、初出の文脈とともにまとめる。
詳しい説明は各章を参照してほしい。アルファベット・五十音の混在を避け、
おおむね登場順・関連順に並べた。

## メモリの基礎

**ヒープ（heap）**
: 実行中に任意のタイミング・任意の順序で確保・解放できるメモリ領域。
  静的領域・スタックと対をなす。本書の主役。→[最初の章](introduction.md)

**動的メモリ割り当て（dynamic memory allocation）**
: 実行時にサイズや寿命の決まるデータのためにメモリを確保・解放する仕組み。
  C では `malloc`/`free` がそのインターフェース。→[最初の章](introduction.md)

**アロケータ（memory allocator）**
: `malloc`/`free` を実装する、ユーザ空間で動くライブラリ。
  OS から粗くメモリを仕入れ、細かく切り分けて配る「小売業者」。
  本書で扱う dlmalloc / glibc / jemalloc / TCMalloc / mimalloc など。

**仮想メモリ（virtual memory）**
: プロセスが見るアドレスを、MMU が物理メモリへ対応づける仕組み。
  対応の単位がページ。→[OS とのインターフェースの章](os-interface.md)

**ページ（page）**
: 仮想→物理の対応づけの単位。x86-64 Linux では標準 4 KiB。

**ヒュージページ（huge page）**
: 2 MiB（や 1 GiB）の大きなページ。TLB 圧迫を緩和するが、
  返却の粒度が粗くなり新種の断片化を生む。→[局所性の章](locality.md)

**デマンドページング（demand paging）**
: 確保した仮想メモリに最初に触れた瞬間に物理ページを割り当てる方式。
  「確保」と「物理メモリ消費」が別イベントになる。→[OS とのインターフェースの章](os-interface.md)

**オーバーコミット（overcommit）**
: 物理メモリ＋スワップを超える確保を許す Linux の既定方針。
  破綻時は OOM killer がプロセスを強制終了する。→[OS とのインターフェースの章](os-interface.md)

**VSZ / RSS**
: VSZ は仮想メモリサイズ（予約も含む）、RSS は実際に物理メモリに
  載っている量（Resident Set Size）。在庫は RSS に現れる。

## システムコール

**brk / sbrk**
: データセグメント末尾（プログラムブレーク）を上下させてヒープを伸縮する
  伝統的 API。末尾からしか返せない。→[OS とのインターフェースの章](os-interface.md)

**mmap / munmap**
: 任意位置に独立した領域を作り（匿名マッピング）、個別に返せる現代の主力 API。
  大きな確保や非メインアリーナに使う。

**madvise**
: 仮想アドレスは保ったまま物理ページだけ返す手段。
  `MADV_DONTNEED`（即破棄）と `MADV_FREE`（遅延回収）がある。
  「`free` してもしばらく RSS が減らない」一因。

## チャンクと管理構造

**チャンク（chunk）**
: dlmalloc/glibc の確保単位。ユーザ領域の直前にサイズ・フラグの
  ヘッダを持つ。→[ヒープを覗く章](inside-malloc.md)・[dlmalloc の章](dlmalloc.md)

**境界タグ（boundary tag）**
: ブロックの末尾にもサイズを複製し、`free` 時に物理的に前のブロックへ
  $O(1)$ で戻れるようにする技法。併合を可能にする。→[フリーリストの章](free-list.md)

**フリーリスト（free list）**
: 空きブロックを連結した台帳。空きブロックの本体にポインタを埋め込む
  「明示的フリーリスト」が基本。→[フリーリストの章](free-list.md)

**bin**
: glibc（および dlmalloc 2.7 系）のサイズ別フリーリスト群。fast / small / large /
  unsorted の 4 種がある（dlmalloc 2.8 系は fast/unsorted を廃し treebin を導入）。→[dlmalloc の章](dlmalloc.md)

**アラインメント（alignment）**
: 戻り値が満たすべきアドレス境界。x86-64/glibc では 16 バイト。
  確保サイズの最小刻みを決める。→[API の章](api-basics.md)

## アルゴリズムとポリシー

**first-fit / best-fit / next-fit**
: 配置方針。最初に見つかった空き／余りが最小の空き／前回位置から再開。
  アドレス順 first-fit と best-fit は実用上良く、next-fit は劣る。→[フリーリストの章](free-list.md)

**分割（splitting）／併合（coalescing）**
: 大きい空きを切って渡す操作／隣り合う空きをくっつける操作。
  併合を怠ると外部断片化が進む。→[フリーリストの章](free-list.md)

**サイズクラス（size class）**
: 確保サイズを離散的な系列に丸める設計。探索を計算に置き換え、
  内部断片化と引き換えに速度を得る。→[サイズクラスの章](segregated-lists.md)

**segregated fit / segregated storage**
: サイズ別フリーリストを使う方式の総称と、その下位分類。
  現代アロケータの背骨。→[サイズクラスの章](segregated-lists.md)

**バディシステム（buddy system）**
: ブロックを 2 の冪に保ち、相棒（buddy = `addr XOR size`）と
  再結合する方式。Linux の物理ページ管理が使う。→[バディとスラブの章](buddy-slab.md)

**スラブアロケータ（slab allocator）**
: 同じ型・サイズのオブジェクトを連続領域（slab）に敷き詰める方式。
  カーネルや言語処理系のヒープの基礎。→[バディとスラブの章](buddy-slab.md)

**TLSF（Two-Level Segregated Fit）**
: 二段ビットマップで確保・解放・併合を真の $O(1)$ にする
  リアルタイム向けアルゴリズム。→[リアルタイムの章](realtime.md)

## 断片化

**内部断片化（internal fragmentation）**
: 渡したブロックの中の無駄（丸め・アラインメント・ヘッダ）。→[断片化の章](fragmentation.md)

**外部断片化（external fragmentation）**
: ブロックとブロックの間の無駄。空きの総量は足りるのに連続していない状態。→[断片化の章](fragmentation.md)

**コンパクション（compaction）**
: 生存オブジェクトを詰め直して断片化を解消すること。
  生ポインタのある C/C++ では一般に不可だが、Mesh は仮想メモリで実現した。→[断片化の章](fragmentation.md)・[最新研究の章](research-frontiers.md)

## 並行性

**アリーナ（arena）**
: 独立したヒープ＋専用ロックの一式。スレッドを振り分けて競合を減らす。
  `MALLOC_ARENA_MAX` で数を制御。→[マルチスレッドの章](multithread.md)・[glibc malloc の章](glibc-malloc.md)

**スレッドキャッシュ（thread cache）/ tcache**
: スレッドごとにロックなしで持つ小さな在庫。大多数の確保をここで済ませる。
  glibc の tcache、TCMalloc の per-CPU cache など。→[マルチスレッドの章](multithread.md)

**偽共有（false sharing）**
: 別々の変数が同じキャッシュライン上にあり、無関係なのに
  キャッシュを奪い合って遅くなる現象。→[マルチスレッドの章](multithread.md)

**blowup**
: スレッドごとの在庫が際限なく膨らむメモリ爆発。Hoard が定数倍に抑えた。
  `MALLOC_ARENA_MAX` 問題の正体。→[マルチスレッドの章](multithread.md)

**遠隔解放（remote free）**
: 確保したスレッドと別のスレッドが `free` すること。
  mimalloc の `thread_free` や snmalloc のメッセージパッシングが扱う。→[マルチスレッドの章](multithread.md)

**ロックフリー（lock-free）**
: ロックを使わず、CAS などのアトミック命令で進行保証を得る方式。→[マルチスレッドの章](multithread.md)

## ハードウェアと局所性

**キャッシュライン（cache line）**
: CPU キャッシュの転送単位。多くは 64 バイト。偽共有の単位。→[マルチスレッドの章](multithread.md)

**TLB（Translation Lookaside Buffer）**
: 仮想→物理アドレス変換結果のキャッシュ。エントリ数が少なく、
  大きなヒープで圧迫される。ヒュージページが緩和する。→[局所性の章](locality.md)

**局所性（locality）**
: 時間的（同じ番地をすぐ再利用）と空間的（近い番地を続けて使う）。
  アロケータの配置が空間的局所性を左右する。→[局所性の章](locality.md)

## セキュリティ

**use-after-free**
: 解放後のポインタ（dangling pointer）を使い続けるバグ。
  攻撃者に別オブジェクトを掴ませる足場になる。→[セキュリティの章](security.md)

**二重解放（double free）**
: 同じポインタを 2 回 `free` する未定義動作。`free` 後の `NULL` 代入で防ぐ。→[API の章](api-basics.md)

**ヒープオーバーフロー（heap overflow）**
: 確保領域を越えた書き込みで隣のチャンクのヘッダやリンクを壊す攻撃。→[セキュリティの章](security.md)

**確率的防御**
: わざとランダムに配置して攻撃の予測可能性を崩す方式（DieHard / DieHarder ほか）。→[セキュリティの章](security.md)

**ケイパビリティ（CHERI）**
: ポインタを「範囲と権限を持つ、ハードウェア検証されるトークン」に
  置き換えるアーキテクチャ。Cornucopia で時間的安全性も実現。→[セキュリティの章](security.md)

## 計測

**AddressSanitizer（ASan）**
: shadow memory と red zone でヒープバグをアクセス時点で検出する
  コンパイラ機能。→[ツールの付録](tools.md)

**ヒーププロファイラ**
: 確保をコールスタックごとに集計し、リーク・肥大の出所を特定する道具
  （jemalloc の prof、TCMalloc の pprof など）。→[ツールの付録](tools.md)

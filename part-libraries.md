# 実際のライブラリ

第 IV 部は本書の総合演習である。第 II 部のアルゴリズム
（フリーリスト、サイズクラス、バディ、スラブ）と
第 III 部の問題意識（断片化、並行性、局所性、セキュリティ）が、
実在するアロケータの中でどう組み合わされているかを、1 章ずつ読み解く。

- [dlmalloc の章](dlmalloc.md) — Doug Lea のアロケータ。
  境界タグ・bin・指定適合という「教科書の理想形」で、
  後続すべての原型。まずここを読むと他が速く読める。
- [glibc malloc の章](glibc-malloc.md) — あなたの Linux で
  今動いている実装（ptmalloc2）。dlmalloc にアリーナと tcache を足した形。
  `MALLOC_ARENA_MAX` の謎がここで解ける。
- [jemalloc の章](jemalloc.md) — FreeBSD・かつての Firefox・
  多くのサーバの主力。アリーナ＋サイズクラス＋積極的な統計とチューニング。
- [TCMalloc の章](tcmalloc.md) — Google の本番アロケータ。
  スレッドキャッシュの語源で、ヒュージページ対応の最前線。
- [mimalloc の章](mimalloc.md) — Microsoft Research 発。
  フリーリストのシャーディングと徹底的に軽い速いパス。
- [その他のアロケータの章](other-allocators.md) — musl、Scudo/hardened_malloc、
  PartitionAlloc、snmalloc、Windows ヒープなど、目的特化の現役たち。

各章は同じ観点——OS との境界、サイズクラス設計、スレッド戦略、
断片化対策、セキュリティ、観察方法——で書いてある。
読み比べると、設計判断の「効き方」が立体的に見えてくるはずだ。

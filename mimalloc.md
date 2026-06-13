# mimalloc — フリーリストのシャーディング

**mimalloc**（"mi" は Microsoft / minimal）は、
Microsoft Research の Daan Leijen らが 2019 年に発表した、
比較的新しいアロケータである[](#cite:leijen2019)。
研究発でありながら実用性も高く、論文・コードともに読みやすい。
設計の核心は「**速いパスを極限まで短くする**」ことにあり、
その手段が **free list sharding**（フリーリストの分割）である。
本書の締めくくりとして、これまで学んだ部品が
「軽さ」のために再構成される様子を見よう。

## ページごとにフリーリストを分割する

[フリーリストの章](free-list.md)以来、フリーリストは
「サイズクラスごとに 1 本」が基本だった。
mimalloc はこれを**さらに細かく刻む**。
フリーリストを**ページ（mimalloc が小オブジェクトを敷き詰める
スラブ相当の単位）ごと**に持つのだ。これが free list sharding である。

なぜ分割するのか。利点は局所性と単純さにある。

- ひとつのページ内の空きだけを連結するので、**確保は常に
  「いま使っているページのローカルフリーリストから 1 つ pop」**になる。
  ここでいう「いま使っているページ」とは、そのサイズクラスについて
  mimalloc が現在の割り当て先として選んでいる**アクティブなページ
  （カレントページ）**のことだ。満杯になるまではこのページから配り続け、
  満杯になったら次のページへカレントを切り替える（後述）。
  ページ内に閉じているので、触るメモリが狭くキャッシュに優しい
  （[局所性の章](locality.md)）。
- ページが満杯になったら次のページへ、空いたページは回収——
  という[スラブ](buddy-slab.md)的な管理が、リスト分割と自然に噛み合う。

この設計のもうひとつの妙が、**3 種類のフリーリストを使い分ける**ことだ。
各ページは (1) 通常の確保に使う `free`、
(2) **同じスレッドが**そのページで解放したものを貯める `local_free`、
(3) **別スレッドが**解放したものを貯める `thread_free`、を持つ。

`free` リストが空になったときだけ `local_free` を `free` に
付け替える。この一手で「確保のたびの解放処理」を
**確保パスから追い出せる**。確保のホットパスは
「`free` リストの先頭を取るだけ、空ならページ切り替え」という、
分岐のごく少ない数命令になる。Leijen らはこの速いパスの短さを
論文で強調している[](#cite:leijen2019)。

言葉だけだと分かりにくいので、ページを「16B ブロック 4 個ぶん」に縮めて動きを追ってみよう。確保は `free` から pop、解放は `local_free` に push、そして `free` が尽きたときだけ `local_free` を `free` に付け替える——という最小モデルを書いて走らせる。

```c
block_t *page_alloc(page_t *pg){
  if (pg->free == NULL) {           // 遅いパス: ここだけ
    pg->free = pg->local_free;      // local_free を free に付け替え
    pg->local_free = NULL;
  }
  block_t *b = pg->free;            // 速いパス: 先頭を pop するだけ
  pg->free = b->next;
  pg->used++;
  return b;
}
void page_free(page_t *pg, block_t *b){
  b->next = pg->local_free;         // 解放は local_free へ積むだけ
  pg->local_free = b;
  pg->used--;
}
```

各操作のあとに 3 つの長さを出力すると、こうなる。

```
初期(満杯のfree)  free=4 local_free=0 used=0
alloc a                free=3 local_free=0 used=1
alloc b                free=2 local_free=0 used=2
free a (->local_free)  free=2 local_free=1 used=1
free b (->local_free)  free=2 local_free=2 used=0
alloc c                free=1 local_free=2 used=1
alloc d                free=0 local_free=2 used=2
  [collect] free <- local_free
alloc e                free=1 local_free=0 used=3
```

注目すべきは `free a`/`free b` の行だ。解放しても `free` の長さは 2 のまま変わらず、回収物は `local_free` にだけ積まれていく。`alloc d` で `free=0` になり、次の `alloc e` で初めて `local_free`（長さ 2）が `free` に付け替えられる。つまり「解放のたびの後始末」が確保パスから消え、`free` を pop し続けるホットパスにはこの付け替え以外の分岐が現れない。

## 遠隔解放を分離する

(3) の `thread_free` は、[マルチスレッドの章](multithread.md)で見た
**遠隔解放**（確保したのと別のスレッドが `free` する）への
mimalloc の回答である。

別スレッドが解放するときは、そのページの `thread_free` リストへ
**アトミックに push** するだけ。所有スレッドは、自分の都合のよいときに
`thread_free` をまとめて回収する。
重要なのは、**所有スレッド側のローカルな確保・解放
（`free`/`local_free`）には一切アトミック操作が要らない**ことだ。
共有が必要なのは遠隔解放という稀なケースだけに局限される。
[マルチスレッドの章](multithread.md)で「mimalloc の遠隔解放リスト」と
紹介したのは、この `thread_free` のことである。
「速いパスにはアトミック命令すら置かない」という徹底ぶりが、
mimalloc の性能の源泉だ。

> [!NOTE]
> 「数命令」は誇張ではない。`free` が非空という前提での pop 本体を gcc 13.3.0 で `-O2` コンパイルすると、本体はこれだけになる（x86-64, Intel 記法）。
>
> ```asm
> mov  rax, QWORD PTR [rdi]      ; b = pg->free
> test rax, rax                  ; 空なら遅いパスへ
> je   .L1
> mov  rdx, QWORD PTR [rax]      ; b->next
> add  QWORD PTR 16[rdi], 1      ; pg->used++
> mov  QWORD PTR [rdi], rdx      ; pg->free = b->next
> ```
>
> ロード 2 回・ストア 2 回・分岐 1 回。アトミック命令（`lock` 接頭辞）は 1 つもない。遠隔解放だけが `thread_free` への `lock` 付き push を払い、所有スレッドの確保・解放はこの素の命令列で済む。

## セキュアモードと encoded free list

mimalloc は[セキュリティ](security.md)も意識した設計を持つ。
通常ビルドでも、フリーリストのポインタを**符号化**して格納する
（encoded free list）。各ページに固有の鍵を持ち、
next ポインタを鍵で変換して保存するので、
ヒープオーバーフローで next を生のアドレスに書き換える攻撃
（[セキュリティの章](security.md)の tcache poisoning 系）が成立しにくい。
glibc の safe-linking（[glibc malloc の章](glibc-malloc.md)）と
同じ発想だが、mimalloc は設計当初から組み込んでいる。

符号化の中身は単純だ。mimalloc 本体（`internal.h`）の `mi_ptr_encode`/`mi_ptr_decode` をそのまま抜き出すと、ページ固有の 2 つの鍵 `keys[0]`/`keys[1]` で「XOR → ローテート → 加算」するだけである。

```c
// next を保存するとき
stored = mi_rotl((uintptr_t)next ^ keys[1], keys[0]) + keys[0];
// next を読むとき
next   = (void*)(mi_rotr(stored - keys[0], keys[0]) ^ keys[1]);
```

これを実際に走らせると、正規の next は往復して元に戻り、攻撃者がオーバーフローで `0x4141...`（"AAAA..."）を書き込んでも、復号結果はまったく別のアドレスに散らばる。

```
next(plain)   = 0x5612a0b34080
stored(enc)   = 0xa42866aef942d274
decoded       = 0x5612a0b34080   (== next ? yes)

-- overflow で next を 0x4141414141414141 に上書き --
decoded       = 0x749bcb2569e8d540
```

攻撃者は `keys` を知らない限り、復号後に狙ったアドレスへ着地させられない。生のアドレスをそのまま next に書けば確保器を誘導できた glibc の tcache poisoning（[セキュリティの章](security.md)）が、この一手でほぼ封じられる。

さらにビルド時に `MI_SECURE`（secure レベル 0〜4）を上げると、
ガードページの挿入、確保位置のランダム化、
解放チャンクの検査などが加わり、[セキュリティの章](security.md)で見た
DieHarder 系の防御に近づく。速度とのトレードオフを
段階的に選べるのは、Guarder[](#cite:silvestro2018) と通じる発想である。

## 採用と影響

mimalloc は登場以来、急速に採用が広がった。
`LD_PRELOAD` で手軽に試せ（[ヒープを覗く章](inside-malloc.md)）、
多くのベンチマークで jemalloc / TCMalloc と互角以上の性能を出す。
.NET ランタイムのネイティブヒープ、各種データベース、
Rust の `mimalloc` クレート（jemalloc から乗り換える例も多い）、
そして言語処理系の実験的なビルドなどで使われている。

研究としての mimalloc の貢献は、
「**フリーリストを細かく分割し、速いパスからすべての例外処理を
追い出す**」という設計原則を、読みやすいコードと論文で示したことだ。
本書でばらばらに学んだ部品——フリーリスト、サイズクラス、スラブ、
スレッドキャッシュ、遠隔解放、セキュア化——が、
「軽さ」という一本の方針のもとに再構成される様子は、
アロケータ設計の良い総まとめになっている。

> [!TIP]
> mimalloc のコードと論文[](#cite:leijen2019)は、本書を読み終えた人の
> 「次の一冊」に最適だ。規模が手頃で、ここまでに出てきた概念が
> ほぼすべて実物として確認できる。読みながら `LD_PRELOAD` で
> 自分のプログラムに適用し、[付録のツール](tools.md)で
> RSS と速度を測れば、理論と実装と計測が一本につながる。

## まとめ

mimalloc は、フリーリストをページ単位に分割（free list sharding）し、
ローカル解放・遠隔解放を別リストに分けることで、
確保のホットパスを「先頭を pop するだけ」の数命令に切り詰めたアロケータである。
遠隔解放だけをアトミック操作に局限し、
encoded free list やセキュアモードでセキュリティにも配慮する。
本書で学んだ全部品が「軽さ」のもとに再構成された、
現代アロケータ設計の好例だ。
次章では、ここまで取り上げきれなかった目的特化のアロケータたちを概観し、
本書を締めくくる。

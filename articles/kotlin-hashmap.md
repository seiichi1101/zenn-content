---
title: "KotlinのHashMapがどう実装されているかを追ってみた"
emoji: "😊"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["kotlin", "java", "hashmap"]
published: true
---

たまたま調べる機会があったので、Kotlin の HashMap の実装がどうなっているのかを追ってみました。
今回はその備忘録です。

なにか間違いや補足があれば、ぜひ教えてください。

## Kotlin の HashMap の実装

- 環境

```
$ kotlin -version
Kotlin version 2.2.21-release-469 (JRE 17.0.17+10-LTS)
```

まず[公式ドキュメント](https://kotlinlang.org/docs/command-line.html#compile-a-library) にしたがって、簡単なコードを実行してみます。

下記のようなコードを`hello.kt`というファイル名で保存します。

```kotlin
fun main() {
  println("Hello, World!")

  val map: HashMap<Int, String> = HashMap()
  map.put(1, "one")
  map.put(2, "two")
  map.put(3, "three")
  println("HashMap contents: $map")
}
```

これを、`kotlinc hello.kt -include-runtime -d hello.jar` でコンパイルします。

`java -jar hello.jar`で実行すると、下記のように出力されます。

```
Hello, World!
HashMap contents: {1=one, 2=two, 3=three}
```

これだけだとピンと来ないので、HashMap ドキュメントを確認してみます。

https://kotlinlang.org/api/core/kotlin-stdlib/kotlin.collections/-hash-map/

定義としては expect (common 仕様) があり、各プラットフォームごとに actual (JS/JVM/Native/Wasm-JS/Wasm-WASI 実装) があるみたいです。
先ほど実行した `kotlin` コマンドは JVM 用の actual 実装を利用しています。

https://github.com/JetBrains/kotlin/blob/v2.2.21/libraries/stdlib/jvm/src/kotlin/collections/TypeAliases.kt#L18

JVM 用のソースコードを確認してみると、 `java.util.HashMap` への `typealias` になっていることが分かります。
**つまり、Kotlin の HashMap は JVM 環境では Java の HashMap をそのまま利用していることになります。**

ちなみに JS 版の HashMap は下記のように Kotlin で実装されています。これはおそらく Kotlin の common 仕様を満たす実装が JS には存在しないため、Kotlin で実装しているのだと思います。

https://github.com/JetBrains/kotlin/blob/v2.2.21/libraries/stdlib/js/src/kotlin/collections/InternalHashMap.kt

今回は JVM 環境の HashMap の実装を追ってみます。
JVM 実装が実際に参照している Java のコードを確認するために、下記のようにコードを修正して実行してみます。

```diff kotlin

fun main() {
  println("Hello, World!")
  val map: HashMap<Int, String> = HashMap()
  map.put(1, "one")
  map.put(2, "two")
  map.put(3, "three")
  println("HashMap contents: $map")
+ println(map::class.java.name)
+ println(map::class.java.module.name)
+ println(map::class.java.classLoader)
+ println(System.getProperty("java.runtime.version"))
+ println(System.getProperty("java.home"))
}

```

結果は下記のようになります。

```
Hello, World!
HashMap contents: {1=one, 2=two, 3=three}
java.util.HashMap
java.base
null
17.0.17+10-LTS
/home/<username>/.local/share/mise/installs/java/corretto-17.0.17.10.1
```

ということで、結局 Java の HashMap (`java.util.HashMap`) の実装を追うことになりました。

## Java の HashMap の実装

`amazon-corretto` の `17.0.17+10-LTS` ソースコードを追って行きます。

https://github.com/corretto/corretto-17/blob/17.0.17.10.1/src/java.base/share/classes/java/util/HashMap.java#L139

### ハッシュ値の生成

まず、HashMap に Key-Value ペアを追加する際の `map.put(key, value)` の中身を追ってみます。

`putVal` で実際に Key-Value ペアの追加処理を行う前に、`hash(Object key)` 関数で hash 値を生成しています。

https://github.com/corretto/corretto-17/blob/be19a43ac1bf6da3f6c90493d6085a3956ce9503/src/java.base/share/classes/java/util/HashMap.java#L609-L611

`hash(Object key)`では、**各オブジェクトごとの `hashCode()` を取得し、ビット操作(16bit 右シフト)** を行っています。

https://github.com/corretto/corretto-17/blob/be19a43ac1bf6da3f6c90493d6085a3956ce9503/src/java.base/share/classes/java/util/HashMap.java#L336-L339

#### オブジェクトごとの `hashCode()` とは？

各オブジェクトごとに定義されている `hashCode()` メソッドを呼び出してハッシュコードを取得しています。

例えば、`Integer` クラスの hashCode() メソッドは、その整数値自体を返します。Integer が`1`の場合、hashCode()は`1`を返します。

https://github.com/corretto/corretto-17/blob/be19a43ac1bf6da3f6c90493d6085a3956ce9503/src/java.base/share/classes/java/lang/Integer.java#L1207-L1209

`String` クラスの `hashCode()` メソッドは、31 を基数とする多項式ハッシュが使われています。
String が`"1"`の場合は `49`、`"12"`の場合は `1569 = 49 * 31 + 50`、`"123"`の場合は `48690 = (49 * 31 * 31) + (50 * 31) + 51` のように計算されます。

https://github.com/corretto/corretto-17/blob/be19a43ac1bf6da3f6c90493d6085a3956ce9503/src/java.base/share/classes/java/lang/StringLatin1.java#L194-L200

こうして各オブジェクトごとに hashCode() を取得した後、下記のようなビット操作を行い、最終的なハッシュ値を生成しています。

#### 16 ビット右シフトとは？

値を右に 16 ビットシフトし、元の値と排他的論理和 (XOR) を取っています。`(h = key.hashCode()) ^ (h >>> 16);` の部分です。

コメントにもありますが、これは上位 16 ビットを下位 16 ビットに XOR で混ぜ込むことで、衝突(collision)を減らすための工夫だそうです。

> Computes key.hashCode() and spreads (XORs) higher bits of hash to lower. Because the table uses power-of-two masking, sets of hashes that vary only in bits above the current mask will always collide. (Among known examples are sets of Float keys holding consecutive whole numbers in small tables.) So we apply a transform that spreads the impact of higher bits downward. There is a tradeoff between speed, utility, and quality of bit-spreading. Because many common sets of hashes are already reasonably distributed (so don't benefit from spreading), and because we use trees to handle large sets of collisions in bins, we just XOR some shifted bits in the cheapest possible way to reduce systematic lossage, as well as to incorporate impact of the highest bits that would otherwise never be used in index calculations because of table bounds.

> key.hashCode() を計算し、ハッシュの上位ビットを下位ビットに拡散（XOR）します。テーブルが 2 の累乗マスクを使用しているため、現在のマスクより上位のビットのみが異なるハッシュの集合は常に衝突します（既知の例としては、小規模テーブルで連続した整数を保持する Float キーの集合など）。そこで上位ビットの影響を下位ビットに拡散する変換を適用します。速度、有用性、ビット拡散の品質の間にはトレードオフが存在する。多くの一般的なハッシュ値の集合は既に合理的に分散されている（したがって拡散の恩恵を受けない）こと、また衝突の多い集合を処理するためにツリー構造を用いることから、体系的な損失を低減し、さらにテーブルの境界によりインデックス計算で決して使用されない最高位ビットの影響を取り込むため、可能な限り低コストな方法でシフトしたビットを XOR 演算する。

これが約に立つのは、最終的に、Key-Value(Node)を保存する table の index を決めるときです。
index の計算式 `(n - 1) & hash` において、ハッシュ値が大きい場合にも均等に分布させることができます。

https://github.com/corretto/corretto-17/blob/be19a43ac1bf6da3f6c90493d6085a3956ce9503/src/java.base/share/classes/java/util/HashMap.java#L628

ここで table とは、`Node<K,V>[] tab` のことで、`Node`を格納する配列です。最終的に、Key-Value ペアは Node オブジェクトとして保存されます。 table は配列として定義されており、連続したメモリ領域に Node オブジェクトの参照を格納します。また、n は table の長さ (配列のサイズ) です。

たとえば、 n(テーブルのサイズ) が 16(デフォルト値) の場合、`n - 1` は `0000...1111` となり、下位 4 ビットしか index の決定に使われません。そのため、下記のような上位のビットのみを持つ値は、`(n - 1) & hash` の計算結果が常に `0` となり、すべて同じ index にマッピングされてしまい、衝突(collision)が発生してしまいます。

- `0000 0000 0000 0001 0000 0000 0000 0000` (2^16 = 65536)
- `0000 0000 0000 0010 0000 0000 0000 0000` (2^17 = 131072)
- `0000 0000 0000 0011 0000 0000 0000 0000` (2^18 = 196608)

要するに、後の Table Index 決定の計算に「上位ビットの情報を反映させて均等に保つため、XOR 操作で上位ビットの情報を下位ビットに混ぜ込んでいる」ということになります。

こうして最終的に得られた、`hash`と`key`と`value`が `putVal` メソッドに渡され、Key-Value ペアの追加処理が行われます。

言葉ではわかりづらいと思うので、上記の動作を図にまとめてみました。

![](/images/kotlin-hashmap/hashmap.png)

ちなみに、`size > threshold`（threshold は `capacity * loadFactor`、デフォルト loadFactor は 0.75）を満たすと [resize() メソッド](https://github.com/corretto/corretto-17/blob/be19a43ac1bf6da3f6c90493d6085a3956ce9503/src/java.base/share/classes/java/util/HashMap.java#L675) が呼ばれて自動的に拡張される仕組みになっています。

### Collision(衝突) 発生時の処理

次に index が同じ値になった場合、いわゆる衝突(collision)が発生した場合の処理を見ていきます。

`(n-1) & hash` の計算で得られた index に Node が存在しない場合は、新しい Node をその index に追加します。628-629 行目の部分です。

一方で、630 行目以降は、すでにその index に Node が存在する場合（collision 発生時）の処理を行っています。

https://github.com/corretto/corretto-17/blob/be19a43ac1bf6da3f6c90493d6085a3956ce9503/src/java.base/share/classes/java/util/HashMap.java#L623-L664

#### 1 つめの分岐: 該当 Index の先頭 Node と hash 値と key が完全に一致する場合

1 つ目の分岐では、hash 値と key の両方が先頭 Node と一致する場合をチェックし、その Node を変数 `e` に保持します。`e` がセットされている場合は、既存 Node の value だけを置き換える処理が後方で書かれています。

https://github.com/corretto/corretto-17/blob/be19a43ac1bf6da3f6c90493d6085a3956ce9503/src/java.base/share/classes/java/util/HashMap.java#L632-L634

#### 2 つめの分岐: 既存のエントリ p が TreeNode の場合

2 つ目の分岐では、既存のエントリ `p` が TreeNode かどうかをチェックしています。TreeNode である場合、`putTreeVal` メソッドを呼び出してツリー構造に新しいエントリを必要に応じて追加します。

`putTreeVal` でも hash 値と key の両方が一致する場合をチェックし、一致すると既存エントリを返します。戻り値がある場合は、 `e` に代入され、後方の処理で既存 Node の value のみが更新されます。

https://github.com/corretto/corretto-17/blob/be19a43ac1bf6da3f6c90493d6085a3956ce9503/src/java.base/share/classes/java/util/HashMap.java#L635-L636

`putTreeVal` の中身も追いたかったですが、理解に時間がかかりそうだったので、今回は割愛します。

#### 3 つめの分岐: 既存のエントリ p が Linked List 構造の場合

3 つ目の分岐では、先頭 Node `p` の次の Node を `p.next` でたどりながら、hash 値と key の両方が一致する Node を探します。
一致した場合は `e` に代入され、例のごとく既存 Node の value のみが更新されます。見つからなかった場合、新しい Node を末尾にぶら下げます。`p.next = newNode(hash, key, value, null);` の部分です。

また、その後の処理で、ある条件 `if (binCount >= TREEIFY_THRESHOLD - 1)` を満たすときに 、データ構造を Linked List 構造から TreeNode 構造に変換しています。`treeifyBin(tab, hash);`の部分です。

`static final int TREEIFY_THRESHOLD = 8;` と設定されているので、デフォルトでは 8 個以上の衝突が発生した場合に Tree 構造に変換されるみたいです。 （table のサイズが `MIN_TREEIFY_CAPACITY = 64` 未満のときは TreeNode 化はせず、まず配列拡張だけを行う模様）

ちなみに、`TreeNode`は`Node`のサブクラスであるため、`Node`型の配列である Table にそのまま格納できるようになっています。

つまり、衝突が多い場合には、Linked List 構造から TreeNode 構造に変換することで、検索や挿入のパフォーマンスを向上させています。
TreeNode は自己平衡二分探索木である赤黒木(Red-Black Tree)を使用しているため、衝突が多い場合でも効率的にデータを管理できます。
通常の Linked List 構造では、最悪の場合 `O(n)` の時間がかかりますが、赤黒木は `O(log n)` の時間で操作が可能になります。

https://github.com/corretto/corretto-17/blob/be19a43ac1bf6da3f6c90493d6085a3956ce9503/src/java.base/share/classes/java/util/HashMap.java#L637-L650

`treeifyBin` の中身も追いたかったですが、理解に時間がかかりそうだったので、今回は割愛します。

## まとめ

Kotlin の HashMap の実装はプラットフォームごとに異なり、JVM 環境では Java の `java.util.HashMap` をそのまま利用しています。`java.util.HashMap` のコードを追うことで、ハッシュの混ぜ込み、collision 時のリスト・木構造への分岐といった動作原理をある程度理解できました。

意外と奥が深くて面白かったです。

なにか間違いや補足があれば、ぜひ教えてください。

---
title: data / newtype / type の使い方
subHeading: Haskell でデータ型を定義する 3 つの方法
headingBackgroundImage: ../../img/background.png
headingDivClass: post-heading
author: Mizunashi Mana
postedBy: Mizunashi Mana
date: June 14, 2020
...
---

Haskell プログラミングにおいて，データ型は非常に重要な役割を持つ．データ型は，扱うデータをプログラミング上で安全かつ容易に加工するために用いられ，またデータに対してどのような操作ができるのかを規定する．

Haskell には，データ型を新たに定義する方法が3つある．

* 1つ目は `type` キーワードによって定義する方法で，これにより定義されたデータ型は型シノニムと呼ばれる．
* 2つ目は `data` キーワードによって定義する方法で，これにより定義されたデータ型は代数的データ型と呼ばれる．
* 3つ目は `newtype` キーワードによってある型を元に新たな型を作る方法だ．

今回は，それぞれどういう使い方をするのか，どういう違いがあるのかについて見ていきたいと思う．

## 型シノニム

例えば，あなたは Web サイトを運営していて，一部年齢制限が必要なため，人の年齢が 20 歳以上かを判定する関数を書かなければいけないとする．年齢は整数だが，入力は必須でないため入力してない人もいる．その場合は，20 歳以上でないと判定する．この関数は，

```haskell
isAdult :: Maybe Int -> Bool
isAdult m = case m of
  Nothing -> False
  Just x  -> x >= 20
```

と書ける．ただ，この定義はどこか味気ない．`isAdult` が受け取るデータは，年齢を表していて，整数か未詳かの状態を持つので，`Maybe Int` はデータを正確に捉えられている．しかし，`Maybe Int` に適合するデータは他に無数にあるため，`isAdult` が受け取るデータが年齢を表すのか知能指数を表すのか，はたまた今までお酒を飲んだことのある回数なのかは推測しないと分からない．年齢を表すデータ型を新たに定義して，それを受け取るようにすればもっとプログラムがクールになるだろう．

Haskell で新しくデータ型を定義する最も簡単な方法は，`type` キーワードを使って型シノニム (type synonym) を定義する方法だ．シノニムとは，別名という意味で，型シノニムは文字通り，ある型の別名を表す．今回は次のように使える:

```haskell
type Age = Maybe Int

isAdult :: Age -> Bool
isAdult age = case age of
  Nothing -> False
  Just x  -> x >= 20
```

これで関数 `isAdult` は，先ほどと比べてとても明確になった．`Age` は `Maybe Int` を元に作られた型シノニムで，つまり `Age` は `Maybe Int` の別名になっている．単なる別名なので，`isAdult` は `Maybe Int -> Bool` 型の関数だと思って使うこともできる．GHCi で試してみよう:

```haskell
>>> (isAdult :: Maybe Int -> Bool) (Just 22 :: Age)
True
```

`Maybe Int` を `Age` だと思うこともできるしその逆もできる．型シノニムと元となった型は自在に取り替え可能だ．型シノニムはとても手軽なので，Haskell の標準ライブラリでも使われている．例えば，次のようなデータ型が型シノニムで定義されている:

```haskell
type String = [Char]
type FilePath = String
```

文字列は文字のリストと見做せる．そこから文字列によるデータ型 `String` は，単に文字のリスト型の型シノニムで定義されている．文字列に対してリストの関数を自由に適用できるのは，このためだ．ファイルのパスによるデータ型 `FilePath` は `String` の型シノニムで定義されている．なので，文字列の関数を自由に適用できる．

Haskell の型シノニムは，これだけに止まらずもっと強力な機能も持っている．例えば，型シノニムは型コンストラクタ，すなわち型を受け取って新たな型を作るコンストラクタに対しても作れる:

```haskell
type Option = Maybe
```

この型シノニムを使うと，`Maybe Int` と書く代わりに `Option Int` と書くことも可能だ．部分適用された型コンストラクタに対する型シノニムも書ける:

```haskell
type Failable = Either String
```

この型シノニムを使うと，`Either String ()` と書く代わりに `Failable ()` と書くことができる．

さらに型シノニムは，パラメータを持つことができる:

```haskell
type List a = [a]
```

この型シノニムを使うと，`[Int]` は `List Int` と書ける．ただし，型シノニムはあくまで別名なので，全てのパラメータを適用した状態でしか書けないことに注意する必要がある．例えば，次のプログラムはコンパイルエラーになる:

```haskell
type Apply f a = f a
type ApplyMaybe = Apply Maybe
```

`Apply` は2つのパラメータをとるが，`ApplyMaybe` は `Apply` に1つのパラメータしか渡していない．この場合，`Apply Maybe` という型がどういう型の別名になるか Haskell は分からないため，この型を拒否する．このプログラムを修正するには，

```haskell
type Apply f a = f a
type ApplyMaybe a = Apply Maybe a
```

というように，`Apply` に全ての引数を渡してやる必要がある．こうすることで，Haskell は `Apply` の定義から `Apply Maybe a` が `Maybe a` の別名であると認識できるようになる [^notice-undecidability-of-type-lambda]．

[^notice-undecidability-of-type-lambda]: 型シノニムに対して部分適用を許容する一般的な方法は，型上にもラムダ抽象にあたる表現を導入することである．ただ，この場合型上の演算が停止しない場合があり，型システムが決定不能になる．このため，Haskell では型シノニムに対しての部分適用は許容していない．

型シノニムは，他にも幾つか用途上で制限がある．1つ目は再帰的な型シノニムが作れないという制限だ．例えば，

```haskell
type InfiniteList a = (a, InfiniteList a)
```

という定義は Haskell では却下される．相互再帰的な定義も許容されていない:

```haskell
type Rec1 = [Rec2]
type Rec2 = [Rec1]
```

`Rec1` の型を具体的に求めようとすると，`[Rec2]` の型になる．`Rec2` はやっぱり型シノニムで，`[Rec1]` の別名なので，この型はさらに `[[Rec1]]` という型になる．このようにして具体的な型を求めようとしても永遠に型シノニムがどこかしらに入り込むことになってしまい，型シノニムが現れない型を求めることはできない．Haskell ではそのようなことがないように，そのような定義を排除している [^notice-equirecursive-types]．

[^notice-equirecursive-types]: 等価再帰データ型 (equirecursive types) と呼ばれる特別な型を型システムに導入することで，このような型を許容する理論は存在するが，この理論はとても複雑で型検査のアルゴリズムも難しくなりがちである．

もう1つの制約は，型シノニムを型クラスのインスタンスとして使えないというものだ．例えば，次のようなことはできない:

```haskell
type I = Int

class C a
instance C I
```

代わりに，

```haskell
class C a
instance C Int
```

というように型シノニムを使わず書く必要がある．これは型シノニムを使って書けない唯一の例外だ．ただ，この制限は本質的なものではなく，Haskell 標準で型シノニムに対する混乱を避けるための制限になっている．もし，型シノニムに対してインスタンスを書けるようにしても，型シノニムは単なる別名なので，それは元となった型に対してインスタンスを定義してることと同じになる．このため，

```haskell
f :: C a => a -> a
f x = x
```

という関数は，`type Age = Int` による型シノニム `Age` に対して `C` のインスタンスが定義されていた場合，`a` が `Age` の場合も `Int` の場合も許容される．これは，プログラマが意図していない動作かもしれない．つまり，年齢のデータだけにインスタンスを定義したつもりが，整数データ全般に対していつのまにかインスタンスを定義してしまったことになるからだ [^notice-type-synonym-instances]．

[^notice-type-synonym-instances]: ただ，このような混乱が起こるかもしれないことを許容し，利便性のため型シノニムをインスタンス定義で使いたい場合，`TypeSynonymInstances` という GHC 拡張を有効にすることで許容されるようになる．

これらの制限はあるものの，型シノニムはデータ型を定義する上でとても強力で，しかも簡単に使用できる機能だ．

## 代数的データ型

さて，型シノニムでデータ型を定義する場合には幾つかの制限があった．では，この制限を超えたデータ型を定義する方法はないのだろうか？ そのような場合には代数的データ型 (algebraic datatype) を使うことができる．

代数的データ型は，複数の型の値を統合して1つの型の値として扱うデータ型の積と，複数の型の表現範囲を合わせて1つの型として扱うデータ型の和を組み合わせることで構成されている．そして，このデータ型の定義は，型シノニムと異なり完全に新しい型を作り出す．実際の例を見てみよう．

あなたは積木パズルのパーツそれぞれの面積を計算する関数を，書かなければいけない．積木パズルのパーツはそれぞれ，長方形，真円，三角形から構成されている．まずはこのパーツを Haskell のデータ型に落とし込む必要がある．それぞれのパーツにおいて，

* 四角形の面積は縦横の長さ
* 真円は半径
* 三角形は三辺の長さ

によって特徴付けられている．では，これを代数的データ型に落とし込んでみよう:

```haskell
data PuzzleElement
  = Rect
      Double
      -- ^ 縦の長さ
      Double
      -- ^ 横の長さ
  | Circle
      Double
      -- ^ 半径
  | Triangle
    -- ^ 三つの辺の長さを与える
      Double Double Double 
```

この定義は，`PuzzleElement` という新しい型を作り，3つの値コンストラクタを作る．それぞれ

* `Rect :: Double -> Double -> PuzzleElement`
* `Circle :: Double -> PuzzleElement`
* `Triangle :: Double -> Double -> Double -> PuzzleElement`

という型を持つ．`Rect` は `Double` 型の値を2つ受け取り，その2つの値を `PuzzleElement` 型の1つの値として統合する．つまり，`Double` 型2つの積を作る．`Circle` や `Triangle` も同様だ．そして，`PuzzleElement` 型は3種類の積の値のいずれかを表し，すなわちこれら3種類の積の和を表す．このように，積和によって新しいデータ型を定義できるのが `data` 宣言であり，それによって定義されるのが代数的データ型になる．

代数的データ型の値から統合した値を取り出したい時は，`case` 文を使ったパターンマッチを行う:

```haskell
areaMeasure :: PuzzleElement -> Double
areaMeasure x = case x of
  Rect w h -> w * h
  Circle r -> r * r * pi
  Triangle s1 s2 s3 ->
    let s = (s1 + s2 + s3) / 2
    in sqrt $ s * (s - s1) * (s - s2) * (s - s3)
```

`areaMeasure` によってパズルのピースの面積を求めることができるようになった．

前に紹介した型シノニムは，ある型に対してその別名を与えるだけだった．それに比べ，代数的データ型では新しいデータ型を作り，その型の値を作る値コンストラクタを定義する．そして，型シノニムと大きく異なる点は，型システム上からは新たに定義された型しか分からず，実際にそのデータ型がどういう型から構成されるか分からない点にある．`PuzzleElement` 型の値は，もしかしたら `Double` 型の2つの値から `Rect` コンストラクタを介して作られているかもしれないし，`Double` 型1つの値から `Circle` コンストラクタを通して作られているかもしれない．これは実行時にその関数でパターンマッチをしてみて初めて分かることだ．型シノニムでは，型システムからそれがどういう型を元にしていたか分かるが，代数的データ型で観測できるのは新たに作られたデータ型があることだけだ．この違いは，代数的データ型と型シノニムの制約の違いに表れてくる．代数的データ型では，型シノニムの時に挙げたような制約はない．

例えば，代数的データ型は型シノニムと同様，パラメータをとることができ，さらに部分適用も可能だ [^notice-type-constructor-is-canonical]:

[^notice-type-constructor-is-canonical]: 型上の計算によって，実際の型が特定される型シノニムとは異なり，代数的データ型の型コンストラクタはそれ自体がもう計算できないものになる．それは部分適用されても同様であり，部分適用を許容することで型シノニムと同様の問題は起こらない．これが，代数的データ型で部分適用が許容されている理由になる．

```haskell
data Apply f a = Apply (f a)
type ApplyMaybe = Apply Maybe
```

これは Haskell の正しいプログラムになる．`Apply` は，2つのパラメータをとる型コンストラクタになっていて，データ型 `Apply f a` の値を作る方法として，`f a` 型の値から値コンストラクタ `Apply :: f a -> Apply f a` を通す方法がある．`ApplyMaybe` は `Apply Maybe` の型シノニムになっていて，これを使えば `Apply Maybe Int` と書く代わりに `ApplyMaybe Int` と書けるようになる．`ApplyMaybe` の定義は，`Apply` に対して1つのパラメータしか渡していない．にも関わらず正しいというのが，型シノニムと異なる点になる．

再帰的なデータ型を代数的データ型で定義することも可能だ:

```haskell
data List a
  = Cons a (List a)
  | Nil
```

データ型 `List a` は `a` 型の要素を持つ単連結リストを表す．値コンストラクタが `List a` 型の値を受け取ることがポイントだ．型シノニムでは，その型の定義に自身を含めることはできなかった．これは実際の具体的な型を求めようとした時，その計算が永遠に終わらなくなってしまうからだった．代数的データ型 `List a` ではその型は単に新しい型として作られ，実際にその型の値がどういう型の値によって構成されているか知る必要はない．`List a` はそれ自体が具体的な型であり [^notice-parameter-is-not-concrete] ，それ以上計算する必要はないからだ．代数的データ型において，定義された型とその型の値を作る方法は分離されている．そのため，データ型の計算においてその型の値を作る方法は考慮されない．よって，自身が定義中で用いられても，型シノニムのようにデータ型の計算が永遠に終わることがないということはないため，その操作が許容されている．

[^notice-parameter-is-not-concrete]: 実際にはパラメータ `a` の部分に具体的な型を当てはめないといけないが，当てはめればそれは完全に具体的な型になる．

もちろん，新しい型が定義されるため，型クラスのインスタンスを混乱なく定義できる．代数的データ型を作成した時，基本的なインスタンスを定義することは Haskell プログラミングにおいてよくあることだ．Haskell では，言語機能としてそれを支援する機能がある．それは，`deriving` 構文というもので，`Eq` / `Ord` などの標準的な型クラスを，データ型の定義から自動で導出してくれる．例えば，`List a` に対して使ってみると，以下のようになる:

```haskell
data List a
  = Cons a (List a)
  | Nil
  deriving (Eq, Ord, Show)
```

このように代数的データ型は，型シノニムでは定義できなかったデータ型を定義することができる．そして，代数的データ型は全く新しい型を作ることもできる:

```haskell
data Nat
  = Succ Nat
  | Zero
```

このデータ型 `Nat` は，他の型には依存しない全く新しい型だ．このように，代数的データ型は型シノニムと異なり全く新しい構造を作り出すことができる．

ただ，その代わり既存の関数を流用できなくなってしまう場合がある．例えば，

```haskell
data Tuple a b = Tuple a b
```

は，`(a, b)` と構造が同じであり，`(a, b)` に対する関数 `fst :: (a, b) -> a` を適用できてもいいはずだ．ところが，データ型 `Tuple a b` とその値コンストラクタは型システム上は切り離されているため，自身の値が `(a, b)` の値と同じ方法でしか構成できないことを知らない．`Tuple a b` と `(a, b)` において型上で言及できることは，それらが異なる型であるということだけだ．なので，`fst` に `Tuple a b` 型の値を渡すことはできない．これは，もし型シノニムを使って，

```haskell
type Tuple a b = (a, b)
```

と定義した場合は解決する問題だ [^notice-generic-programming]．

[^notice-generic-programming]: なお，代数的データ型でも型シノニムと同様の利点を手に入れるための研究は，Haskell では盛んに行われている．例えば，`Generic` / `Data` 型クラス，`lens` パッケージなどを使うことで，構造が同じだが異なるデータ型で関数が流用できない問題を回避できる場合がある．

このように両者にはトレードオフがあり，利用目的に合った使い分けをするのがいいだろう．

さて，`data` 宣言の構文は他に2つ，便利な機能がある．

1つは正格性フラグと呼ばれる機能で，値コンストラクタにおいて引数を正格に評価することを強制できる．例えば，

```haskell
data StrictTuple a b = StrictTuple !a !b
```

というように，正格性フラグ `!` を使った定義を行うと，値コンストラクタ `StrictTuple :: a -> b -> StrictTuple` はその引数を正格に評価してから格納するようになる．通常，

```haskell
data Tuple a b = Tuple a b
```

のように正格性フラグを使わない定義では，

```haskell
>>> case Tuple undefined undefined of Tuple _ _ -> ()
()
```

のように値コンストラクタは受け取った引数の評価を行わず，素直にそのままの形で遅延させて格納するため，エラーを出す式を渡してもその式の評価を行わない限りエラーにはならない．これは通常の関数の動作と同じになる．ところが，正格性フラグを使用した `StrictTuple` の場合，

```haskell
>>> case StrictTuple undefined undefined of StrictTuple _ _ -> ()
*** Exception: Prelude.undefined
```

のように引数の評価を行うため，エラーを出す式を受け取った場合値コンストラクタの適用においてその式を評価しエラーを出す．データ型を作成する際，その元となる式の評価を強制させることはパフォーマンスに大きく寄与する．そのため，そのようなことを支援するために正格性フラグは設けられている．

また，代数的データ型の値コンストラクタはフィールド名を持つことができる:

```haskell
data Tuple a b = Tuple
  { firstVal  :: a
  , secondVal :: b
  }
```

この場合，型コンストラクタ `Tuple`，値コンストラクタ `Tuple :: a -> b -> Tuple a b` の他に，関数 `firstVal :: Tuple a b -> a`， `secondVal :: Tuple a b -> b` が作られる．また，値コンストラクタの呼び出しにおいて特別なレコード構文 `Tuple { firstVal = 0, secondVal = 1 }` を使用でき，またレコード更新構文 `(Tuple 2 1) { firstVal = 0 }` を使用できる．これらは両者 `Tuple 0 1` と同様の値が作成される．

## ある型を元に新たな型を作る (Datatype Renaming)

さて，これまで見てきたように，型シノニムは型の別名を定義し，代数的データ型は型の積和により新たなデータ型を定義するものだった．Haskell にはもう1つデータ型を定義する方法がある．それが `newtype` 宣言だ．この宣言によって作られるデータ型は，型システム上は代数的データ型と同じように扱われ，実行時は型シノニムと同様の動作をする．

`newtype` 宣言の構文は，`data` 宣言と同じような形をしている:

```haskell
newtype Identity a = Identity a
```

フィールド名をつけることもできる:

```haskell
newtype Identity a = Identity
  { unIdentity :: a
  }
```

この場合 `data` 宣言と同様に，型コンストラクタ `Identity`，値コンストラクタ `Identity` が作られることになる．ただし，`data` 宣言と異なり `newtype` は積和の機能を使用することはできない．単にある1つの型を受け取る値コンストラクタしか定義できない．なので，

```haskell
newtype Unit = Unit
newtype Tuple a b = Tuple a b
newtype Enum = A | B | C
```

はいずれも受け入れられない．この `newtype` の制約はいまいちよく分からない．では，このような制約によりどのような違いが出るのだろうか？ `newtype` と `data` は型システム上は違いはない．しかし，パターンマッチの動作など，実行時の動作に少し差異が設けられている．例えば，通常

```haskell
data DataIdentity a = DataIdentity a
```

において，

```haskell
>>> case undefined :: DataIdentity () of DataIdentity _ -> ()
*** Exception: Prelude.undefined
```

のようにエラーを出す式をパターンマッチで分解しようとするとエラーが出力される．ところが，`newtype` によって作られた値コンストラクタの場合，

```haskell
>>> case undefined :: Identity () of Identity _ -> ()
()
```

のようにパターンマッチ時にエラーが出されることはない．Haskell では `newtype` で作られた値コンストラクタが実行動作に影響することはないと規定されている．よって，上のパターンマッチは，以下と同様の動きをすることになっている:

```haskell
>>> case undefined :: Identity () of _ -> ()
()
```

このように値コンストラクタを指定しないパターンマッチの場合，`data` 宣言で作られたものもエラーを出さない:

```haskell
>>> case undefined :: DataIdentity () of _ -> ()
()
```

よって，`data` と `newtype` で作られた値コンストラクタの動作が異なるのは，パターンマッチにおいて値コンストラクタを指定した場合だけということになる．

では，`newtype` はなぜ値コンストラクタを無視するよう規定されているのだろう？ これは，`newtype` によるデータ型が実行時の動作として型シノニムと同様の動作をすることを目的としてしているからだ．値コンストラクタが無視されるのは，

```haskell
newtype Identity a = Identity a
```

という宣言は，

```haskell
type IdentitySynonym a = a
```

という宣言と同様の意味を持って欲しいことを Haskell の設計者が意図しているからだ．よって，

```haskell
>>> case undefined :: Identity () of Identity _ -> ()
()
```

の動作は，

```haskell
>>> case undefined :: IdentitySynonym () of _ -> ()
()
```

のように，代数的データ型ではなく型シノニムに合わせてあるため，`data` 宣言主体に見ると一見不思議な動作をしていたというわけだ．

さて，ではなぜわざわざ型シノニムとは別に `newtype` 宣言を導入したのだろうか？ 型シノニムには幾つか制約があったのを思い出して欲しい．そして，それらの制約は代数的データ型では解決されたのだった．それは `type` 宣言が単に型の別名を導入するのに対し，`data` 宣言が完全に新たな型を作るからだった．`newtype` はその点に着目し，実行時には単なる別名として動作するが型システム上は完全に別の新たな型を導入することで，`type` 宣言同様ある型の別名を作りたいものの型シノニムの制約は回避したい需要を満たすようにしたものだ．

例えば，大文字小文字を区別しない文字列データを考えてみよう．この場合，`"aBc" == "Abc"` であって欲しいが，これは型シノニムで

```haskell
type CaseInsensString = String
```

と定義するだけでは，

```haskell
>>> ("aBc" :: CaseInsensString) == ("Abc" :: CaseInsensString)
False
```

のままだ．そこで，`newtype` を使って，

```haskell
import qualified Data.Char as Char

newtype CaseInsensString = CaseInsens String

instance Eq CaseInsensString where
  CaseInsens s1 == CaseInsens s2 = go s1 s2
    where
      go []       []       = True
      go []       (_:_)    = False
      go (_:_)    []       = False
      go (c1:cs1) (c2:cs2) = Char.toLower c1 == Char.toLower c2 && go cs1 cs2
```

とすれば，

```haskell
>>> CaseInsens "aBc" == CaseInsens "Abc"
True
```

とできる．型シノニムは単なる `String` の別名なので，`String` と異なるインスタンスを新しく定義することはできない．それに対して，`newtype` によるデータ型は代数的データ型と同様に自由に定義することができる．そして，値コンストラクタ `CaseInsens` は単なる飾りであり，実行時には完全に無視されるため，`CaseInsensString` は動作としては `String` の別名としてみることができる．

`newtype` は型シノニムでの制約であった，

* 再帰的なデータ型が定義できない
* 型コンストラクタに対する部分適用ができない

といった問題も解決する．このように `newtype` は型シノニムの問題を改善したデータ型を定義するが，`data` 宣言と同様型シノニムでは起きなかった問題も一緒に顕在化させてしまう．

上の例で，`CaseInsens` は飾りだと言ったが，実際にはこの値コンストラクタは必要不可欠であり，重要な役割を持っている．例えば，

```haskell
>>> CaseInsens "aBc" == CaseInsens "Abc"
True
```

の例は，片方だけ

```haskell
>>> "aBc" == CaseInsens "Abc"
```

としてしまうと，コンパイルエラーになってしまう．なぜなら，`(==)` は2つの引数が同じ型の値である必要があり，`"aBc"` の型である `String` と `CaseInsens "Abc` の型である `CaseInsensString` は全く異なる型であるからだ．つまり，値コンストラクタ `CaseInsens` は，実行時には何の影響も与えないが，型システム上は全く異なる型の値であることを示すマーカーとなる．そして，型シノニムではデータ型は単なる別名であったが，`newtype` は `data` と同様全く新たな型として導入する道を選んだため，元の型として受け入れてもらうことが出来なくなってしまったのだ．

といっても，これは一長一短である．`data` と同様 `newtype` で作られた型は，型シノニムのように既存の関数を使い回すことができない．その反面，データの意味に沿わないプログラムを型によって弾くことができるという点は長所になる場合もある．例えば，`"aBc" == CaseInsens "Abc"` の例は，一体どのような結果を返すべきか一見して分からない．両者は単なる文字列と，大文字小文字を区別しない文字列という異なるデータを表しており，その比較は定義されないとするのが自然だろう．このような場合に，型シノニムでは定義されないことを表す方法はなかったが，`newtype` は元の型と異なる型を持つので，そのような仕組みを作ることができる．

さて，`newtype` において値コンストラクタは実行時に何の影響も及ぼさないことと，何故そうなっているかについて分かってもらえただろうか？ この影響は，パターンマッチ以外にも表れる．例えば，`newtype` の値コンストラクタに正格性フラグの機能はない．

```haskell
newtype StrictNewtype = StrictNewtype !Int
```

というプログラムは，Haskell では受け入れられない．なぜなら，これを受け入れた場合，値コンストラクタがあるかどうかによって実行時の動作が変わってしまうからだ．ただ，その他の `data` 宣言の機能は使用できる．`deriving` も使用できる．`newtype` で作られたデータ型は，元のデータ型のインスタンスを継承することはできない．全く新たな型を作ったため，更地の状態から始まる．ただし，`deriving` を使うことでインスタンスを用意に導出することは可能だ．ただ，標準クラスのインスタンスしか自動で導出できないため，自身で定義した型クラスなどのインスタンスは一から書く必要がある．そのことには，注意する必要があるだろう [^notice-deriving-extensions]．

[^notice-deriving-extensions]: GHC 拡張では，`deriving` 構文の拡張として強力な機能がいくつか搭載されている．特に `newtype` によるデータ型の場合は，`GeneralizedNewtypeDeriving` や `DerivingVia` 拡張を使えば，インスタンスの自動導出の範囲を大幅に拡大できる．

最後に少し応用的な `newtype` の使い方を紹介しよう．`newtype` は上のように目的に合わせて型を既存の型から作る他，型シノニムの制約によって定義できない型上の計算を実現するのにも使用できる．例えば，

```haskell
newtype Fix f = Fix (f (Fix f))
```

という変わったデータ型を使うと，型上の不動点演算をエミュレートできる．また，`newtype` を使うことで幽霊型による曖昧な型を避けることもできる．例えば，

```haskell
type WithAnn ann a = a

readShow :: (Read a, Show a) => WithAnn a String -> String
readShow s = show $ read s
```

を考える．この関数 `readShow` は，`WithAnn` で引数に `a` を使っているにもかかわらず `a` が曖昧な型になるため弾かれる．なぜなら，型シノニム `WithAnn a String` は `String` と書いてるのと同じであり，`readShow` は

```haskell
readShow :: (Read a, Show a) => String -> String
```

という型を持つのと同様になってしまうからだ．このため，制約だけに `a` が現れることになってしまい，曖昧な型になってしまう．この例のような，型シノニムが具体化されてしまうことで曖昧な型が生じる問題は，`newtype` を使用することで回避できる:

```haskell
newtype WithAnn ann a = WithAnn a

readShow :: (Read a, Show a) => WithAnn a String -> String
readShow (WithAnn s) = show $ read s
```

Haskell は型システム上は `WithAnn a String` が実行時に単なる `String` の別名として扱われることを知らず，これを1つの具体化された型として認識する．このため，実際には `a` が引数の値に何ら関与しない場合も，型 `a` を伴う型として残る．よって，この場合は `a` は曖昧な型にならず，`WithAnn a String` の `a` の部分にあてがわれる型から特定することができる．このように，型シノニムで早期に元となった型に具体化されることで生じる問題は，`newtype` を使うことで実際に値を作る箇所とパターンマッチの箇所での型計算に遅延させることができ，回避できる場合がある．

## まとめ

Haskell の3つのデータ型定義方法について紹介した．

型シノニムは，ある型に対してその別名を与えることで，データ型を定義するものだった．簡易で元の型に対する関数をそのまま流用でき，使いやすい反面，部分適用ができない，再帰的データ型が定義できない，型クラスのインスタンスにできないと言う制約があった．

代数的データ型は複数の型の積和によって全く新しいデータ型を定義するものだった．型シノニムであった制約を回避でき，新たな構造を導入できるが，関数の流用が困難な場合があり型シノニムとの使い分けが必要だった．

`newtype` によるデータ型は，型システム上は代数的データ型と，実行時の動作は型シノニムと同様といった，それぞれの中間をとったようなものだった．型シノニムのような関数の流用ができない場合はあるものの，その代わり型シノニムの制約を回避でき，型システム上は全く異なる振る舞いを行うことも可能だった．

これらは，それぞれが一長一短を持ち，目的にあった使い分けをする必要がある．この記事が，そのような場合の助けになればいいと思う．では，今回はこれで．

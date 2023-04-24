---
title: "【Ruby】Range#include?とRange#cover?の違い"
emoji: "😸"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["Ruby","CRuby","include","cover"]
published: false
---

# Range#include? と Range#cover?の違い

## 前提

本記事の内容は'MRI'いわゆる'CRuby'を対象としています。
また、Rubyのバージョンは3.2です。

## 概要

Range#include? と Range#cover? は以下のように、どちらも引数が指定した範囲に含まれるかどうかを判定する時に使われています。

```ruby
(1.1..2.3).include?(1.1)    # => true
(1.1..2.3).cover?(1.1)      # => true
```

しかし、その違いがよく分からなかったため、本記事ではその違いについて説明します。
本記事では、公式ドキュメントへの理解を深め、最終的にはCRubyのコードについても考察していきたいと思います。

## 結論

### Range#include?の考察

公式ドキュメントに対しての考察は以下のようになっています。

[Range\#include? \(Ruby 3\.2 リファレンスマニュアル\)](https://docs.ruby-lang.org/ja/latest/method/Range/i/include=3f.html)

> <=> メソッドによる演算により範囲内かどうかを判定するには Range#cover? を使用してください。

2文字以上の文字範囲を比較する時、Range#include?はRange#eachを使用して、引数とRangeの各要素を`==`で比較することで、範囲内に要素が存在するかどうか判定します。

1文字の時は、引数が1文字なら境界と比較し、違うならfalseになります。

> 始端・終端・引数が数値であれば、 Range#cover? と同様の動きをします。

C言語では、`cover?`の処理の関数`r_cover_p`を呼び出しています。

[ruby/range\.c at v3\_2\_1 · ruby/ruby](https://github.com/ruby/ruby/blob/v3_2_1/range.c#LC1811)

### Range#cover?の考察

公式ドキュメントに対しての考察は以下のようになっています。

[Range\#cover? \(Ruby 3\.2 リファレンスマニュアル\)](https://docs.ruby-lang.org/ja/latest/method/Range/i/cover=3f.html)

> Range#include? と異なり <=> メソッドによる演算により範囲内かどうかを判定します。 Range#include? は原則として離散値を扱い、 Range#cover? は連続値を扱います。（数値については、例外として Range#include? も連続的に扱います。）

引数とRangeの境界を`<=>`で比較して、範囲内にあるかを判定します。そのため、Range#include?よりも処理が早いです。

## 詳細

以降は、先の考察に至った理由を公式ドキュメント元に説明していきます。
### Range#include? と Range#cover?の文字列比較について

[Range\#cover? \(Ruby 3\.2 リファレンスマニュアル\)](https://docs.ruby-lang.org/ja/latest/method/Range/i/cover=3f.html)では以下のような文字列に対する、両者の比較があります。

```ruby
('b'..'d').include?('ba')   # => false
('b'..'d').cover?('ba')     # => true
```

以下、この違いについて考察します。

#### 文字列比較について

まず最初に文字列の比較方法について考察します。複数文字の文字列の比較に関しては以下の記事を参考にしました。

[文字列の大小比較をもう少し詳しく調べてみる（チェリー本の補足として） \- Qiita](https://qiita.com/jnchito/items/077f6d541d53152aa680)

こちらでは、文字比較を各バイトの文字コード（半角英数字ではASCIIコード）に対して、一要素ずつ比較していることが分かります。そのため、以下において'b'と'd'のバイトが比較され大小が決められています。

```ruby
'ba' <=> 'd' # => -1

'ba'.bytes # => [98, 97]
'd'.bytes #=> [100]
'b'.bytes[0] <=> 'd'.bytes[0] # => -1
```

実際、[ruby/string\.c at v3\_2\_1 · ruby/ruby](https://github.com/ruby/ruby/blob/v3_2_1/string.c#LC3603)では、同じ文字列の長さまでは`memcmp`で文字列を比較しています。また、同じ文字列が続いて長さが違う場合、長さの少ない方が小さいと判定されます。

```c
rb_str_cmp(VALUE str1, VALUE str2)
{
    long len1, len2;
    const char *ptr1, *ptr2;
    int retval;

    if (str1 == str2) return 0;
    RSTRING_GETMEM(str1, ptr1, len1);
    RSTRING_GETMEM(str2, ptr2, len2);
    if (ptr1 == ptr2 || (retval = memcmp(ptr1, ptr2, lesser(len1, len2))) == 0) {
        if (len1 == len2) {
            if (!rb_str_comparable(str1, str2)) {
                if (ENCODING_GET(str1) > ENCODING_GET(str2))
                    return 1;
                return -1;
            }
            return 0;
        }
        if (len1 > len2) return 1;
        return -1;
    }
    if (retval > 0) return 1;
    return -1;
}

```

#### 文字列比較におけるRange#include?とRange#cover?の動き

次に、二つのメソッドの内部の動きを見てみます。
内部の演算がわかりやすいように、以下のようにメソッドをオーバーライドします。

```ruby
module Foo
  def <=>(other)
    puts "#{self} <=> #{other}"
    super
  end
  def ==(other)
    puts "#{self} == #{other}"
    super
  end
  def ===(other)
    puts "#{self} === #{other}"
    super
  end
  def equal?(other)
    puts "#{self} equal? #{other}"
    super
  end
  def eql?(other)
    puts "#{self} eql? #{other}"
    super
  end
  def hash
    puts "hash #{self}"
    super
  end
  def each(*)
    puts "#{self} each"
    super
  end
end

String.prepend(Foo)
# Range.prepend(Foo) => 文字列比較の時は必要ない

puts "include?('ba')?"
puts ('b'..'d').include?('ba')
puts "\ncover?('ba')"
puts ('b'..'d').cover?('ba') 
```

この結果が、以下のようになります。

```terminal
include?('ba')?
false

cover?('ba')
b <=> ba
ba <=> d
true
```

これで、少なくとも`cover?`が境界としか比較していないことが分かります。
一方、`include?`は、これだとどういう動作をしているか分からないため、Cコードの方を見てみます。

[ruby/string\.c at v3\_2\_1 · ruby/ruby](https://github.com/ruby/ruby/blob/v3_2_1/string.c#LC5104)

```c
if (RSTRING_LEN(val) == 0 || RSTRING_LEN(val) > 1)
                return Qfalse;
            else {
                char b = *bp;
                char e = *ep;
                char v = *vp;

                if (ISASCII(b) && ISASCII(e) && ISASCII(v)) {
                    if (b <= v && v < e) return Qtrue;
                    return RBOOL(!RTEST(exclusive) && v == e);
                }
            }
```

上記の部分で分かる通り、対象範囲が1文字で引数が1文字でない時は問答無用でfalseになるみたいです。
範囲が2文字以上の場合は、次章で説明します。

### Range#include? と Range#cover?の日付の比較について

[Range\#cover? \(Ruby 3\.2 リファレンスマニュアル\)](https://docs.ruby-lang.org/ja/latest/method/Range/i/cover=3f.html)では以下のような日付に対する、両者の比較があります。

```ruby
require 'date'
(Date.today - 365 .. Date.today + 365).include?(DateTime.now)  #=> false
(Date.today - 365 .. Date.today + 365).cover?(DateTime.now)    #=> true
```

以下、この違いについて考察します。

#### 日付の比較

そもそも、DateとDateTimeの比較は以下のようになります。

```ruby
require 'date'
d = Date.new(2023, 4, 11)
puts d #=> #<Date: 2023-04-11 ((2460046j,0s,0n),+0s,2299161j)>

dt_00_00 = DateTime.new(2023, 4, 11, 0, 0)
puts dt_00_00 #=> #<DateTime: 2023-04-11T00:00:00+00:00 ((2460046j,0s,0n),+0s,2299161j)>

dt_00_01 = DateTime.new(2023, 4, 11, 0, 1)
puts dt_00_01 #=> #<DateTime: 2023-04-11T00:01:00+00:00 ((2460046j,60s,0n),+0s,2299161j)>

# 両者は等しい
puts d <=> dt_00_00 #=> 0

# dはdt_00_01より小さい（時刻部分を無視するなら戻り値は0になりそう）
puts d <=> dt_00_01 #=> -1
```

このことから、以下の`cover?`は日付の範囲に対して、0時00分で比較していると予想できます。

```ruby
(Date.today - 365 .. Date.today + 365).cover?(DateTime.now) #=> true
```

#### 日付の比較におけるRange#include?とRange#cover?の動き

次に、二つのメソッドの内部の動きを見てみます。
簡単のために、日付の範囲は`(Date.today - 1 .. Date.today + 1)`とします。
こちらも内部の演算がわかりやすいように、以下のようにメソッドをオーバーライドします。

```ruby
require 'date'

module Foo
  def <=>(other)
    puts "#{self} <=> #{other}"
    super
  end
  def ==(other)
    puts "#{self} == #{other}"
    super
  end
  def ===(other)
    puts "#{self} === #{other}"
    super
  end
  def equal?(other)
    puts "#{self} equal? #{other}"
    super
  end
  def eql?(other)
    puts "#{self} eql? #{other}"
    super
  end
  def hash
    puts "hash #{self}"
    super
  end
  def each(*)
    puts "#{self} each"
    super
  end
end

Date.prepend(Foo)
Range.prepend(Foo)

puts "(Date.today - 1 .. Date.today + 1).include?(DateTime.now)"
puts (Date.today - 1 .. Date.today + 1).include?(DateTime.now)

puts "---"

puts "(Date.today - 1 .. Date.today + 1).cover?(DateTime.now)"
puts (Date.today - 1 .. Date.today + 1).cover?(DateTime.now) 
```

この実行結果は以下となります。

```terminal
(Date.today - 1 .. Date.today + 1).include?(DateTime.now)
2023-04-23 <=> 2023-04-25
2023-04-23..2023-04-25 each
2023-04-23 <=> 2023-04-25
2023-04-23 == 2023-04-24T19:10:22+09:00
2023-04-23 <=> 2023-04-24T19:10:22+09:00
2023-04-24 <=> 2023-04-25
2023-04-24 == 2023-04-24T19:10:22+09:00
2023-04-24 <=> 2023-04-24T19:10:22+09:00
2023-04-25 <=> 2023-04-25
2023-04-25 == 2023-04-24T19:10:22+09:00
2023-04-25 <=> 2023-04-24T19:10:22+09:00
false
---
(Date.today - 1 .. Date.today + 1).cover?(DateTime.now)
2023-04-23 <=> 2023-04-25
2023-04-23 <=> 2023-04-24T19:10:22+09:00
2023-04-24T19:10:22+09:00 <=> 2023-04-25
true
```

まず、`cover?`の方から見ていくと、こちらは文字列の時と同じく境界と比較していることが分かります。
一方で、`include?`は、eachにより範囲内の日付それぞれと'2023-04-24T19:10:22+09:00'を比較していることが分かります。

ちなみに、

```terminal
2023-04-23 <=> 2023-04-25
2023-04-23..2023-04-25 each
2023-04-23 <=> 2023-04-25
```

でeachの前後で同じものを比較しているのは、範囲オブジェクトを初期化するタイミングで [ruby/range\.c at v3\_2\_1 · ruby/ruby](https://github.com/ruby/ruby/blob/v3_2_1/range.c#L52) の部分で呼ばれています。
この処理は範囲オブジェクトとして初期化可能かを判定しています。
たとえば`Date.today + 1 .. '日付でない文字'`のような範囲オブジェクトを作ると、`Date.today + 1 <=> '日付でない文字'` （Cではrb_funcall(beg, id_cmp, 1, end)）で比較不能のnilが返ります。

### Range#include? と Range#cover? のベンチマーク結果

最後に、両メソッドの実行速度を比較してみます。
以下のようにベンチマークのコードを用意します。

```ruby
require 'benchmark'

time = Benchmark.realtime do
  ('ab'..'ad').include?('ac')
end
puts "('ab'..'ad').include?('ac'): #{time}s"


time = Benchmark.realtime do
  ('ab'..'ad').cover?('ac')
end
puts "('ab'..'ad').cover?('ac'): #{time}s"
```

実行結果は以下のとおりです。

```terminal
('ab'..'ad').include?('ac'): 8.00006091594696e-06s
('ab'..'ad').cover?('ac'): 1.00000761449337e-06s
```

上記より、確かに`include?`の方が遅いと思われます。この原因は前節でも述べたように、`each`で各要素を比較していることにあると思われます。
ちなみに数値に対して、比較すると以下のようになります。

```ruby
time = Benchmark.realtime do
  (1..3).include?(2)
end
puts "(1..3).include?(2): #{time}s"


time = Benchmark.realtime do
  (1..3).cover?(2)
end
puts "(1..3).cover?(2): #{time}s"
```

以下が実行結果です。

```terminal
(1..3).include?(2): 1.00000761449337e-06s
(1..3).cover?(2): 1.00000761449337e-06s
```

数値の場合は、`include?`もC言語上では`cover?`と同じ関数を呼びに行っているので、実行時間も同じになります（たまにズレるので参考値です）。

## まとめ

結論は最初に挙げた通りです。
まだ、Rubyを習い始めてから1ヶ月ですが、裏のC言語を読むと少し理解が深まる気がしました。
やはり、公式ドキュメントとRuby自体のコードをしっかり読むのは大事だと感じたテーマだったと思います。

## 分からなかったこと

本題とはずれますが、`(b..d).include?(d)`の動作に関して、疑問点があります。
「文字列比較におけるRange#include?とRange#cover?の動き」の節において、

[ruby/string\.c at v3\_2\_1 · ruby/ruby](https://github.com/ruby/ruby/blob/v3_2_1/string.c#LC5104)

```c
if (RSTRING_LEN(val) == 0 || RSTRING_LEN(val) > 1)
                return Qfalse;
            else {
                char b = *bp;
                char e = *ep;
                char v = *vp;

                if (ISASCII(b) && ISASCII(e) && ISASCII(v)) {
                    if (b <= v && v < e) return Qtrue;
                    return RBOOL(!RTEST(exclusive) && v == e);
                }
            }
```

を引用しました。ここから、`(b..d).include?(d)`のような1文字同士の比較の場合、引数を境界との文字のバイトで比較しているように見えます。実際、メソッドをオーバーライドすると境界としか比較をしていませんでした。

しかし、ここで述べた処理の下にある`rb_str_upto_each`関数内でも以下の処理をしています。

[ruby/string\.c at v3\_2\_1 · ruby/ruby](https://github.com/ruby/ruby/blob/v3_2_1/string.c#LC4969)

```c
/* single character */
    if (RSTRING_LEN(beg) == 1 && RSTRING_LEN(end) == 1 && ascii) {
        char c = RSTRING_PTR(beg)[0];
        char e = RSTRING_PTR(end)[0];

        if (c > e || (excl && c == e)) return beg;
        for (;;) {
            if ((*each)(rb_enc_str_new(&c, 1, enc), arg)) break;
            if (!excl && c == e) break;
            c++;
            if (excl && c == e) break;
        }
        return beg;
    }
```

コメントだけ読むと、ここでも1文字の比較をしています。
こちらの処理ではfor文を回しており、各要素と引数を比較してそうですが、eachのコールバック関数がよく分からないため、具体的に何をやっているのか分かっていません。

もし、上記の処理（特にeachのコールバック関数の動作）に関して詳しい方がおりましたら、コメント頂きたいです。
また、なぜ`include?`において1文字の比較に対して二つ処理があるのかも、合わせて教えていただきたいです。

よろしくお願い致します。

## 謝辞

ここまでの内容は、自分が所属しているプログラミングスクールであるFjord Boot Campにおいて質問した回答が元となっています。
そのため、以下のお二方の回答を参考にしております。

- [伊藤淳一 (id:JunichiIto)さん](https://blog.jnito.com/)
- [はるぐち　ゆうま (id:haruguchi_yuma)さん](https://haruguchi-yuma.hatenablog.com/)

本当にありがとうございました！！

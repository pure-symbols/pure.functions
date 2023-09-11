At the first, by [*cs3110 here*](https://cs3110.github.io/textbook/chapters/hop/higher_order.html?highlight=pipe), I got a simple define of the `pipe` operator in OCaml: 

我最初从 CS3110 了解到 `pipe` 运算符的定义：

~~~ ocaml
let pipeline x f = f x
let (|>) = pipeline
let x = 5 |> add 3

(* val pipeline : 'a -> ('a -> 'b) -> 'b = <fun> *)
(* val ( |> ) : 'a -> ('a -> 'b) -> 'b = <fun> *)
(* val x : int = 8 *)
~~~

so I made: 

所以我也做了一个：

~~~ typescript
type pipe = <T,> (x: T) => <R,> (f: Fn<T, R>) => R ;
const pipe: pipe = <T,> (x: T) => <R,> (f: Fn<T, R>): R => f(x) ;
~~~

But it is not a middle operator, its usage will be like `pipe ( pipe ( pipe (5) (add (3) ) ) (add (4) ) ) (add (5) )`, its original written `add (5) (add (4) (add (3) (5) ) )` is much clear !!

但它不是一个中缀运算符，所以用起来会是 `pipe ( pipe ( pipe (5) (add (3) ) ) (add (4) ) ) (add (5) )` 这样，它反而比原来的 `add (5) (add (4) (add (3) (5) ) )` 更繁琐！（虽说如此但这也是一小步）

I cannot make a middle operator in TS just like `let (|>) = pipeline` in OCaml ... then, I found that: If `add (1) (2)` got `3`, then `add (1)` **is just a function** right !? So can I make a pipeline function, but not return a result, we make it return a new function that can support us to give another one-parameter function to it ?

我也没法在 TS 里像上面 OCaml 那样自定义一个中缀运算符。但，这时候，我就发现，柯里化带来了特殊的力量：当它没有完全被应用时，它就总是个单参函数（闭包）。所以，我是不是能做一个也返回这种闭包的管线呢？说干就干。

So I found this calling form: 

它用起来会是这样：

~~~ typescript
pipeline (5)
    (add (3) ) ()
    (add (4) ) ()
    (add (5) ) ()
    ...
~~~

so the define be like: 

我试着搞了搞它的定义：

~~~ typescript
const pipeline = <T,> (x: T) => pipework (x)
const pipework = <T,> (x: T) => <R,> (f: Fn<T, R>) => () => pipeline(f(x))
~~~

only switch `f(x)` in `pipe` to `() => pipeline(f(x))`.

其中只是替换了最后一小部分，并删掉了返回类型标注。

But ... how can I got the final value in such calling form ? There need a pair to store the result also ... then, the calling form shall be like: 

但这样我好像没有一个办法来取得结果，因为一直都是返回一个闭包。此时我又想到了元组，所以这里就用 Pair 来存放每一步的结果。那用的时候就得像这样：

~~~ typescript
pipeline (5)
    (add (3) ) .tail()
    (add (4) ) .tail()
    (add (5) ) .head() // get the result: 17
~~~

so the define be like: 

那么定义就得：

~~~ typescript
const pipeline = <T,> (x: T) => pipework (x)
const pipework = <T,> (x: T) => <R,> (f: Fn<T, R>) => Pair (f(x)) (() => pipeline(f(x)))
~~~

and here `pipeline` is same as `pipework` ... if you have knowledge about currying you will understand that ... so we just have: 

显而易见， `pipeline` 就是 `pipework` …… 如果你能明白柯里化（或者说闭包的效果）是什么，你就能明白我在说什么 …… 那么简化：

~~~ typescript
const pipeline = <T,> (x: T) => <R,> (f: Fn<T, R>) => Pair (f(x)) (() => pipeline(f(x)))
~~~

and, not all the `head` be needed, so change it be lack of evaluation (means lazy): 

并且，并非所有的中间值都是要取得的。我希望当用户需要的时候再计算（评估在这里要匮乏）（这也就是一些人常说的惰性），那么就：

~~~ typescript
const pipeline = <T,> (x: T) => <R,> (f: Fn<T, R>) => Pair (() => f(x)) (() => pipeline(f(x)))
~~~

That's OK.

这就是我想要的定义了。接下来可以像上面那样的调用形式去使用了。

See them on [*playground*](https://www.typescriptlang.org/play?#code/PTAEBMFMAdIO3AZwLACg0BcCetQDE5QAeAFQBpQAlAPlAF5QAKADwC5QSBKe2y0Abkw5IoAAoBDAJYAnYgAlI48BRJSANrQYBvUAAtF4dgqUUM69qslrQAXwFo0AYwD2cRBjFTZDNEWPLaRn0lIwNuOlpfSzUyQLMrC3VONHYJGT8DFXVNSNRQfNA0AqYdYMM9TNB4tXZq225xRE90-yyrWkF0LodUEFBoSVg1STgRKAAzEcgnV3d+wchh0fpickC2DnDaIkpYpnH2AlIKGi3m2UZGM-GWTm5Ls4GhqcYb5jvuTrQ+xwBXaWkWBGAHNQO4sGoRONnNJ2C43B4lOAVoxxOw4L8ALYAI0g0go2PRWNx0jO4lAAGpQNj7Kh4XMkSi0aAMTi8WdGISWcT2TxQOSqTSvt1emBEM4Zm5nJCAHRqZzAtCMIoFJ6LKZMACsyTyxQKqPAyMYAGZuNwZdUriq9UxGYwACxm0AW9RW3U2g1G7Wgc1lN2cfigH6zaWQOUK0DOX4YdgARgA7D0+jC0GqliJGNrrbbDUxTT7nZadcVPUxHQWXVY3SW7QA2J2VtTV-X00Ph4G+gxXQPBqWy+WgqMx0CxgAcaCAA)

可在 [Playground](https://tsplay.dev/WKKLyW) 查看完整代码及其验证。

--------

If you want a type comment here, you shall see this: `<T>(x: T) => <R>(f: Fn<T, R>) => Pair<() => R, () => <R>(f: Fn<R, R>) => Pair<() => R, () => <R>(f: Fn<R, R>) => Pair<() => R, () => <R>(f: Fn<R, R>) => Pair<() => R, () => <R>(f: Fn<...>) => Pair<...>>>>>` .

这时候，如果看一下类型标记，你会看到一个写不完的类型标记。

I simplified it by these three things: 

我用三个东西概括了这个写不完的类型标记：

~~~ typescript
type Pipework <T> = () => Pipeline<T> ;
type Pipeline <T> = <R> (f: Fn<T, R>) => Rivulet<R> ;
type Rivulet <T> = Pair<T, Pipework<T> > ;
~~~

And, the type of this function can be this: 

然后，函数的类型就被简化为了这个：

~~~ typescript
type pipeline = <T> (x: T) => Pipeline<T> ;
~~~

After the formats and some others, we get this result (we've see in our code) finally: 

再格式化一下（并做了一些补充），这就是最终版本的了：

~~~ typescript
const Rivulet = 
<T,> (head: T) => 
(tail: Pipework<T>)
: Rivulet<T> => 
    
    Pair (head) (tail) ;

export 
const Pipeline: pipeline = 
<T,> (x: T): Pipeline<T> => 
    
    <R,> (f: Fn<T, R>) => Rivulet (pipe (x) (f)) (() => Pipeline (pipe (x) (f)) ) ;
~~~

That's all.

就这样。

Thanks for the *lexical closure* feature.

感谢语言提供的 *词法闭包* 特性。没有对这一特性的支持就做不了这些事。

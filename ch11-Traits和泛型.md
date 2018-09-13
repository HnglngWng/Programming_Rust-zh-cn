# Traits和泛型(Traits and Generics)

> 计算机科学家倾向于能够处理不一致的结构--情况1,情况2,情况3--而数学家倾向于想要一个统一公理来管理整个系统.
> --唐纳德 高德纳

原文

> [A] computer scientist tends to be able to deal with nonuniform structures--case 1, case 2, case 3--while a mathematician will tend to want one unifying axiom that governs an entire system.
> --Donald Knuth

编程的一个重大发现是,编写可以处理许多不同类型的值的代码是可能的,甚至是尚未发明的类型(even types that haven't been invented yet).这里有两个例子:

- `Vec<T>`是泛型的:你可以创建任何类型值的向量,包括程序中定义的`Vec`作者从未预料到的类型.

- 很多东西都有`.write()`方法,包括`File`和`TcpStream`.你的代码可以通过引用获取writer(任何writer),并向其发送数据.你的代码不必关心它是哪种类型的writer.之后,如果有人添加了一种新类型的writer你的代码就会支持它.

当然,这种能力对Rust来说并不新鲜.它被称为 *多态(polymorphism)* ,它是20世纪70年代热门的新编程语言技术.到目前为止它实际上是普遍的.Rust用两个相关特性支持多态性:traits和泛型.这些概念对于许多程序员来说都很熟悉,但Rust采用了一种受Haskell的类型类(typeclasses)启发的新方法.

*Traits*是Rust对接口或抽象基类的理解.首先,它们看起来就像Java或C#中的接口.写入字节的trait称为`std::io::Write`,它在标准库中的定义如下所示:

```Rust
trait Write {
    fn write(&mut self, buf: &[u8]) -> Result<usize>;
    fn flush(&mut self) -> Result<()>;
    fn write_all(&mut self, buf: &[u8]) -> Result<()> { ... }
    ...
}
```

这个trait提供了几种方法;我们只展示了前三个.

标准类型`File`和`TcpStream`都实现了`std::io::Write`.`Vec`也是如此.这三种类型都提供名为`.write()`,`.flush()`等的方法.使用writer而不关心其类型的代码如下所示:

```Rust
use std::io::Write;

fn say_hello(out: &mut Write) -> std::io::Result<()> {
    out.write_all(b"hello world\n")?;
    out.flush()
}
```

`out`的类型是`&mut Write`,意思是"对实现`Write`trait的任何值的可变引用".

```Rust
use std::fs::File;

let mut local_file = File::create("hello.txt")?;
say_hello(&mut local_file)?;  // works

let mut bytes = vec![];
say_hello(&mut bytes)?;  // also works
assert_eq!(bytes, b"hello world\n");
```

本章首先介绍如何使用traits,它们是如何工作的以及如何定义自己的traits.但到目前为止，我们所暗示的traits还有很多.我们将使用它们为现有类型添加扩展方法,甚至是`str`和`bool`等内置类型.我们将解释为什么向类型添加trait不需要额外的内存以及如何在没有虚方法调用开销的情况下使用trait.我们将看到内置的trait是Rust为操作符重载和其他功能提供的语言的钩子(hook).我们将介绍`Self`类型,关联方法和关联类型,Rust从Haskell中提取的三个特性,它可以优雅地解决其他语言使用变通方法和黑客(hacks)解决的问题.

*泛型(Generics)* 是Rust中的另一种多态性.与C++模板一样,泛型函数或类型可以与许多不同类型的值一起使用.

```Rust
/// Given two values, pick whichever one is less.
fn min<T: Ord>(value1: T, value2: T) -> T {
    if value1 <= value2 {
        value1
    } else {
        value2
    }
}
```

此函数中的`<T: Ord>`表示`min`可以与任何实现`Ord`trait的类型`T`的参数一起使用,即任何有序类型.编译器为你实际使用的每种类型`T`生成自定义机器代码.

泛型和traits密切相关.在使用`<=`运算符比较类型`T`的两个值之前,Rust使我们在前面声明`T: Ord`要求(称为 *限制(bound)* ).因此,我们还将讨论`&mut Write`和`<T：Write>`如何相似,它们如何不同,以及如何在这两种使用traits的方式之间进行选择.

## 使用Traits(Using Traits)

trait是任何给定类型可能支持或不支持的功能.大多数情况下,trait代表一种能力:一种类型可以做的事情.

- 实现`std::io::Write`的值可以写出字节.

- 实现`std::iter::Iterator`的值可以生成一系列值.

- 实现`std::clone::Clone`的值可以在内存中创建自身的克隆.

- 实现`std::fmt::Debug`的值可以使用带有`{:?}`格式说明符的`println!()`打印.

这些trait都是Rust标准库的一部分,许多标准类型都实现了它们.

- `std::fs::File`实现`Write`trait;它将字节写入本地文件.`std::net::TcpStream`写入网络连接.`Vec<u8>`也实现了`Write`.对字节向量的每个`.write()`调用都会将一些数据附加到末尾.

- `Range<i32>`(`0..10`的类型)实现`Iterator`trait,与切片,哈希表等相关联的一些迭代器类型也是如此.

- 大多数标准库类型都实现了`Clone`.例外主要是像`TcpStream`这样的类型,它们不仅仅代表内存中的数据.

- 同样,大多数标准库类型都支持`Debug`.

关于trait方法有一个不寻常的规则:trait本身必须在作用域内.否则,它的所有方法都被隐藏.

```Rust
let mut buf: Vec<u8> = vec![];
buf.write_all(b"hello")?;  // error: no method named `write_all`
```

在这种情况下,编译器会打印一条友好的错误消息,建议添加使用`std::io::Write;`确实解决了这个问题:

```Rust
use std::io::Write;

let mut buf: Vec<u8> = vec![];
buf.write_all(b"hello")?;  // ok
```

Rust有这个规则,因为正如我们将在本章后面看到的那样,你可以使用traits将新方法添加到任何类型--甚至标准库类型,如`u32`和`str`.第三方crates可以做同样的事情.显然,这可能导致命名冲突!但是由于Rust让你导入你计划使用的traits,crates可以自由地利用这个超能力,在实践中冲突是罕见的.

`Clone`和`Iterator`方法在没有任何特殊导入的情况下工作的原因是,它们默认情况下始终在作用域内:它们是标准前置的一部分,Rust自动导入到每个模块的名称.事实上,前置主要是精心挑选的traits选择.我们将在第13章中介绍其中的许多内容.

C++和C#程序员已经注意到trait方法就像虚方法.尽管如此,类似上面显示的调用速度很快,与任何其他方法调用一样快.简单地说,这里没有多态性.很明显,`buf`是一个向量,而不是文件或网络连接.编译器可以发出对`Vec<u8>::write()`的简单调用.它甚至可以内联该方法.(C++和C#通常都会这样做,虽然子类化的可能性有时会排除这种情况.)只有通过`&mut Write`调用才会产生虚方法调用的开销.

### Trait对象(Trait Objects)

在Rust中使用traits编写多态代码有两种方法:trait对象和泛型.我们首先介绍trait对象,然后在下一节中转向泛型.

Rust不允许`Write`类型的变量:

```Rust
use std::io::Write;

let mut buf: Vec<u8> = vec![];
let writer: Write = buf;  // error: `Write` does not have a constant size
```

变量的大小必须在编译时知道,并且实现`Write`的类型可以是任何大小.

如果你来自C#或Java,这可能会很惊讶,但原因很简单.在Java中,`OutputStream`类型的变量(类似于`std::io::Write`的Java标准接口)是对实现`OutputStream`的任何对象的引用.它是一个引用,不言而喻的事实.它与C#和大多数其他语言中的接口相同.

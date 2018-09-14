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

变量的大小必须在编译时知道,但是实现`Write`的类型可以是任何大小.

如果你来自C#或Java,这可能会很惊讶,但原因很简单.在Java中,`OutputStream`类型的变量(类似于`std::io::Write`的Java标准接口)是对实现`OutputStream`的任何对象的引用.它是一个引用,不言而喻的事实.它与C#和大多数其他语言中的接口相同.

我们在Rust中想要的是同样的东西,但在Rust中,引用是显式的:

```Rust
let mut buf: Vec<u8> = vec![];
let writer: &mut Write = &mut buf;  // ok
```

对trait类型(如`writer`)的引用称为 *trait对象(trait object)* .与任何其他引用一样,trait对象指向某个值,它有生命周期,并且可以是`mut`或共享的.

使trait对象与众不同的是,Rust通常在编译时不知道所引用的对象的类型.因此,trait对象包含关于引用的对象的类型的一些额外信息.这完全是为了Rust在幕后使用的:当你调用`writer.write(data)`时,Rust需要类型信息来根据`*writer`的类型动态调用正确的`write`方法.你不能直接查询类型信息,Rust不支持从trait对象`&mut Writer`向下转换到像`Vec<u8>`这样的具体类型.

### Trait对象布局(Trait Object Layout)

在内存中,trait对象是一个胖指针,由指向值的指针和指向表示该值类型的表的指针组成.因此,每个trait对象占用两个机器字,如图11-1所示.

*图11-1. 内存中的Trait对象*

C++也有这种运行时类型信息.它被称为 *虚函数表(virtual table)* 或 *(虚表)vtable* .在Rust中,与在C++中一样,虚表在编译时生成一次,并由相同类型的所有对象共享.图11-1中以深灰色显示的所有内容(包括虚表)都是Rust的私有实现细节.同样,这些不是你可以直接访问的字段和数据结构.相反,当你调用trait对象的方法时,该语言会自动使用虚表,以确定要调用的实现.

经验丰富的C++程序员会注意到Rust和C++使用的内存有点不同.在C++中,虚表指针(或 *vptr* )存储为结构的一部分.Rust使用胖指针代替.结构本身只包含其字段. 这样,一个结构可以实现几十个特征,而不必包含几十个虚表指针.甚至像`i32`这样的类型,它们不足以容纳虚表指针,也可以实现trait.

Rust会在需要时自动将普通引用转换为trait对象.这就是为什么我们能够在这个例子中将`&mut local_file`传递给`say_hello`:

```Rust
let mut local_file = File::create("hello.txt")?;
say_hello(&mut local_file)?;
```

`&mut local_file`的类型是`&mut File`,`say_hello`的参数类型是`&mut Write`.由于`File`是一种writer,Rust允许这样做,自动将普通引用转换为trait对象.

同样,Rust会很乐意将`Box<File>`转换为`Box<Write>`,这是一个拥有堆中writer的值:

```Rust
let w: Box<Write> = Box::new(local_file);
```

`Box<Write>`,和`&mut Write`一样,是一个胖指针:它包含writer本身的地址和虚表指针的地址.其他指针类型(如`Rc<Write>`)也是如此.

这种转换是创建trait对象的唯一方法.计算机实际上在这里做的非常简单.在转换发生时,Rust知道引用的对象的真实类型(在本例中为`File`),因此它只是添加了相应虚表的地址,将常规指针转换为胖指针.

### 泛型函数(Generic Functions)

在本章的开头,我们展示了一个将trait对象作为参数的`say_hello()`函数.让我们将该函数重写为泛型函数:

```Rust
fn say_hello<W: Write>(out: &mut W) -> std::io::Result<()> {
    out.write_all(b"hello world\n")?;
    out.flush()
}
```

只有类型签名发生了变化:

```Rust
fn say_hello(out: &mut Write)         // plain function

fn say_hello<W: Write>(out: &mut W)   // generic function
```

短语`<W：Write>`是使函数泛型的原因.这是一个 *类型参数(type parameter)* .这意味着在整个函数体中,`W`代表实现`Write`trait的某种类型.按照惯例,类型参数通常是单个大写字母.

`W`代表哪种类型取决于泛型函数的使用方式:

```Rust
say_hello(&mut local_file)?;  // calls say_hello::<File>
say_hello(&mut bytes)?;       // calls say_hello::<Vec<u8>>
```

当你将`&mut local_file`传递给泛型的`say_hello()`函数时,你正在调用`say_hello::<File>()`.Rust为此函数生成调用`File::write_all()`和`File::flush()`的机器码.当你传递`&mut byte`时,你正在调用`say_hello::<Vec<u8>>()`.Rust为此版本的函数生成单独的机器码,调用相应的`Vec<u8>`方法.在这两种情况下,Rust都会根据参数的类型推断出类型`W`.你可以总是拼写出类型参数:

```Rust
say_hello::<File>(&mut local_file)?;
```

但它很少需要,因为Rust通常可以通过查看参数来推断出类型参数.这里,`say_hello`泛型函数需要一个`&mut W`参数,我们传递给它一个`&mut File`因此Rust推断出`W = File`.

如果你正在调用的泛型函数没有任何提供有用线索的参数,那么你可能需要拼写出来:

```Rust
// calling a generic method collect<C>() that takes no arguments
let v1 = (0 .. 1000).collect();  // error: can't infer type
let v2 = (0 .. 1000).collect::<Vec<i32>>(); // ok
```

有时我们需要一个类型参数的多种能力.例如,如果我们想打印出向量中前10个最常见的值,我们需要这些值是可打印的:

```Rust
use std::fmt::Debug;

fn top_ten<T: Debug>(values: &Vec<T>) { ... }
```

但这还不够好.我们如何计划确定哪些值最常见?通常的方法是将值用作哈希表中的键.这意味着值需要支持`Hash`和`Eq`操作.`T`上的限制必须包括这些以及`Debug`.其语法使用`+`号:

```Rust
fn top_ten<T: Debug + Hash + Eq>(values: &Vec<T>) { ... }
```

有些类型实现`Debug`,有些实现`Hash`,有些支持`Eq`;还有一些,比如`u32`和`String`,实现了所有这三个,如图11-2所示.

*图11-2. 作为类型集合的Traits*

类型参数也可以根本没有限制,但如果没有为其指定任何限制,则无法对值进行多少操作.你可以移动它.你可以把它放在一个盒子(box)或向量中.就是这样.

泛型函数可以有多个类型参数:

```Rust
/// Run a query on a large, partitioned data set.
/// See <http://research.google.com/archive/mapreduce.html>.
fn run_query<M: Mapper + Serialize, R: Reducer + Serialize>(
    data: &DataSet, map: M, reduce: R) -> Results
{ ... }
```

正如这个例子所示,限制可能会很长,以至于眼睛很难看.Rust提供了另一种语法,使用关键字`where`:

```Rust
fn run_query<M, R>(data: &DataSet, map: M, reduce: R) -> Results
    where M: Mapper + Serialize,
          R: Reducer + Serialize
{ ... }
```

类型参数`M`和`R`仍然在前面声明,但是限制被移动到单独的行.在泛型结构,枚举,类型别名和方法上也允许使用这种`where`子句--允许任何限制.

当然,`where`子句的替代方法是保持简单:找到一种编写程序的方法,而不是非常集中地使用泛型.

第105页的"接受引用作为参数(Receiving References as Parameters)"介绍了生命周期参数的语法.泛型函数可以包含生命周期参数和类型参数.生命周期参数先出现.

```Rust
/// Return a reference to the point in `candidates` that's
/// closest to the `target` point.
fn nearest<'t, 'c, P>(target: &'t P, candidates: &'c [P]) -> &'c P
    where P: MeasureDistance
{
    ...
}
```

这个函数有两个参数,`target`和`candidate`.两者都是引用,我们给它们不同的生命周期`'t`和`'c`(如第111页的"不同的生命周期参数(Distinct Lifetime Parameters)"中所述).此外,该函数适用于实现`MeasureDistance`trait的任何类型`P`,因此我们可以在一个程序中使用`Point2d`值而在另一个程序中使用`Point3d`值.

生命周期永远不会对机器码产生任何影响.对`nearest()`的两次调用使用相同的类型`P`,但生命周期不同,将调用相同的编译函数.只有不同的类型才会导致Rust编译泛型函数的多个副本.

当然,函数不是Rust中唯一的泛型代码.

- 我们已经在第202页的"泛型结构(Generic Structs)"和第218页的"泛型枚举(Generic Enums)"中介绍了泛型类型.

- 单个方法可以是泛型的,即使它定义的类型不是泛型的:

```Rust
impl PancakeStack {
    fn push<T: Topping>(&mut self, goop: T) -> PancakeResult<()> {
        ...
    }
}
```

- 类型别名也可以是泛型的:

```Rust
type PancakeResult<T> = Result<T, PancakeError>;
```

- 我们将在本章后面介绍泛型trait.

此节介绍的所有功能--限制,`where`子句,生命周期参数等--都可用于所有泛型项,而不仅仅是函数.

### 使用哪种(Which to Use)

选择使用trait对象还是泛型代码是很微妙的.由于这两个特性都基于trait,因此它们有很多共同之处.

当你需要混合类型的值的集合时,Trait对象是正确的选择.制作泛型沙拉在技术上是可行的:

```Rust
trait Vegetable {
    ...
}

struct Salad<V: Vegetable> {
    veggies: Vec<V>
}
```

但这是一个相当严格的设计.每个这样的沙拉完全由单一类型的蔬菜组成.并不是每个人都适合这种东西.你的一位作者曾经花了14美元购买了一份`Salad<IcebergLettuce>`,但从未完全体会过这种经历.

我们怎样才能做出更好的沙拉?由于`Vegetable`值可以是不同的大小,我们不能问Rust要一个`Vec<Vegetable>`:

```Rust
struct Salad {
    veggies: Vec<Vegetable>  // error: `Vegetable` does not have
                             //        a constant size
}
```

Trait对象是解决方案:

```Rust
struct Salad {
    veggies: Vec<Box<Vegetable>>
}
```
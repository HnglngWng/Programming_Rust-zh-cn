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

*图11-1. 内存中的Trait对象.*

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

*图11-2. 作为类型集合的Traits.*

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

每个`Box<Vegetable>`可以拥有任何类型的蔬菜,但盒子本身有一个恒定的大小--两个指针--适合存储在向量中.除了在一个人的食物中有盒子的不幸混合比喻之外,这正是所需要的,它也可以用于绘图应用程序中的形状,游戏中的怪物,网络路由器中的可插拔路由算法,等等.

使用trait对象的另一个可能原因是减少编译代码的总量.Rust可能需要多次编译泛型函数,对于它使用的每种类型都要编译一次.这可能会使二进制文件变大,这种现象在C++圈子中称为 *代码膨胀* .现在,内存非常充足,我们大多数人都奢侈地忽略了代码大小;但确实存在受限制的环境.

在涉及沙拉或微控制器的情况之外,泛型比trait对象有两个重要的优势,结果是在Rust中,泛型是更常见的选择.

第一个优势是速度.每次Rust编译器为泛型函数生成机器码时,它都知道它正在使用哪种类型,因此它在那时就知道要调用哪个`writer`方法.不需要动态分发.

引言中显示的泛型`min()`函数与我们编写单独的函数`min_u8`,`min_i64`,`min_string`等一样快.编译器可以像任何其他函数一样内联它,因此在发布版本中,对`min::<i32>`的调用可能只有两个或三个指令.具有常量参数的调用(如`min(5, 3)`)将更快:Rust可以在编译时对其进行求值,因此根本没有运行时成本.

或者考虑这个泛型函数调用:

```Rust
let mut sink = std::io::sink();
say_hello(&mut sink)?;
```

`std::io::sink()`返回一个类型为`Sink`的writer,它悄悄地丢弃写入到它的所有字节.

当Rust为此生成机器代码时,它可以发出调用`Sink::write_all`的代码,检查错误,然后调用`Sink::flush`.这就是泛型函数体所要做的.

或者,Rust可以查看这些方法并意识到以下内容:

- `Sink::write_all()`什么也不做.

- `Sink::flush()`什么也不做.

- 两种方法都没有返回错误.

简而言之,Rust拥有完全优化此函数所需的所有信息.

将其与trait对象的行为进行比较.Rust直到运行时才知道trait对象所指向的值的类型.因此,即使你传递了一个`Sink`,调用虚方法和检查错误的开销仍然适用.

泛型的第二个优势是并非每个trait都可以支持trait对象.Traits支持几种功能,例如静态方法,只适用于泛型:它们完全排除了trait对象.我们遇到时,将指出这些功能.

## 定义和实现Traits(Defining and Implementing Traits)

定义trait很简单.给它命名并列出trait方法的类型签名.如果我们正在编写游戏,我们可能会有这样一个trait:

```Rust
/// A trait for characters, items, and scenery -
/// anything in the game world that's visible on screen.
trait Visible {
    /// Render this object on the given canvas.
    fn draw(&self, canvas: &mut Canvas);

    /// Return true if clicking at (x, y) should
    /// select this object.
    fn hit_test(&self, x: i32, y: i32) -> bool;
}
```

要实现trait,请使用语法`impl TraitName for Type`:

```Rust
impl Visible for Broom {
    fn draw(&self, canvas: &mut Canvas) {
        for y in self.y - self.height - 1 .. self.y {
            canvas.write_at(self.x, y, '|');
        }
        canvas.write_at(self.x, self.y, 'M');
    }

    fn hit_test(&self, x: i32, y: i32) -> bool {
        self.x == x
        && self.y - self.height - 1 <= y
        && y <= self.y
    }
}
```

请注意,此`impl`包含`Visible`trait的每个方法的实现,而不包含任何其他方法.trait`impl`中定义的所有内容实际上必须是trait的功能;如果我们想添加一个辅助方法来支持`Broom::draw()`,我们必须在一个单独的`impl`块中定义它:

```Rust
impl Broom {
    /// Helper function used by Broom::draw() below.
fn broomstick_range(&self) -> Range<i32> {
    self.y - self.height - 1 .. self.y
    }
}

impl Visible for Broom {
    fn draw(&self, canvas: &mut Canvas) {
        for y in self.broomstick_range() {
            ...
        }
        ...
    }
    ...
}
```

### 默认方法(Default Methods)

我们之前讨论的`Sink`writer类型可以用几行代码实现.首先,我们定义类型:

```Rust
/// A Writer that ignores whatever data you write to it.
pub struct Sink;
```

`Sink`是一个空结构,因为我们不需要在其中存储任何数据.接下来,我们为`Sink`提供`Write`trait的实现:

```Rust
use std::io::{Write, Result};

impl Write for Sink {
    fn write(&mut self, buf: &[u8]) -> Result<usize> {
        // Claim to have successfully written the whole buffer.
        Ok(buf.len())
    }

    fn flush(&mut self) -> Result<()> {
        Ok(())
    }
}
```

到目前为止,这非常像`Visible`trait.但是我们也看到`Write`trait有一个`write_all`方法:

```Rust
out.write_all(b"hello world\n")?;
```

为什么不定义此方法的情况下,Rust也允许我们`impl Write for Sink`?答案是标准库的`Write`trait定义包含`write_all`的 *默认实现(default implementation)* :

```Rust
trait Write {
    fn write(&mut self, buf: &[u8]) -> Result<usize>;
    fn flush(&mut self) -> Result<()>;

    fn write_all(&mut self, buf: &[u8]) -> Result<()> {
        let mut bytes_written = 0;
        while bytes_written < buf.len() {
            bytes_written += self.write(&buf[bytes_written..])?;
        }
        Ok(())
    }

    ...
}
```

`write`和`flush`方法是每个writer必须实现的基本方法.writer也可以实现`write_all`,但如果没有,将使用上面显示的默认实现.

你自己的triat可以包括使用相同语法的默认实现.

标准库中最常用的默认方法是`Iterator`trait,它有一个必需的方法(`.next()`)和几十个默认方法.第15章解释了原因.

### Traits和其他人的类型(Traits and Other People's Types)

Rust允许你在任何类型上实现任何trait,只要在当前包中引入trait或类型即可.

这意味着只要你想要将方法添加到任何类型,你就可以使用trait来执行此操作:

```Rust
trait IsEmoji {
    fn is_emoji(&self) -> bool;
}

/// Implement IsEmoji for the built-in character type.
impl IsEmoji for char {
    fn is_emoji(&self) -> bool {
        ...
    }
}

assert_eq!('$'.is_emoji(), false);
```

与任何其他trait方法一样,只有当`IsEmoji`在作用域内时,这个新的`is_emoji`方法才可见.

此特定trait的唯一目的就是向现有类型`char`添加方法.这被称为 *扩展trait(extension trait)* .当然,也可以通过编写`impl IsEmoji for str {...}`来将此trait添加到类型中,依此类推.

你甚至可以使用泛型`impl`块一次性向一系列类型添加扩展trait.以下扩展trait为所有Rust writer添加了一个方法:

```Rust
use std::io::{self, Write};

/// Trait for values to which you can send HTML.
trait WriteHtml {
    fn write_html(&mut self, &HtmlDocument) -> io::Result<()>;
}

/// You can write HTML to any std::io writer.
impl <W: Write> WriteHtml for W {
    fn write_html(&mut self, html: &HtmlDocument) -> io::Result<()> {
        ...
    }
}
```

`impl <W：Write> WriteHtml for W`这行意味着"对于实现`Write`的每个类型`W`,这里是对于`W`的`WriteHtml`的实现(for every type W that implements Write, here's an implementation of WriteHtml for W)."

`serde`库提供了一个很好的例子,说明在标准类型上实现用户定义的trait是多么有用.`serde`是一个序列化库.也就是说,你可以使用它将Rust数据结构写入磁盘并在以后重新加载它们.该库定义了一个trait,`Serialize`,它为库支持的每种数据类型实现.所以在`serde`源代码中,有为`bool`,`i8`,`i16`,`i32`,数组和元组类型等等(包括所有标准数据结构,如`Vec`和`HashMap`),实现`Serialize`的代码.

所有这一切的结果是`serde`为所有这些类型添加了`.serialize()`方法.它可以像这样使用:

```Rust
use serde::Serialize;
use serde_json;

pub fn save_configuration(config: &HashMap<String, String>)
    -> std::io::Result<()>
{

    // Create a JSON serializer to write the data to a file.
    let writer = File::create(config_filename())?;
    let mut serializer = serde_json::Serializer::new(writer);

    // The serde `.serialize()` method does the rest.
    config.serialize(&mut serializer)?;
    Ok(())
}
```

我们之前说过,当你实现一个trait时,trait或类型必须是当前crate中新的.这称为 *一致性规则(coherence rule)* .它有助于Rust确保trait实现是独一无二的.你的代码不能`impl Write for u8`,因为`Write`和`u8`都是在标准库中定义的.如果Rust允许crate这样做,则在不同的crates中可能有多个`Write for u8`的实现,而Rust没有合理的方式来决定对于给定方法调用使用哪个实现.

(C++有一个类似的唯一性限制:单定义规则(One Definition Rule)在典型的C++方式中,它不是由编译器强制执行的,除非在最简单的情况下,如果你破坏它就会得到未定义行为.)

### Traits中的Self(Self in Traits)

trait可以使用关键字`Self`作为类型.例如,标准`Clone`trait,看起来像这样(略微简化):

```Rust
pub trait Clone {
    fn clone(&self) -> Self;
    ...
}
```

在这里使用`Self`作为返回类型意味着`x.clone()`的类型与`x`的类型相同,无论它是什么.如果`x`是`String`,则`x.clone()`的类型是`String`--不是`Clone`或任何其他可克隆类型.

同样,如果我们定义这个特征:

```Rust
pub trait Spliceable {
    fn splice(&self, other: &Self) -> Self;
}
```

有两个实现:

```Rust
impl Spliceable for CherryTree {
    fn splice(&self, other: &Self) -> Self {
        ...
    }
}

impl Spliceable for Mammoth {
    fn splice(&self, other: &Self) -> Self {
        ...
    }
}
```

然后在第一个`impl`中,`Self`只是`CherryTree`的别名,而在第二个中,它是`Mammoth`的别名.这意思是我们可以将两棵樱桃树或两只猛犸象拼接在一起,而不是我们可以创造一个猛犸-樱桃混合物.`self`的类型和`other`的类型必须匹配.

使用`Self`类型的trait与trait对象不兼容:

```Rust
// error: the trait `Spliceable` cannot be made into an object
fn splice_anything(left: &Spliceable, right: &Spliceable) {
    let combo = left.splice(right);
    ...
}
```

当我们深入研究trait的高级特性时,我们会一次又一次地看到这个原因.Rust拒绝此代码,因为它无法对调用`left.splice(right)`进行类型检查.trait对象的重点是直到运行时才知道该类型.Rust在编译时无法知道`left`和`right`是否会为同一类型(要求如此).

Trait对象实际上是针对最简单的trait,可以使用Java中的接口或C++中的抽象基类来实现的trait.traits的更高级的特性是有用的,但它们不能与trait对象共存,因为有了trait对象,你将丢失Rust用来对程序进行类型检查的类型信息.

现在,如果我们想要基因上不可能的拼接,我们可以设计出一种trait对象友好的trait:

```Rust
pub trait MegaSpliceable {
    fn splice(&self, other: &MegaSpliceable) -> Box<MegaSpliceable>;
}
```

此trait与trait对象兼容.对`.splice()`方法调用的类型检查没有问题,因为只要两个类型都是`MegaSpliceable`,参数`other`的类型不需要匹配`self`的类型.

### Subtraits(Subtraits)

我们可以声明trait是另一个trait的扩展:

```Rust
/// Someone in the game world, either the player or some other
/// pixie, gargoyle, squirrel, ogre, etc.
trait Creature: Visible {
    fn position(&self) -> (i32, i32);
    fn facing(&self) -> Direction;
    ...
}
```

短语`trait Creature: Visible`意味着所有生物都是可见的.实现`Creature`的每个类型也必须实现`Visible`trait:

```Rust
impl Visible for Broom {
    ...
}

impl Creature for Broom {
    ...
}
```

我们可以按任意顺序实现这两个trait,但是在没有实现`Visible`的情况下为类型实现`Creature`是错误的.

Subtraits就像Java或C#中的子接口.它们是一种描述trait的方式,该trait通过更多方法扩展现有trait.在此示例中,与`Creature`一起使用的所有代码也可以使用`Visible`trait中的方法.

### 静态方法(Static Methods)

在大多数面向对象语言中,接口不能包含静态方法或构造函数.但是,Rust trait可以包括静态方法和构造函数,这里这样:

```Rust
trait StringSet {
    /// Return a new empty set.
    fn new() -> Self;

    /// Return a set that contains all the strings in `strings`.
    fn from_slice(strings: &[&str]) -> Self;

    /// Find out if this set contains a particular `value`.
    fn contains(&self, string: &str) -> bool;

    /// Add a string to this set.
    fn add(&mut self, string: &str);
}
```

实现`StringSet`trait的每个类型都必须实现这四个关联的函数.前两个,`new()`和`from_slice()`不接受`self`参数.他们担任构造函数.

在非泛型代码中,可以使用`::`语法调用这些函数,就像任何其他静态方法一样:

```Rust
// Create sets of two hypothetical types that impl StringSet:
let set1 = SortedStringSet::new();
let set2 = HashedStringSet::new();
```

在泛型代码中,它是相同的,除了类型通常是类型变量,如在此处显示的对`S::new()`的调用:

```Rust
/// Return the set of words in `document` that aren't in `wordlist`.
fn unknown_words<S: StringSet>(document: &Vec<String>, wordlist: &S) -> S {
    let mut unknowns = S::new();
    for word in document {
        if !wordlist.contains(word) {
            unknowns.add(word);
        }
    }
    unknowns
}
```

与Java和C#接口一样,trait对象不支持静态方法.如果要使用`&StringSet`trait对象,则必须更改trait,将`where Self：Sized`的限制添加到每个静态方法:

```Rust
trait StringSet {
    fn new() -> Self
        where Self: Sized;

    fn from_slice(strings: &[&str]) -> Self
        where Self: Sized;

    fn contains(&self, string: &str) -> bool;

    fn add(&mut self, string: &str);
}
```

这个限制告诉Rust,trait对象可以不支持这个方法.然后允许使用`StringSet`trait对象;它们仍然不支持这两个静态方法,但你可以创建它们并使用它们来调用  `.contains()`和`.add()`.同样的技巧也适用于与trait对象不兼容的其他方法.(我们将放弃对其工作原理的相当繁琐的技术解释,但是第13章将介绍`Sized`trait.)

## 完全限定的方法调用(Fully Qualified Method Calls)

方法只是一种特殊的函数.这两个调用是等价的:

```Rust
"hello".to_string()

str::to_string("hello")
```

第二种形式看起来与静态方法调用完全相同.即使`to_string`方法接受`self`参数,这也可行.只需将`self`作为函数的第一个参数.

由于`to_string`是标准`ToString`trait的一个方法,因此你可以使用另外两种形式:

```Rust
ToString::to_string("hello")

<str as ToString>::to_string("hello")
```

所有这四个方法调用完全相同.通常,你只需编写`value.method()`.其他形式是 *限定的(qualified)* 方法调用.它们指定与方法关联的类型或trait.最后一种形式(带尖括号的)两个都指定: *完全限定的(fully qualified)* 方法调用.

当你写`"hello".to_string()`时,使用`.`运算符,你没有确切地说你正在调用哪个`to_string`方法.Rust有一个方法查找算法可以根据类型,deref强制多态等来推断出来.通过完全限定的调用,你可以确切地说出你所指的方法,这在一些奇怪的情况下可以提供帮助:

- 当两个方法具有相同的名称时.经典的假意的示例是`Outlaw`,它有两个来自两个不同trait的`.draw()`方法,一个用于在屏幕上绘制,另一个用于与法律交互:

```Rust
outlaw.draw();  // error: draw on screen or draw pistol?

Visible::draw(&outlaw);  // ok: draw on screen
HasPistol::draw(&outlaw);  // ok: corral
```

通常情况下,你最好只是重命名其中一种方法,但有时你不能.

- 当无法推断`self`参数的类型时:

```Rust
let zero = 0;  // type unspecified; could be `i8`, `u8`, ...

zero.abs();  // error: method `abs` not found
i64::abs(zero);  // ok
```

- 当使用函数本身作为函数值时:

```Rust
let words: Vec<String> =
    line.split_whitespace()  // iterator produces &str values
        .map(<str as ToString>::to_string)  // ok
        .collect();
```

这里完全限定的`<str as ToString>::to_string`只是一种命名我们想要传递给`.map()`的特定函数的方法.

- 在宏中调用trait方法时.我们将在第20章解释.

完全限定语法也适用于静态方法.在上一节中,我们编写了`S::new()`来在泛型函数中创建一个新集合.我们也可以编写`StringSet::new()`或`<S as StringSet>::new()`.

## 定义类型之间关系的traits(Traits That Define Relationships Between Types)

到目前为止,我们所看到的每个trait都是独立的:trait是类型可以实现的方法的集合.trait也可用于有多种类型必须协同工作的情况.他们可以描述类型之间的关系.

- `std::iter::Iterator`trait将每个迭代器类型与其生成的值的类型相关联.

- `std::ops::Mul`trait关联可以相乘的类型.在表达式`a * b`中,值`a`和`b`可以是相同类型,也可以是不同类型.

- `rand`crate包括随机数生成器的trait(`rand::Rng`)和可以随机生成的类型的特征(`rand::Rand`).trait本身确定了这些类型如何协同工作.

你不需要每天都创建这样的traits,但是你会在整个标准库和第三方crates中遇到它们.在本节中,我们将展示每个示例是如何实现的,并根据需要选择相关的Rust语言特性.这里的关键技能是能够阅读traits和方法签名,并弄清楚他们对所涉及的类型的看法.

### 关联类型(或迭代器如何工作)(Associated Types (or How Iterators Work))

我们将从迭代器开始.到目前为止,每种面向对象的语言都对迭代器具有某种内置支持,迭代器代表遍历某些值序列的对象.

Rust有一个标准的`Iterator`trait,定义如下:

```Rust
pub trait Iterator {
    type Item;

    fn next(&mut self) -> Option<Self::Item>;
    ...
}
```

这个trait的第一个特性,`type Item;`,是一种 *关联类型(associated type)* .实现`Iterator`的每种类型都必须指定它生成的项目类型.

第二个特性,`next()`方法,在其返回值中使用关联类型.`next()`返回一个`Option<Self::Item>`:`Some(item)`,序列中的下一个值,或者没有更多值可访问时的`None`.该类型写为`Self::Item`,而不仅仅是普通的`Item`,因为`Item`是每种迭代器的一个特性,而不是一个独立的类型.与往常一样,`self`和`Self`类型在使用它们的字段,方法等的任何地方显式地显示在代码中.

以下是为类型实现`Iterator`的样子:

```Rust
// (code from the std::env standard library module)
impl Iterator for Args {
    type Item = String;

    fn next(&mut self) -> Option<String> {
        ...
    }
    ...
}
```

`std::env::Args`是标准库函数`std::env::args()`返回的迭代器类型,我们在第2章中使用它来访问命令行参数.它产生`String`值,因此`impl`声明为`type Item = String;`.

泛型代码可以使用关联类型:

```Rust
/// Loop over an iterator, storing the values in a new vector.
fn collect_into_vector<I: Iterator>(iter: I) -> Vec<I::Item> {
    let mut results = Vec::new();
    for value in iter {
        results.push(value);
    }
    results
}
```

在这个函数的主体内部,Rust为我们推断了`value`的类型,这很好;但是我们必须拼出`collect_into_vector`的返回类型,而`Item`关联类型是唯一的方法.(`Vec<I>`将完全错误:我们将声称返回迭代器的向量!)

前面的示例不是你自己要编写的代码,因为在阅读第15章之后,你将知道迭代器已经有一个标准方法来执行此操作:`iter.collect()`.因此,在继续之前,让我们再看一个例子.

```Rust
/// Print out all the values produced by an iterator
fn dump<I>(iter: I)
    where I: Iterator
{
    for (index, value) in iter.enumerate() {
        println!("{}: {:?}", index, value);   // error
    }
}
```

这几乎可行.只有一个问题:`value`可能不是可打印的类型.

```Rust
error[E0277]: the trait bound `<I as std::iter::Iterator>::Item:
              std::fmt::Debug` is not satisfied
  --> traits_dump.rs:10:37
   |
10 | println!("{}: {:?}", index, value);   // error
   |                             ^^^^^ the trait `std::fmt::Debug`
   |                                   is not implemented for
   |                                   `<I as std::iter::Iterator>::Item`
   |
   = help: consider adding a
           `where <I as std::iter::Iterator>::Item: std::fmt::Debug` bound
   = note: required by `std::fmt::Debug::fmt`
```

Rust使用语法`<I as std::iter::Iterator>::Item`,这是表示`I::Item`的一种很长的,最明确的方式,因此错误消息稍微有些模糊.这是有效的Rust语法,但你很少需要以这种方式编写类型.

错误消息的要点是,要编译这个泛型函数,我们必须确保`I::Item`实现`Debug`trait,这是用`{:?}`格式化值的特征.我们可以通过在`I::Item`上放置一个限制来做到这一点:

```Rust
use std::fmt::Debug;

fn dump<I>(iter: I)
    where I: Iterator, I::Item: Debug
{
    ...
}
```

或者,我们可以写,"`I`必须是`String`值的迭代器":

```Rust
fn dump<I>(iter: I)
    where I: Iterator<Item=String>
{
    ...
}
```

`Iterator<Item=String>`本身就是一个trait.如果你将`Iterator`视为所有迭代器类型的集合,那么`Iterator<Item=String>`是`Iterator`的一个子集:生成`String`的迭代器类型的集合.可以在任何可以使用trait名称的地方使用此语法,包括trait对象类型:

```Rust
fn dump(iter: &mut Iterator<Item=String>) {
    for (index, s) in iter.enumerate() {
        println!("{}: {:?}", index, s);
    }
}
```

具有关联类型的traits(如`Iterator`)与trait方法兼容,但仅在拼写出所有关联类型时才会兼容,如此处所示.否则,`s`的可能是任何类型,并且Rust再也无法对此代码进行类型检查.

我们已经展示了很多涉及迭代器的例子.很难不这样做; 到目前为止,它们是关联类型最突出的用途.但是,只要trait需要涵盖的不仅仅是方法,关联类型通常是有用的.

- 在线程池库中,表示工作单元的`Task`trait可以具有关联的`Output`类型.

- 表示搜索字符串的方式的`Pattern`trait可以具有关联的`Match`类型,表示通过将模式与字符串匹配而收集的所有信息.

```Rust
trait Pattern {
    type Match;

    fn search(&self, string: &str) -> Option<Self::Match>;
}

/// You can search a string for a particular character.
impl Pattern for char {
    /// A "match" is just the location where the
    /// character was found.
    type Match = usize;

    fn search(&self, string: &str) -> Option<usize> {
        ...
    }
}
```

如果您熟悉正则表达式,很容易看出`impl Pattern for RegExp`将具有怎样更精细的`Match`类型,可能是一个包含匹配的开始和长度,括号组匹配的位置等等的结构.

- 用于处理关系数据库的库可能具有`Database Connection`trait,其中关联类型表示事务,游标,预处理语句等等.

关联类型非常适用于每个实现具有 *一种(one)* 特定相关类型的情况:每个`Task`类型产生特定类型的`Output;`每个`Pattern`类型会查找特定类型的`Match`.但是,正如我们将看到的,类型之间的某些关系不是这样的.

### 泛型Traits(或运算符重载如何工作)(Generic Traits (or How Operator Overloading Works))

Rust中的乘法使用此trait:

```Rust
/// std::ops::Mul, the trait for types that support `*`.
pub trait Mul<RHS> {
    /// The resulting type after applying the `*` operator
    type Output;

    /// The method for the `*` operator
    fn mul(self, rhs: RHS) -> Self::Output;
}
```

`Mul`是一个泛型trait.类型参数,`RHS`是 *右侧(right hand side)* 的缩写.

这里的类型参数意思和它在结构或函数上的含义相同:`Mul`是一个泛型trait,它的实例`Mul<f64>`,`Mul<String>`,`Mul<Size>`等都是不同的特征,就像`min::<i32>`和`min::<String>`是不同的函数,`Vec<i32>`和`Vec<String>`是不同的类型.

单一类型--比如`WindowSize`--可以实现`Mul<f64>`和`Mul<i32>`等等.然后,你可以将`WindowSize`乘以许多其他类型.每个实现都有自己的关联`Output`类型.

上面显示的trait缺少一个小细节.真正的`Mul`trait看起来像这样:

```Rust
pub trait Mul<RHS=Self> {
    ...
}
```

语法`RHS=Self`表示`RHS`默认为`Self`.如果我写`impl Mul for Complex`,而没有指定`Mul`的类型参数,则意味着`impl Mul<Complex> for Complex`.在一个限制中,如果我写`where T: Mul`,那就意味着`where T: Mul<T>`.

在Rust中,表达式`lhs * rhs`是`Mul::mul(lhs, rhs)`的简写.因此,在Rust中重载`*`运算符就像实现`Mul`trait一样简单.我们将在下一章中展示示例.

### 伙伴Traits(或rand::random()如何工作)(Buddy Traits(or How rand::random() Works))

还有一种使用traits来表达类型之间的关系的方式.这种方式可能是最简单的,因为你不必学习任何新的语言特性来理解它:我们称之为 *伙伴trait(buddy traits)* 的只是旨在协同工作的trait.

在`rand`crate(一个用于生成随机数的流行crate)里面有一个很好的例子.`rand`的主要功能是`random()`函数,它返回一个随机值:

```Rust
use rand::random;
let x = random();
```

如果Rust无法推断随机值的类型(通常是这种情况),则必须指定它:

```Rust
let x = random::<f64>();   // a number, 0.0 <= x < 1.0
let b = random::<bool>();  // true or false
```

对于许多程序,只需要这个泛型函数即可.但是`rand`crate还提供了几种不同的,但可互操作的随机数生成器.库中的所有随机数生成器都实现了一个共同的trait:

```Rust
/// A random number generator.
pub trait Rng {
    fn next_u32(&mut self) -> u32;
    ...
}
```

`Rng`只是一个可以根据需要吐出整数的值.`rand`库提供了一些不同的实现,包括`XorShiftRng`(快速伪随机数生成器)和`OsRng`(慢得多,但真正不可预测,用于加密).

伙伴trait叫`Rand`:

```Rust
/// A type that can be randomly generated using an `Rng`.
pub trait Rand: Sized {
    fn rand<R: Rng>(rng: &mut R) -> Self;
}
```

像`f64`和`bool`这样的类型实现了这个trait.将任意随机数生成器传递给它们的`::rand()`方法,并返回一个随机值:

```Rust
let x = f64::rand(rng);
let b = bool::rand(rng);
```

实际上,`random()`只不过是一个薄薄的包装,它将全局分配的`Rng`传递给这个`rand`方法.实现它的一种方法是这样的:

```Rust
pub fn random<T: Rand>() -> T {
    T::rand(&mut global_rng())
}
```

当你看到使用其他trait作为限制的trait时(`Rand::rand()`使用`Rng`的方式),你知道这两个特征是混合匹配( mix-and-match):任何`Rng`都可以生成每个`Rand`类型的值.由于所涉及的方法是泛型的,因此Rust为你的程序实际使用的`Rng`和`Rand`的每个组合生成优化的机器码.

这两个trait也有助于分离关注点.无论你是为你的`Monster`类型实现`Rand`还是实现一个非常快速但不那么随机的`Rng`,你都不必为这两段代码的协同工作做任何特别的事情,如图11-3所示.

*图11-3. 伙伴trait说明.左边列出的Rng类型是rand crate提供的真正的随机数生成器.*

标准库对计算哈希码的支持提供了另一个伙伴trait的例子.实现`Hash`的类型是可哈希的,因此它们可以用作哈希表键.实现`Hasher`的类型是哈希(散列)算法.这两个链接的方式与`Rand`和`Rng`相同:`Hash`有一个泛型方法`Hash::hash()`,它接受任何类型的`Hasher`作为参数.

另一个例子是`serde`库的`Serialize`trait,你可以在第247页的"Traits和其他人的类型(Traits and Other People's Types)"中看到它.它有一个我们没有谈到的伙伴trait:`Serializer`trait,它代表输出格式.`serde`支持可插拔的序列化格式.有JSON,YAML,称为CBOR的二进制格式,等等的`Serializer`实现.由于两个trait之间的密切关系,每种格式都自动支持每种可序列化类型.

在最后三节中,我们展示了traits可以描述类型之间关系的三种方式.所有这些也可以被视为避免虚方法开销和向下转换的方式,因为它们允许Rust在编译时知道更多具体类型.

## 逆向工程限制(Reverse-Engineering Bounds)

当没有单一的trait可以满足你所需的一切时,编写泛型代码可能是一个真正的困难.假设我们已经编写了这个非泛型函数来进行一些计算:

```Rust
fn dot(v1: &[i64], v2: &[i64]) -> i64 {
    let mut total = 0;
    for i in 0 .. v1.len() {
        total = total + v1[i] * v2[i];
    }
    total
}
```

现在我们想要对浮点值使用相同的代码.我们可能会试着这样做:

```Rust
fn dot<N>(v1: &[N], v2: &[N]) -> N {
    let mut total: N = 0;
    for i in 0 .. v1.len() {
        total = total + v1[i] * v2[i];
    }
    total
}
```

没有那么幸运:Rust抱怨使用`+`和`*`的使用以及`0`的类型.我们可以要求`N`是一个支持`+`和`*`的类型,使用`Add`和`Mul`trait.但是,我们对`0`的使用需要改变,因为`0`在Rust中总是一个整数;相应的浮点值为`0.0`.幸运的是,对于具有默认值的类型,存在标准的`Default`trait.对于数字类型,默认值始终为0.

```Rust
use std::ops::{Add, Mul};

fn dot<N: Add + Mul + Default>(v1: &[N], v2: &[N]) -> N {
    let mut total = N::default();
    for i in 0 .. v1.len() {
        total = total + v1[i] * v2[i];
    }
    total
}
```

这更接近,但仍然不太有效:

```Rust
error[E0308]: mismatched types
  --> traits_generic_dot_2.rs:11:25
   |
11 | total = total + v1[i] * v2[i];
   |                 ^^^^^^^^^^^^^ expected type parameter, found associated type
   |
   = note: expected type `N`
              found type `<N as std::ops::Mul>::Output`
```

我们的新代码假设将`N`类型的两个值相乘会产生另一个`N`类型的值.情况不一定如此.你可以重载乘法运算符以返回你想要的任何类型.我们需要以某种方式告诉Rust这个泛型函数仅适用于具有正常乘法风格的类型,其中相乘`N * N`返回`N`.我们通过将`Mul`替换为`Mul<Output=N>`来实现这一点,并对`Add`相同的操作:

```Rust
fn dot<N: Add<Output=N> + Mul<Output=N> + Default>(v1: &[N], v2: &[N]) -> N
{
    ...
}
```

此时,限制开始堆积,使代码难以阅读.让我们将限制移动到`where`子句中:

```Rust
fn dot<N>(v1: &[N], v2: &[N]) -> N
    where N: Add<Output=N> + Mul<Output=N> + Default
{
    ...
}
```

很好.但Rust还是抱怨这行代码:

```Rust
error[E0508]: cannot move out of type `[N]`, a non-copy array
 --> traits_generic_dot_3.rs:7:25
  |
7 |         total = total + v1[i] * v2[i];
  |                         ^^^^^ cannot move out of here
```

这可能是一个真正的难题,即使现在我们熟悉术语.是的,将值`v1[i]`移出切片是违法的.但数字是可复制的.所以有什么问题?

答案是 *Rust不知道(Rust doesn't know)* `v1[i]`是一个数字.事实上,它不是--类型`N`可以是满足我们给出的限制的任何类型.如果我们还希望`N`是可复制的类型,我们必须这样说:

```Rust
where N: Add<Output=N> + Mul<Output=N> + Default + Copy
```

有了这个,代码编译并运行.最终代码如下所示:

```Rust
use std::ops::{Add, Mul};

fn dot<N>(v1: &[N], v2: &[N]) -> N
    where N: Add<Output=N> + Mul<Output=N> + Default + Copy
{
    let mut total = N::default();
    for i in 0 .. v1.len() {
        total = total + v1[i] * v2[i];
    }
    total
}

#[test]
fn test_dot() {
    assert_eq!(dot(&[1, 2, 3, 4], &[1, 1, 1, 1]), 10);
    assert_eq!(dot(&[53.0, 7.0], &[1.0, 5.0]), 88.0);
}
```

在Rust中偶尔会发生这种情况:有一段时间与编译器激烈争论,最后代码看起来相当不错,就好像编写起来很轻松,运行得很漂亮.

我们在这里做的是对`N`的限制进行逆向工程,使用编译器来指导和检查我们的工作.所以有点麻烦是因为标准库中没有一个包含我们想要使用的所有运算符和方法的`Number`trait.碰巧的是,有一个名为`num`的流行开源crate,它定义了这样的trait!如果我们知道,我们可以在我们的 *Cargo.toml* 中添加`num`然后编写:

```Rust
use num::Num;

fn dot<N: Num + Copy>(v1: &[N], v2: &[N]) -> N {
    let mut total = N::zero();
    for i in 0 .. v1.len() {
        total = total + v1[i] * v2[i];
    }
    total
}
```

就像在面向对象的编程中一样,正确的接口使一切都很好,在泛型编程中,正确的trait使一切变得美好.

不过,为什么要这么麻烦呢?为什么Rust的设计者没有让泛型变得更像c++模板(代码中隐式保留了约束),就像"鸭子类型(duck typing)"一样?

Rust的方法的一个优点是泛型代码的向前兼容性.你可以更改公有泛型函数或方法的实现,如果没有更改签名,则不会其破坏任何用户.

限制的另一个好处是,当你确实收到编译器错误时,至少编译器可以告诉你哪里有问题.涉及到模板的c++编译器错误消息可能比Rust的要长得多,它指向许多不同的代码行,因为编译器无法告诉谁应该为一个问题负责:模板--或者它的调用者,也可能是模板--或者 *那个(that)* 模板的调用者...

也许显式地写出限制最重要的优点就是它们存在于代码和文档中.你可以查看Rust中泛型函数的签名,并确切地了解它接受哪种参数.对于模板,不能这样说.完整记录像Boost这样的c++库中的参数类型的工作要比我们在这里所经历的工作 *更加(more)* 艰巨.Boost开发人员没有用于检查其工作的编译器.

## 结论(Conclusion)

特征是Rust中的主要组织特性之一,并且有充分的理由.设计一个程序或库没有比一个好的接口更好的了.

本章充斥着大量的语法,规则和解释.现在我们已经奠定了基础,我们可以开始讨论Rust代码中使用traits和泛型的许多方式.事实是,我们才刚刚开始触及表面.接下来的两章将介绍标准库提供的常见traits.即将到来的章节涵盖闭包,迭代器,输入/输出和并发.特征和泛型在所有这些主题中发挥着核心作用.

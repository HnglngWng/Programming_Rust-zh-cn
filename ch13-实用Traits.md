# 实用Traits(Utility Traits)

> 科学无非是在大自然的千变万化中发现统一的探索--更确切地说,是在我们丰富多彩的经验中发现统一的探索.用柯勒律治的话来说,诗歌,绘画和艺术也是在寻求多样性中的统一.
> --Jacob Bronowski

原文

> Science is nothing else than the search to discover unity in the wild variety of nature--or,more exactly, in the variety of our experience.Poetry, painting, the arts are the same search,in Coleridge's phrase, for unity in variety.
> --Jacob Bronowski

除了运算符重载(我们在前一章中已经介绍过)之外,还有其他几个内置特性可以让你挂钩(hook into)到Rust语言和标准库的部分内容:

- 你可以使用`Drop`trait在值超出作用域时清理值,类似C++中的析构函数.

- 智能指针类型(如`Box<T>`和`Rc<T>`)可以实现`Deref`trait,使指针反映包装的值的方法.

- 通过实现`From<T>`和`Into<T>`trait,你可以告诉Rust如何将值从一种类型转换为另一种类型.

本章是Rust标准库中的一些有用traits,我们将介绍表13-1中显示的每个traits.

还有其他重要的标准库traits.我们将在第15章介绍Iterator和IntoIterator.第16章介绍了用于计算哈希码的`Hash`trait.第19章介绍了标记线程安全类型的一对traits(`Send`和`Sync`).

*表13-1. 实用traits总结.*

|Trait|描述|
|:--|:--|
|`Drop`|析构函数.每当删除值时Rust会自动运行的清理代码.|
|`Sized`|标记trait,用于在编译时已知的固定大小的类型的,而不是动态大小的类型(如切片).|
|`Clone`|支持克隆值的类型.|
|`Copy`|标记trait,用于只需对包含该值的内存进行逐字节复制即可克隆的类型.|
|`Deref`和`DerefMut`|用于智能指针类型的trait.|
|`Default`|具有合理"默认值"的类型.|
|`AsRef`和`AsMut`|转换trait,用于从另一种类型借用一种类型的引用|
|`Borrow`和`BorrowMut`|转换trait,类似`AsRef`和`AsMut`,但另外保证一致性哈希,排序和相等性|
|`From`和`Into`|转换trait,用于转换一种类型的值为另一种类型|
|`ToOwned`|转换trait,用于将引用转换为拥有值的|

## Drop(Drop)

当一个值的所有者消失时,我们说Rust会 *删除(drops)* 该值.删除值需要释放值拥有的任何其他值,堆存储和系统资源.删除是在各种情况下发生的:当变量超出作用域时;当表达式的值被`;`运算符丢弃时;当你截断一个向量时,从它的末尾删除元素;等等.

在大多数情况下,Rust会自动为你处理删除值.例如,假设你定义以下类型:

```Rust
struct Appellation {
    name: String,
    nicknames: Vec<String>
}
```

`Appellation`拥有字符串内容的堆存储和向量的元素的缓冲区.每当一个`Appellation`被删除时,Rust都会负责清理所有这些内容,而无需你进行任何进一步的编码.但是,如果需要,可以通过实现`std::ops::Drop`trait来自定义Rust如何删除类型的值:

```Rust
trait Drop {
    fn drop(&mut self);
}
```

Drop的实现类似于C++中的析构函数,或者其他语言中的终结器(finalizer).当一个值被删除时,如果它实现了`std::ops::Drop`,Rust会调用它的`drop`方法,然后继续删除它的字段或元素拥有的任何值,就像通常那样.这种隐式的`drop`调用是调用该方法的唯一方式;如果你尝试自己显式调用它,Rust会将其标记为错误.

因为Rust在删除其字段或元素之前调用`Drop::drop`,所以方法接收的值始终仍然是完全初始化的.对于`Appellation`类型的`Drop`实现可以充分利用其字段:

```Rust
impl Drop for Appellation {
    fn drop(&mut self) {
        print!("Dropping {}", self.name);
        if !self.nicknames.is_empty() {
            print!(" (AKA {})", self.nicknames.join(", "));
        }
        println!("");
    }
}
```

鉴于该实现,我们可以编写以下内容:

```Rust
{
    let mut a = Appellation { name: "Zeus".to_string(),
                            nicknames: vec!["cloud collector".to_string(),
                                            "king of the gods".to_string()] };
    println!("before assignment");
    a = Appellation { name: "Hera".to_string(), nicknames: vec![] };
    println!("at end of block");
}
```

当我们将第二个`Appellation`赋值给`a`时,第一个将被删除,当我们离开`a`的作用域时,第二个将被删除.此代码打印以下内容:

```Shell
before assignment
Dropping Zeus (AKA cloud collector, king of the gods)
at end of block
Dropping Hera
```

由于我们的`Appellation`的`std::ops::Drop`实现除了打印消息之外什么都不做,那么它的内存是如何清理的呢?`Vec`类型实现`Drop`,删除它的每个元素,然后释放它们占用的堆分配缓冲区.`String`在内部使用`Vec<u8>`来保存其文本,因此`String`本身不需要实现`Drop`;它让它的`Vec`负责释放字符.同样的原则扩展到`Appellation`值;当一个字符串被删除时,`Vec`的`Drop`实现实际上负责释放每个字符串的内容,最后释放持有向量元素的缓冲区.至于保存`Appellation`值本身的内存,它也有某一所有者,可能是一个局部变量或某一数据结构,负责释放它.

如果变量的值被移动到其他地方,变量超出作用域时是未初始化的,那么Rust将不会尝试删除该变量:它中没有值可以删除.

即使变量可能会或可能没有将其值移走(取决于控制流程)时,这个原则仍然存在.在这种情况下,Rust使用一个不可见的标志跟踪变量的状态,该标志指示是否需要删除变量的值:

```Rust
let p;
{
    let q = Appellation { name: "Cardamine hirsuta".to_string(),
                        nicknames: vec!["shotweed".to_string(),
                                        "bittercress".to_string()] };
    if complicated_condition() {
        p = q;
    }
}
println!("Sproing! What was that?");
```

根据`complex_condition`是返回`true`还是`false`,`p`或`q`将最终拥有`Appellation`,而另一个未初始化.它所在的位置决定了它在`println!`之前或之后删除,因为`q`在`println!`之前超出了作用域,`p`之后.虽然值可能会从一个地方移动到另一个地方,但Rust只会删除一次.

除非你定义一个拥有Rust还不知道的资源的类型,否则你通常不需要实现`std::ops::Drop`.例如,在Unix系统上,Rust的标准库在内部使用以下类型来表示操作系统文件描述符:

```Rust
struct FileDesc {
    fd: c_int,
}
```

`FileDesc`的`fd`字段只是程序完成后应该关闭的文件描述符的编号;`c_int`是`i32`的别名.标准库为`FileDesc`实现了`Drop`,如下所示:

```Rust
impl Drop for FileDesc {
    fn drop(&mut self) {
        let _ = unsafe { libc::close(self.fd) };
    }
}
```

这里,`libc::close`是C库的`close`函数的Rust名称.Rust代码只能在`unsafe`块中调用C函数,因此库在这里使用一个.

如果类型实现`Drop`,则无法实现`Copy`trait.如果类型是`Copy`,则意味着简单的逐字节复制足以生成值的独立副本.但是在同一数据上多次调用相同的`drop`方法通常是错误的.

标准前置包括一个删除值的函数,`drop`,但它的定义却一点也不神奇:

```Rust
fn drop<T>(_x: T) { }
```

换句话说,它通过值接收其参数,从调用者那里获得所有权--然后什么也不做.当`_x`超出作用域时,Rust会删除它的值,就像其他变量一样.

## Sized(Sized)

*有大小的(sized)* 类型是其值在内存中都具有相同大小的类型.Rust中的几乎所有类型都是有大小的(sized):每个`u64`占用8个字节,每个`(f32, f32, f32)`元组12个.甚至枚举也是有大小的:无论实际是哪种变体,枚举总是占据足够的空间来容纳其最大的变体.虽然`Vec<T>`拥有堆分配的缓冲区,其大小可以变化,但Vec值本身是指向缓冲区,其容量和长度的指针,因此`Vec<T>`是一个有大小的类型.

但是,Rust还有一些 *无大小的类型(unsized types)* ,其值的大小不同.例如,字符串切片类型`str`(注意,没有`&`)是无大小的.字符串字面量`"diminutive"`和`"big"`是对`str`切片的引用,占用10个和3个字节.两者都显示在图13-1中.像`[T]`这样的数组切片类型(同样没有`&`)也是无大小的:像`&[u8]`这样的共享引用可以指向任何大小的`[u8]`切片.因为`str`和`[T]`类型表示不同大小的值的集,所以它们是无大小的类型.

*图13-1. 对无大小的值的引用.*

Rust中另一种常见的无大小的类型是trait对象的引用.正如我们在第238页的"Trait对象"中所述,trait对象是指向实现给定trait的某个值的指针.例如,类型`&std::io::Write`和`Box<std::io::Write>`是指向实现`Write`trait的某个值的指针.引用的对象可能是文件,网络套接字,或你自己的已实现`Write`的某种类型.由于实现`Write`的类型的集是开放式的,因此`Write`将被视为无大小的类型:其值具有各种大小.

Rust无法在变量中存储无大小的值或将它们作为参数传递.你只能通过像`&str`或`Box<Write>`(它们本身就是有大小的)这样的指针来处理它们.如图13-1所示,指向无大小的值的指针始终是 *胖指针(fat pointer)* ,两个字宽:指向切片的指针也包含切片的长度,而trait对象也携带指向方法实现的虚表的指针.

trait对象和指向切片的指针非常对称.在这两种情况下,类型都缺少使用它所需的信息:你不能在不知道其长度的情况下索引`[u8]`,也不能在`Box<Write>`上调用方法而不知道`Write`的实现是否合适它指向的特定值.在这两种情况下,胖指针都填充了类型中缺少的信息,带有长度或虚表指针.省略的静态信息被动态信息替换.

所有的有大小的类型都实现了`std::marker::Sized`trait,它没有方法或关联类型.Rust会自动为它适用的所有类型实现它;你不能自己实现它.`Sized`的唯一用途是作为类型变量的限制:类似`T：Sized`的限制要求`T`是一个在编译时已知大小的类型.这种traits称为 *标记traits(marker traits)* ,因为Rust语言本身使用它们将某些类型标记为具有感兴趣的特征.

由于无大小的类型非常有限,因此大多数泛型类型变量应限制为`Sized`类型.实际上,这是必要的,以致于它是Rust中的隐式默认值:如果你编写`struct S<T> { ... }`,Rust会理解你的意思是`struct S<T: Sized> { ... }`.如果你不想以这种方式约束`T`,则必须显式地选择退出,编写`struct S<T: ?Sized>  { ... }`.`?Sized`语法特定于这种情况,意思是"不一定是`Sized`."例如,如果你编写`struct S<T: ?Sized> { b: Box<T> }`,然后Rust将允许你编写`S<str>`和`S<Write>`,其中box成为一个胖指针,`S<i32>`和`S<String>`也一样,其中box是一个普通的指针.

尽管有这些限制,但无大小的类型使Rust的类型系统更加顺畅.阅读标准库文档,偶尔会遇到类型变量的`?Sized`限制;这几乎总是意味着只指向给定的类型,并允许相关的代码使用切片和trait对象以及普通值.当一个类型变量具有`?Sized`限制时,人们常说它是 *大小有问题的(questionably sized)* :它可能是`Sized`,也可能不是.

除了切片和trait对象之外,还有一种无大小的类型.结构类型的最后一个字段(但只是它的最后一个)可以无大小的,并且这样的结构本身是无大小的.例如,`Rc<T>`引用计数指针在内部实现为指向私有类型`RcBox<T>`的指针,其将引用计数存储在`T`旁边.这是`RcBox`的简化定义:

```Rust
struct RcBox<T: ?Sized> {
    ref_count: usize,
    value: T,
}
```

`value`字段是`Rc<T>`计数引用的`T`;`Rc<T>`解引用指向该字段的指针.`ref_count`字段保存引用计数.

你可以将`RcBox`用于有大小的类型,如`RcBox<String>`;结果是一个有大小的结构类型.或者你可以将它与无大小的类型一起使用,如`RcBox<std::fmt::Display>`(其中`Display`是可以通过`println!`和类似宏格式化的类型的trait);`RcBox<Display>`是无大小的结构类型.

你无法直接构建`RcBox<Display>`值.相反,你必须首先创建一个普通的,有大小的`RcBox`,其`value`类型实现`Display`,如`RcBox<String>`.Rust然后允许你将引用`&RcBox<String>`转换为胖引用`&RcBox<Display>`:

```Rust
let boxed_lunch: RcBox<String> = RcBox {
    ref_count: 1,
    value: "lunch".to_string()
};

use std::fmt::Display;
let boxed_displayable: &RcBox<Display> = &boxed_lunch;
```

将值传递给函数时,此转换会隐式发生,因此你可以将`&RcBox<String>`传递给期望`&RcBox<Display>`的函数:

```Rust
fn display(boxed: &RcBox<Display>) {
    println!("For your enjoyment: {}", &boxed.value);
}

display(&boxed_lunch);
```

这会产生以下输出:

```Shell
For your enjoyment: lunch
```

## Clone(Clone)

`std::clone::Clone`trait适用于可以复制自身的类型.`Clone`定义如下:

```Rust
trait Clone: Sized {
    fn clone(&self) -> Self;
    fn clone_from(&mut self, source: &Self) {
        *self = source.clone()
    }
}
```

`clone`方法应构建`self`的独立副本并返回它.由于此方法的返回类型为`Self`,并且函数可能不返回无大小的值,因此`Clone`trait本身扩展了`Sized`trait:这具有将`Self`类型的实现限制为`Sized`的效果.

克隆一个值通常需要分配它拥有的任何东西的副本,因此克隆在时间和内存方面都很昂贵.例如,克隆`Vec<String>`不仅复制向量,还复制其每个`String`元素.这就是为什么Rust不自动克隆值,而是要求你进行显式方法调用.引用计数的指针类型如`Rc<T>`和`Arc<T>`是例外:克隆其中一个只是增加引用计数并递给你一个新指针.

`clone_from`方法将`self`修改为`source`的副本. `clone_from`的默认定义只是克隆`source`,然后将其移动到`*self`.这总是有效,但对于某些类型,有一种更快的方式来获得相同的效果.例如,假设`s`和`t`是`String`.语句`s = t.clone();`必须克隆`t`,删除`s`的旧值,然后将克隆值移动到`s`;这是一个堆分配,一个堆释放.但是如果属于原始`s`的堆缓冲区有足够的容量来保存`t`的内容,则不需要分配或释放:你可以简单地将`t`的文本复制到`s`的缓冲区中,并调整长度.在泛型代码中,你应尽可能使用`clone_from`,以便在可用时允许此优化.

如果你的`Clone`实现只是将`clone`应用于你的类型的每个字段或元素,然后从这些克隆构造一个新值,并且`clone_from`的默认定义足够好,那么Rust将为你实现:只需将`#[derive(Clone)]`放在你的类型定义之上.

几乎所有标准库中的类型都有意义复制实现`Clone`.像`bool`和`i32`这样的原始类型可以.像`String`,`Vec<T>`和`HashMap`这样的容器类型也可以.有些类型没有意义复制,比如`std::sync::Mutex`;那些没有实现`Clone`.某些类型如`std::fs::File`可以复制,但如果操作系统没有必要的资源,则复制可能会失败;这些类型没有实现`Clone`,因为`clone`必须是绝对可靠的.相反,`std::fs::File`提供了一个`try_clone`方法,它返回一个`std::io::Result<File>`,它可以报告失败.

## Copy(Copy)

在第4章中,我们解释说,对于大多数类型,赋值移动值而不是复制它们.移动值使跟踪其拥有的资源变得更加简单.但是在第86页的"Copy类型:移动的例外"中,我们指出了一个例外;不拥有任何资源的简单类型可以是`Copy`类型,其赋值创建源的副本,而不是移动值并使源未初始化.

那时候,我们对`Copy`的确切定义很模糊,但是现在我们可以告诉你:如果一个类型实现了`std::marker::Copy`标记trait,则类型为`Copy`,这个trait的定义如下:

```Rust
trait Copy: Clone { }
```

当然很容易为你自己的类型实现:

```Rust
impl Copy for MyType { }
```

但是因为`Copy`是一个对语言具有特殊含义的标记trait,所以Rust允许一种类型只有在需要一个浅的逐字节复制时才能实现`Copy`.拥有任何其他资源的类型(如堆缓冲区或操作系统句柄)无法实现`Copy`.

任何实现`Drop`trait的类型都不能是`Copy`.Rust假定如果某个类型需要特殊的清理代码,那么它也必须要求特殊的复制代码,因此不能是`Copy`.

与`Clone`一样,你可以使用`#[derive(Copy)]`让Rust为你派生`Copy`.你会经常看到用`#[derive(Copy, Clone)]`同时派生两者.

在使类型`Copy`之前要仔细考虑.虽然这样做使得类型更容易使用,但它对其实现施加了很大的限制.隐式复制也可能很昂贵.我们在第86页的"Copy类型:移动的例外"中详细解释了这些因素.

## Deref和DerefMut(Deref and DerefMut)

你可以通过实现`std::ops::Deref`和`std::ops::DerefMut`trait来指定解引用运算符如`*`和`.`在你的类型上的作用是怎样的.像`Box<T>`和`Rc<T>`这样的指针类型实现了这些trait,因此它们可以像Rust的内置指针类型那样工作.例如,如果你有一个`Box<Complex>`值`b`,则`*b`表示`b`指向的`Complex`值,而`b.re`表示其实部(real component).如果上下文为引用对象赋值或借用可变引用,则Rust使用`DerefMut`("dereference mutably)trait;否则,只读访问就足够了,它使用`Deref`.

这些traits定义如下:

```Rust
trait Deref {
    type Target: ?Sized;
    fn deref(&self) -> &Self::Target;
}

trait DerefMut: Deref {
    fn deref_mut(&mut self) -> &mut Self::Target;
}
```

`deref`和`deref_mut`方法接受`&Self`引用并返回`&Self::Target`引用.`Target`应该是`Self`包含,拥有或引用的东西:对于`Box<Complex>`,`Target`类型是`Complex`.请注意,`DerefMut`扩展了`Deref`:如果你可以解引用并修改它,当然你也应该能够借用它的共享引用.由于这些方法返回与`&self`相同生命周期的引用,因此只要返回的引用存在,`self`就会一直被借用.

`Deref`和`DerefMut`trait也扮演着另一个角色.由于`deref`接受`&Self`引用并返回`&Self::Target`引用,因此Rust使用它自动将前一类型的引用转换为后者.换句话说,如果插入一个`deref`调用会阻止类型不匹配,那么Rust会为你插入.实现`DerefMut`可以为可变引用进行相应的转换.这些被称为 *解引用强制多态(deref coercions)* :一种类型被"强制"成为另一种类型.

虽然解引用强制多态不是你自己无法显式写出的东西,但它们很方便:

- 如果你有某一`Rc<String>`值`r`,并想要对它应用`String::find`,你可以简单地写`r.find('?')`,而不是`(*r).find('?')`:方法调用隐式借用`r`,并且`&Rc<String>`强制转换为`&String`,因为`Rc<T>`实现了`Deref<Target=T>`.

- 你可以在`String`值上使用`split_at`等方法,即使`split_at`是`str`切片类型的方法,因为`String`实现了`Deref<Target=str>`.不需要`String`来重新实现`str`的所有方法,因为你可以从`&String`强制一个`&str`.

- 如果你有一个字节向量`v`,并且你想将它传递给一个需要字节切片`&[u8]`的函数,你可以简单地传递`&v`作为参数,因为`Vec<T>`实现了`Deref<Target=[T]>`.

如有必要,Rust将连续应用几个解引用强制.例如,使用前面提到的强制,你可以将`split_at`直接应用于`Rc<String>`,因为`&Rc<String>`解引用为`&String`,它解引用为具有`split_at`方法的`&str`.

例如,假设你有以下类型:

```Rust
struct Selector<T> {
    /// Elements available in this `Selector`.
    elements: Vec<T>,

    /// The index of the "current" element in `elements`. A `Selector`
    /// behaves like a pointer to the current element.
    current: usize
}
```

要使`Selector`的行为与文档注释声明一致,你必须为类型实现`Deref`和`DerefMut`:

```Rust
use std::ops::{Deref, DerefMut};

impl<T> Dereffor Selector<T> {
    type Target = T;
    fn deref(&self) -> &T {
        &self.elements[self.current]
    }
}

impl<T> DerefMut for Selector<T> {
    fn deref_mut(&mut self) -> &mut T {
        &mut self.elements[self.current]
    }
}
```

鉴于这些实现,你可以这样使用`Selector`:

```Rust
let mut s = Selector { elements: vec!['x', 'y', 'z'],
                    current: 2 };

// Because `Selector` implements `Deref`, we can use the `*` operator to
// refer to its current element.
assert_eq!(*s, 'z');

// Assert that 'z' is alphabetic, using a method of `char` directly on a
// `Selector`, via deref coercion.
assert!(s.is_alphabetic());

// Change the 'z' to a 'w', by assigning to the `Selector`'s referent.
*s = 'w';

assert_eq!(s.elements, ['x', 'y', 'w']);
```

`Deref`和`DerefMut`trait是为实现智能指针类型而设计的,如`Box`,`Rc`和`Arc`,以及用于你经常通过引用使用的某些类型的拥有版本,就像`Vec<T>`和`String`作为`[T]`和`str`拥有版本.你不应该为一个类型实现`Deref`和`DerefMut`,只是为了让`Target`类型的方法自动出现在它上面,就像C++基类的方法在子类上可见一样.这并不总是像你期望的那样工作,并且当它出错时可能会令人困惑.

解引用强制带来了一个警告,可能会引起一些混淆:Rust用它们来解决类型冲突,而不是满足类型变量的限制.例如,以下代码可以正常工作:

```Rust
let s = Selector { elements: vec!["good", "bad", "ugly"],
                current: 2 };

fn show_it(thing: &str) { println!("{}",thing); }
show_it(&s);
```

在调用`show_it(&s)`中,Rust看到类型为`&Selector<&str>`的参数和类型为`&str`的参数,找到`Deref<Target=str>`实现,并根据需要将调用重写为`show_it(s.deref()`.

但是,如果将`show_it`更改为泛型函数,则Rust突然不再合作:

```Rust
use std::fmt::Display;
fn show_it_generic<T: Display>(thing: T) { println!("{}", thing); }
show_it_generic(&s);
```

Rust抱怨:

```Rust
error[E0277]: thetrait bound `Selector<&str>: Display` is not satisfied
    |
542 |         show_it_generic(&s);
    |         ^^^^^^^^^^^^^^^ trait `Selector<&str>: Display` not satisfied
    |
```

这可能令人困惑:如何使函数泛型引入错误?是的,`Selector<&str>`本身并没有实现`Display`,但它解引用了`&str`,这是肯定的.

由于你传递了类型为`&Selector<&str>`的参数,并且函数的参数类型为`&T`,类型变量`T`必须为`Selector<&str>`.然后,Rust检查是否满足限制`T: Display`:因为它不应用解引用强制来满足类型变量的限制,所以此检查失败.

要解决此问题,你可以使用`as`运算符说明强制:

```Rust
show_it_generic(&s as &str);
```

## Default(Default)

某些类型具有相当明显的默认值:默认向量或字符串为空,默认数字为零,默认`Option`为`None`,等等.像这样的类型可以实现`std::default::Default`trait:

```Rust
trait Default {
    fn default() -> Self;
}
```

`default`方法只是返回`Self`类型的新值.`String`的`Default`实现很简单:

```Rust
impl Default for String {
    fn default() -> String {
        String::new()
    }
}
```

所有Rust的集合类型--`Vec`,`HashMap`,`BinaryHeap`等都实现了`Default`,`default`方法返回一个空集合.当你需要构建值的集合,但是希望让调用者确定要构建的集合类型时,这很有用.例如,`Iterator`trait的`partition`方法将迭代器生成的值拆分为两个集合,使用闭包来决定每个值的位置:

```Rust
use std::collections::HashSet;
let squares = [4, 9, 16, 25, 36, 49, 64];
let (powers_of_two, impure): (HashSet<i32>, HashSet<i32>)
    = squares.iter().partition(|&n| n & (n-1) == 0);

assert_eq!(powers_of_two.len(), 3);
assert_eq!(impure.len(), 4);
```

闭包`|&n| n & (n-1) == 0`使用一些位处理(bit-fiddling)来识别2的幂数,而`partition`使用它来产生两个`HashSet`.但当然,`partition`并不是特定于`HashSet`的;你可以使用它来生成你喜欢的任何类型的集合(只要集合类型实现`Default`)来生成一个空集合(以及`Extend<T>`)以便将`T`添加到集合中.`String`实现了`Default`和`Extend<char>`,因此你可以编写:

```Rust
let (upper, lower): (String, String)
    = "Great Teacher Onizuka".chars().partition(|&c| c.is_uppercase());
assert_eq!(upper, "GTO");
assert_eq!(lower, "reat eacher nizuka");
```

`Default`的另一个常见用途是为表示大量参数的结构生成默认值,其中大多数参数通常不需要更改.例如,`glium`crate为强大而复杂的OpenGL图形库提供了Rust绑定.`glium::DrawParameters`结构包含22个字段,每个字段控制OpenGL应如何呈现一些图形的不同细节.`glium draw`函数需要`DrawParameters`结构作为参数.由于`DrawParameters`实现了`Default`,你可以创建一个传递给`draw`,只提到你想要更改的那些字段:

```Rust
let params = glium::DrawParameters {
    line_width: Some(0.02),
    point_size: Some(0.02),
    .. Default::default()
};

target.draw(..., &params).unwrap();
```

这会调用`Default::default()`创建一个`DrawParameters`值,用默认值初始化其所有字段,然后使用结构的`..`语法创建一个更改了`line_width`和`point_size`字段的新结构,准备将其传递给`target.draw`.

如果类型`T`实现`Default`,则标准库自动为`Rc<T>`,`Arc<T>`,`Box<T>`,`Cell<T>`,`RefCell<T>`,`Cow<T>`,`Mutex<T>`以及`RwLock<T>`实现`Default`.例如,类型`Rc<T>`的默认值是指向类型`T`的默认值的`Rc`指针.

如果元组类型的所有元素类型都实现`Default`,那么元组类型也会实现,默认为保存每个元素的默认值的元组.

Rust不会隐式地为结构类型实现`Default`,但是如果结构的所有字段都实现`Default`,则可以使用`#[derive(Default)]`自动为结构实现`Default`.

任何`Option<T>`的默认值为`None`.

## AsRef和AsMut(AsRef and AsMut)

当类型实现`AsRef<T>`时,这意味着你可以有效地从中借用`&T`.`AsMut`类似于可变引用.它们的定义如下:

```Rust
trait AsRef<T: ?Sized> {
    fn as_ref(&self) -> &T;
}

trait AsMut<T: ?Sized> {
    fn as_mut(&mut self) -> &mut T;
}
```

因此,例如,`Vec<T>`实现`AsRef<[T]>`,`String`实现`AsRef<str>`.你还可以将`String`的内容借用为字节数组,因此`String`也实现了`AsRef<[u8]>`.

`AsRef`通常用于使函数接受的参数类型更加灵活.例如,`std::fs::File::open`函数声明如下:

```Rust
fn open<P: AsRef<Path>>(path: P) -> Result<File>
```

`open`真正想要的是`&Path`,一个代表文件系统路径的类型.但是使用这个签名,`open`接受任何可以从中借用`&Path`的东西--即实现`AsRef<Path>`的任何东西.这些类型包括`String`和`str`,操作系统接口字符串类型`OsString`和`OsStr`,当然还有`PathBuf`和`Path`;有关完整列表,请参阅库文档.这允许你将字符串字面量传递给`open`:

```Rust
let dot_emacs = std::fs::File::open("/home/jimb/.emacs")?;
```

所有标准库的文件系统访问函数都以这种方式接受路径参数.对于调用者来说,效果类似于C++中的重载函数,尽管Rust采用不同的方式来确定哪些参数类型是可接受的.

但这不可能是整个故事.字符串字面量是`&str`,但实现`AsRef<Path>`的类型是`str`,没有`&`.正如我们在第289页的"Deref和DerefMut"中所解释的那样,Rust不会尝试使用解引用强制来满足类型变量限制,因此它们在这里也没有帮助.

幸运的是,标准库包括全部实现:

```Rust
impl<'a, T, U> AsRef<U> for &'a T
    where T: AsRef<U>,
        T: ?Sized, U: ?Sized
{
    fn as_ref(&self) -> &U {
        (*self).as_ref()
    }
}
```

换句话说,对于任何类型`T`和`U`,如果`T： AsRef<U>`,那么也`&T：AsRef<U>`:只需​​跟随引用并像以前一样继续.特别是,因为`str: AsRef<Path>`,所以`&str：AsRef<Path>`也是如此.从某种意义上说,这是一种在检查类型变量的`AsRef`限制时获得有限形式的解引用强制的方法.

您可以假设,如果类型实现`AsRef<T>`,那么它也应该实现`AsMut<T>`.但是,有些情况下这是不合适的.例如,我们已经提到`String`实现了`AsRef<[u8]>`;这是有意义的,因为每个`String`都有一个字节缓冲区,可以作为二进制数据访问.但是,`String`进一步保证这些字节是Unicode文本的格式良好的UTF-8编码;如果`String`实现了`AsMut<[u8]>`,那么调用者就可以将`String`的字节更改为他们想要的任何形式,并且你不能再相信`String`是格式良好的UTF-8.只有当修改给定的`T`不能违反类型的不变性时,类型实现`AsMut<T>`才有意义.

尽管`AsRef`和`AsMut`非常简单,但为引用转换提供标准的,泛型的trait可以避免更特定的转换trait的扩散.当你可以实现`AsRef<Foo>`时,你应该避免定义你自己的`AsFoo`trait.

## Borrow和BorrowMut(Borrow and BorrowMut)

`std::borrow::Borrow`trait类似于`AsRef`:如果一个类型实现`Borrow<T>`,那么它的`borrow`方法有效地从它借用一个`&T`.但是`Borrow`强加了更多的限制:一个类型应该只在`&T`哈希,和比较,与它借用的值的方式相同时实现`Borrow<T>`.(Rust没有强制执行此操作;它只是trait的文档意图.)这使得`Borrow`在处理哈希表和树中的键时很有用,或者在处理由于某些其他原因而被哈希或比较的值时.

从`String`借用时这种区别很重要,例如:`String`实现了`AsRef<&str>`,`AsRef<[u8]>`和`AsRef<Path>`,但这三种目标类型通常会有不同的哈希值.只有`&str`切片保证像等效的`String`一样哈希,因此`String`只实现`Borrow<str>`.

`Borrow`的定义与`AsRef`的定义相同;只有名字变了:

```Rust
trait Borrow<Borrowed: ?Sized> {
    fn borrow(&self) -> &Borrowed;
}
```

`Borrow`旨在通过泛型哈希表和其他相关集合类型来解决特定情况.例如,假设你有一个`std::collections::HashMap<String, i32>`,将字符串映射到数字.这个表的键是`String`;每个条目都拥有一个.查找此表中条目的方法的签名应该是什么?这是第一次尝试:

```Rust
impl HashMap<K, V> where K: Eq + Hash
{
    fn get(&self, key: K) -> Option<&V> { ... }
}
```

这是有道理的:要查找条目,你必须为表提供适当类型的键.但在这种情况下,`K`是`String`;这个签名会强制你通过值传递一个`String`来进行每次`get`调用,这显然很浪费.你实际只需要一个键的引用:

```Rust
impl HashMap<K, V> where K: Eq + Hash
{
    fn get(&self, key: &K) -> Option<&V> { ... }
}
```

这稍微好一点,但是现在你必须将键作为`&String`传递,所以如果你想查找一个常量字符串,你必须写:

```Rust
hashtable.get(&"twenty-two".to_string())
```

这很荒谬:它在堆上分配一个`String`缓冲区并将文本复制到其中,这样它就可以将它作为`&String`借用,传递给`get`,然后删除它.

它应该足以传递任何可以进行哈希并与我们的键类型进行比较的内容;例如,`&str`应该是完全足够的.所以这是最后的迭代,这是你在标准库中可以找到的:

```Rust
impl HashMap<K, V> where K: Eq + Hash
{
    fn get<Q: ?Sized>(&self, key: &Q) -> Option<&V>
        where K: Borrow<Q>,
              Q: Eq + Hash
    { ... }
}
```

换句话说,如果你可以借用条目的键为`&Q`,并且结果引用哈希和比较以键本身的方式,那么显然`&Q`应该是可接受的键类型.由于`String`实现了`Borrow<str>`和`Borrow<String>`,因此`get`的最终版本允许你根据需要传递`&String`或`&str`作为键.

`Vec<T>`和`[T: N]`实现`Borrow<[T]>`.每个类似字符串的类型都允许借用其相应的切片类型:`String`实现`Borrow<str>`,`PathBuf`实现`Borrow<Path>`,等等.并且所有标准库的相关集合类型都使用`Borrow`来决定哪些类型可以传递给它们的查找函数.

标准库包含一个通用的实现,这样每个类型`T`都可以从它自己借用:`T: Borrow<t>`.这确保了`&K`始终是用于查找`HashMap<K, V>`中的条目的可接受类型.

为了方便起见,每个`&mut T`类型也实现了`Borrow<T>`,像往常一样返回共享引用`&T`.这允许你将可变引用传递给集合查找函数,而不必重新借用共享引用,模拟Rust通常的隐式强制从可变引用到共享引用.

`BorrowMut`trait类似于`Borrow`,用于可变引用:

```Rust
trait BorrowMut<Borrowed: ?Sized>: Borrow<Borrowed> {
    fn borrow_mut(&mut self) -> &mut Borrowed;
}
```

对`Borrow`的描述也适用于`BorrowMut`.

## From和Into(From and Into)

`std::convert::From`和`std::convert::Into`traits表示转换,消费一种类型的值,返回另一种类型的值.`AsRef`和`AsMut`traits从另一个类型借用一种类型的引用,而`From`和`Into`获取其参数的所有权,转换它,然后将结果的所有权返回给调用者.

他们的定义非常对称:

```Rust
trait Into<T>: Sized {
    fn into(self) -> T;
}

trait From<T>: Sized {
    fn from(T) -> Self;
}
```

标准库自动实现从每种类型到自身的简单转换:每个类型`T`实现`From<T>`和`Into<T>`.

虽然这些traits只是提供了两种方法来做同样的事情,但它们适合不同的用途.

你通常使用`Into`来使你的函数在它们接受的参数中更加灵活.例如,如果你写:

```Rust
use std::net::Ipv4Addr;
fn ping<A>(address: A) -> std::io::Result<bool>
    where A: Into<Ipv4Addr>
{
    let ipv4_address = address.into();
    ...
}
```

然后`ping`不仅可以接受`Ipv4Addr`作为参数,还可以接受`u32`或`[u8; 4]`数组,因为这些类型都很方便地实现了`Into<Ipv4Addr>`.(将IPv4地址视为单个32位值或四个字节的数组有时很有用.)因为`ping`对`address`的唯一了解是它实现了`Into<Ipv4Addr>`,所以当你调用`into`时,不需要指定你想要哪种类型;只有一种类型可以工作,所以类型推断帮你解决.

与之前一节中的`AsRef`一样,效果非常类似于C++中的重载函数.通过之前的`ping`定义,我们可以进行以下任何调用:

```Rust
println!("{:?}", ping(Ipv4Addr::new(23, 21, 68, 141))); // pass an Ipv4Addr
println!("{:?}", ping([66, 146, 219, 98]));             // pass a [u8; 4]
println!("{:?}", ping(0xd076eb94_u32));                 // pass a u32
```

然而,`From`trait有着不同的作用.`from`方法用作泛型构造函数,用于从其他单个值生成类型的实例.例如,`Ipv4Addr`没有两个名为`from_array`和`from_u32`的方法,它只是实现了`From<[u8; 4]>`和`From<u32>`,允许我们写:

```Rust
let addr1 = Ipv4Addr::from([66, 146, 219, 98]);
let addr2 = Ipv4Addr::from(0xd076eb94_u32);
```

我们可以让类型推断决定应用哪种实现.

给定适当的`From`实现,标准库自动实现相应的`Into`trait.当你定义自己的类型时,如果它具有单参数构造函数,则应将它们编写为对于适当的类型的`From<T>`的实现;你将免费获得相应的`Into`实现.

由于`from`和`into`转换方法获取其参数的所有权,因此转换可以重用原始值的资源来构造转换后的值.例如,假设你这样写:

```Rust
let text = "Beautiful Soup".to_string();
let bytes: Vec<u8> = text.into();
```

用于`String`的`Into<Vec<u8>>`的实现只是获取`String`的堆缓冲区,将其重新定位(不改变)为返回的向量的元素缓冲区.转换无需分配或复制文本.这是移动实现高效实现的另一种情况.

这些转换也提供了一种很好的方法,可以将约束类型的值放宽到更灵活的范围,而不会削弱约束类型的保证.例如,`String`保证其内容始终时有效UTF-8;它的变异方法受到严格限制,以确保你所做的一切都不会引入错误的UTF-8.但是这个例子有效地将一个`String`"降级(demotes)"为一个普通字节块,你可以用它做任何你喜欢的事情:也许你要压缩它,或者将它与其他不是UTF-8的二进制数据结合起来.因为`into`通过值获取其参数,所以在转换后不再初始化`text`,这意味着我们可以自由访问前一个`String`的缓冲区而不会破坏任何现存的`String`.

但是,廉价的转换不是`Into`和`From`合同的一部分.虽然`AsRef`和`AsMut`转换预计会很便宜,但`From`和`Into`转换可能会分配,复制或以其他方式处理值的内容.例如,`String`实现了`From<&str>`,它将字符串切片复制到`String`的新的堆分配缓冲区中.`std::collections::BinaryHeap<T>`实现`From<Vec<T>>`,它根据算法的要求对元素进行比较和重新排序.

请注意,`From`和`Into`仅限于永不失败的转换.方法的类型签名不提供任何方式来指示给定的转换没有成功.要提供进入或退出类型的可出错转换,最好使用返回`Result`类型的函数或方法.

在将`From`和`Into`添加到标准库之前,Rust代码充满了临时转换trait和构造方法,每个特定于单个类型.`From`和`Into`整理了你可以遵循的约定,使你的类型更易于使用,因为你的用户已经很熟悉它们.

## ToOwned(ToOwned)

给定引用,生成其所指对象的拥有副本的通用方式是调用`clone`,假设该类型实现了`std::clone::Clone`.但是如果你想克隆一个`&str`或者`&[i32]`怎么办?你可能需要的是`String`或`Vec<i32>`,但`Clone`的定义不允许:根据定义,克隆`&T`必须始终返回`T`类型的值,但是`str`和`[u8]`是无大小的;它们甚至不是函数可以返回的类型.

`std::borrow::ToOwned`trait提供了一种稍微宽松的方式来将引用转换为拥有的值:

```Rust
trait ToOwned {
    type Owned: Borrow<Self>;
    fn to_owned(&self) -> Self::Owned;
}
```

与`clone`(它必须精确地返回`Self`)不同,`to_owned`可以返回任何你可以从中借用`&Self`的东西:`Owned`类型必须实现`Borrow<Self>`.你可以从`Vec<T>`借用一个`&[T]`,所以`[T]`可以实现`ToOwned<Owned=Vec<T>>`,只要`T`实现`Clone`,这样我们就可以将切片的元素复制到向量中.类似地,`str`实现`ToOwned<Owned=String>`,`Path`实现`ToOwned<Owned=PathBuf>`,等等.

## Borrow和ToOwned的使用:谦恭的Cow(Borrow and ToOwned at Work: The Humble Cow)

充分利用Rust需要思考所有权问题,比如函数是应该通过引用还是通过值接收参数.通常你可以采用一种方法或另一种方法,参数的类型反映了你的决定.但在某些情况下,在程序运行之前,你无法决定是借用还是拥有;`std::borrow::Cow`类型(用于"写时克隆(“clone on write)")提供了一种方式.

其定义如下：

```Rust
enum Cow<'a, B: ?Sized + 'a>
    where B: ToOwned
{
    Borrowed(&'a B),
    Owned(<B as ToOwned>::Owned),
}
```

`Cow<B>`要么借用对`B`的共享引用,要么拥有一个值,我们可以从中借用这样的引用.由于`Cow`实现了`Deref`,你可以调用它上面的方法,好像它是对`B`的共享引用:如果它是`Owned`,它借用对拥有值的共享引用;如果它是`Borrowed`,它只是递出它所持有的引用.

你还可以通过调用其`to_mut`方法获取对`Cow`的值的可变引用,该方法返回`&mut B`.如果`Cow`恰好是`Cow::Borrowed`,`to_mut`只是调用引用的`to_owned`方法来获取它自己的引用对象副本,将`Cow`更改为`Cow::Owned`,并借用对新的拥有值的可变引用.这是类型名称所指的"写时克隆(clone on write)"行为.

类似地,`Cow`有一个`in_owned`方法,在必要时提升对拥有值的引用,然后返回它,将所有权移动到调用者并在此过程中消耗`Cow`.

`Cow`的一个常见用途是返回静态分配的字符串常量或计算的字符串.例如,假设你需要将错误枚举转换为消息.大多数变体可以使用固定字符串进行处理,但其中一些变量还包含应包含在消息中的其他数据.你可以返回一个`Cow<'static, str>`:

```Rust
use std::path::PathBuf;
use std::borrow::Cow;
fn describe(error: &Error) -> Cow<'static, str> {
    match *error {
        Error::OutOfMemory => "out of memory".into(),
        Error::StackOverflow => "stack overflow".into(),
        Error::MachineOnFire => "machine on fire".into(),
        Error::Unfathomable => "machine bewildered".into(),
        Error::FileNotFound(ref path) => {
            format!("file not found: {}", path.display()).into()}
    }
}
```

此代码使用`Cow`的`Into`实现来构造值.这个`match`语句的大多数分支返回一个`Cow::Borrowed`,指的是静态分配的字符串.但是当我们得到`FileNotFound`变体时,我们使用`format!`构造包含给定文件名的消息.`match`语句的这一分支产生一个`Cow::Owned`值.

`describe`的调用者不需要更改值可以简单地将`Cow`视为`&str`:

```Rust
println!("Disaster has struck: {}", describe(&error));
```

需要拥有值的调用者可以轻松生成一个:

```Rust
let mut log: Vec<String> = Vec::new();
...
log.push(describe(&error).into_owned());
```

使用`Cow`可以帮助`describe`和其调用者将分配推迟到需要分配的时候.
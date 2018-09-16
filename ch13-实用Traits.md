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

*表13-1. 实用traits总结*

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

*图13-1. 对无大小的值的引用*

Rust中另一种常见的无大小的类型是trait对象的引用.正如我们在第238页的"Trait对象"中所述,特征对象是指向实现给定trait的某个值的指针.例如,类型`&std::io::Write`和`Box<std::io::Write>`是指向实现`Write`trait的某个值的指针.引用的对象可能是文件,网络套接字,或你自己的已实现`Write`的某种类型.由于实现`Write`的类型的集是开放式的,因此`Write`将被视为无大小的类型:其值具有各种大小.

Rust无法在变量中存储无大小的值或将它们作为参数传递.你只能通过像`&str`或`Box<Write>`(它们本身就是有大小的)这样的指针来处理它们.如图13-1所示,指向无大小的值的指针始终是 *胖指针(fat pointer)* ,两个字宽:指向切片的指针也包含切片的长度,而trait对象也携带指向方法实现的虚表的指针.

trait对象和指向切片的指针非常对称.在这两种情况下,类型都缺少使用它所需的信息:你不能在不知道其长度的情况下索引`[u8]`,也不能在`Box<Write>`上调用方法而不知道`Write`的实现是否合适它指向的特定值.在这两种情况下,胖指针都填充了类型中缺少的信息,带有长度或虚表指针.省略的静态信息被动态信息替换.

所有的有大小的类型都实现了`std::marker::Sized`trait,它没有方法或关联类型.Rust会自动为它适用的所有类型实现它;你不能自己实现它.`Sized`的唯一用途是作为类型变量的限制:类似`T：Sized`的限制要求`T`是一个在编译时已知大小的类型.这种traits称为 *标记traits(marker traits)* ,因为Rust语言本身使用它们将某些类型标记为具有感兴趣的特征.

由于无大小的类型非常有限,因此大多数泛型类型变量应限制为`Sized`类型.实际上,这是必要的,以致于它是Rust中的隐式默认值:如果你编写`struct S<T> { ... }`,Rust会理解你的意思是`struct S<T: Sized> { ... }`.如果你不想以这种方式约束`T`,则必须显式地选择退出,编写`struct S<T: ?Sized>  { ... }`.`?Sized`语法特定于这种情况,意思是"不一定是`Sized`."例如,如果你编写`struct S<T: ?Sized> { b: Box<T> }`,然后Rust将允许你编写`S<str>`和`S<Write>`,其中box成为一个胖指针,`S<i32>`和`S<String>`也一样,其中box是一个普通的指针.

尽管有这些限制,但无大小的类型使Rust的类型系统更加顺畅.阅读标准库文档,偶尔会遇到类型变量的`?Sized`限制;这几乎总是意味着只指向给定的类型,并允许相关的代码使用切片和trait对象以及普通值.当一个类型变量具有`?Sized`限制时,人们常说它是 *大小有问题的(questionably sized)* :它可能是`Sized`,也可能不是.

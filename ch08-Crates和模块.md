# Crates和模块(Crates and Modules)

> 这是Rust主题中的一点:系统程序员可以拥有美好的东西.
> --Robert O'Callahan,["Random Thoughts on Rust: Crates.io and IDEs"](http://robert.ocallahan.org/2016/08/random-thoughts-on-rust-cratesio-and.html)

原文

> This is one note in a Rust theme: systems programmers can have nice things.

假设你正在编写一个模拟蕨类植物生长的程序,从单个细胞的水平开始.你的程序就像蕨类植物一样,开始时非常简单,所有代码可能都在一个文件中--只是一个想法的孢子.随着它的增长,它将开始有内部结构.不同的部分会有不同的用途.它将扩展到多个文件.它可能涵盖整个目录树.随着时间的推移,它可能会成为整个软件生态系统的重要组成部分.

本章介绍帮助你组织程序的Rust特性:crates和模块.我们还将讨论随着项目的发展自然而然出现的各种主题,包括如何记录和测试Rust代码,如何消除不需要的编译器警告,如何使用Cargo管理项目依赖和版本控制,如何在crates.io上发布开源库,等等.

## Crates

Rust程序由crates组成.每个crate都是一个Rust项目:单个库或可执行文件的所有源代码,以及任何相关的测试,示例,工具,配置和其他东西.对于你的蕨类植物模拟器,你可以使用第三方库用于3D图形,生物信息学,并行计算等.这些库作为crates分发(见图8-1).

*图8-1. 一个crate和它的依赖*

了解crate是什么以及它们如何协同工作的最简单方法是使用带有`--verbose`标志的`cargo build`来构建具有某些依赖的现有项目.我们这样做,使用第35页的"并发的Mandelbrot程序"作为示例.结果如下所示:

```Shell
$ cd mandelbrot
$ cargo clean    # delete previously compiled code
$ cargo build --verbose
    Updating registry `https://github.com/rust-lang/crates.io-index`
 Downloading image v0.6.1
 Downloading crossbeam v0.2.9
 Downloading gif v0.7.0
 Downloading png v0.4.2

 ... (downloading and compiling many more crates)

   Compiling png v0.4.2
     Running `rustc .../png-0.4.2/src/lib.rs
          --crate-name png
          --crate-type lib
          --extern num=.../libnum-a2e6e61627ca7fe5.rlib
          --extern inflate=.../libinflate-331fc425bf167339.rlib
          --extern flate2=.../libflate2-857dff75f2932d8a.rlib
          ...`
   Compiling image v0.6.1
     Running `rustc .../image-0.6.1/./src/lib.rs
          --crate-name image
          --crate-type lib
          --extern png=.../libpng-16c24f58491a5853.rlib
          ...`
   Compiling mandelbrot v0.1.0 (file://.../mandelbrot)
     Running `rustc src/main.rs
          --crate-name mandelbrot
          --crate-type bin
          --extern crossbeam=.../libcrossbeam-ba292320058da7df.rlib
          --extern image=.../libimage-254ec48c8f0684f2.rlib
          ...`
$
```

为了便于阅读,我们重新格式化了`rustc`命令行,并删除了许多与我们的讨论无关的编译器选项,并用省略号(...)替换它们.

你可能还记得,当我们完成时,Mandelbrot程序的 *main.rs* 包含三个`extern crate`声明:

```Rust
extern crate num;
extern crate image;
extern crate crossbeam;
```

这几行只是告诉Rust,`num`,`image`和`crossbeam`是外部库,而不是Mandelbrot程序本身的一部分.

我们还在 *Cargo.toml* 文件中指定了我们想要的每个crate的版本:

```Toml
[dependencies]
num = "0.1.27"
image = "0.6.1"
crossbeam = "0.2.8"
```

这里的词 *dependencies* 意味着该项目使用的其他crate:我们依赖的代码.我们在[crates.io](https://crates.io/)上发现了这些crate,这是Rust社区开源crate的网站.例如,我们通过访问crates.io并搜索图像库(mage library)来了解`image`库.crates.io上的每个crate页面都提供了文档和源代码的链接,以及一行配置,如`image ="0.6.1"`,你可以复制并添加到你的 *Cargo.toml* .这里显示的版本号只是我们编写程序时这三个软件包的最新版本.

Cargo副本讲述了如何使用这些信息的故事.当我们运行`cargo build`时,Cargo首先从crates.io下载这些crate的指定版本的源代码.然后,它会读取这些`crate`的 *Cargo.toml* 文件,下载 *它们的(their)* 依赖,等等递归.例如,`image`crate的0.6.1版的源代码包含一个`Cargo.toml`文件,它包含以下内容:

```Toml
[dependencies]
byteorder = "0.4.0"
num = "0.1.27"
enum_primitive = "0.1.0"
glob = "0.2.10"
```

看到这一点,Cargo知道在它可以使用`image`之前,它也必须获取这些crate.稍后,我们将看到如何告诉Cargo从Git存储库或本地文件系统而不是crates.io获取源代码.

一旦获得所有源代码,Cargo就会编译所有的crate.它运行`rustc`(Rust编译器),对项目的依赖图中的每个crate运行一次.编译库时,Cargo使用`--crate-type lib`选项.这告诉`rustc`不要查找`main()`函数,而是生成一个包含已编译代码的 *.rlib* 文件,后面的`rustc`命令可以用作输入.在编译程序时,Cargo使用`--crate-type bin`,结果是目标平台的二进制可执行文件:例如,Windows上的 *mandelbrot.exe* .

使用每个`rustc`命令,Cargo传递`--extern`选项,给出crate将使用的每个库的文件名.这样,当`rustc`看到一行代码如`extern crate crossbeam;`时,它知道在磁盘上的哪里找到编译的包.Rust编译器需要访问这些 *.rlib* 文件,因为它们包含库的已编译代码.Rust会将该代码静态链接到最终的可执行文件中. *.rlib* 还包含类型信息,因此Rust可以检查我们在代码中使用的库功能是否实际存在于crate中,并且我们正确使用它们.它还包含crate的公有的内联函数,泛型和宏的副本,这些功能在Rust看到我们如何使用它们之前无法完全编译为机器码.

`cargo build`支持各种选项,其中大部分超出了本书的范围,但我们将在此提及:`cargo build --release`产生优化的构建.发布版本运行得更快,但编译需要更长时间,它们不检查整数溢出,它们跳过`debug_assert！()`断言,它们在恐慌时生成的堆栈跟踪通常不太可靠.

### 构建配置文件(Build Profiles)

你可以在你的 *Cargo.toml* 文件中放置几个配置设置,这些设置会影响`cargo`生成的`rustc`命令行.

|命令行|Cargo.toml使用的部分|
|:--|:--|
|`cargo build`|`[profile.debug]`|
|`cargo build --release`|`[profile.release]`|
|`cargo test`|`[profile.test]`|

默认设置通常很好,但我们发现的一个例外是你想要使用一个分析器--一个测量程序花费CPU时间的工具.要从分析器获取最佳数据,你需要优化(通常仅在发布版本中启用)和调试符号(通常仅在调试版本中启用).要同时启用它们,请将其添加到你的 *Cargo.toml* :

```Toml
[profile.release]
debug = true  # enable debug symbols in release builds
```

`debug`设置控制`rustc`的`-g`选项.使用此配置,当你键入`cargo build --release`时,你将获得带有调试符号的二进制文件.优化设置不受影响.

[Cargo文档](http://doc.crates.io/manifest.html)列出了你可以调整的许多其他设置.

## 模块(Modules)

*Modules* 是Rust的命名空间.它们是构成Rust程序或库的函数,类型,常量等的容器.虽然crate是关于项目之间的代码共享,但模块是关于项目 *内(within)* 的代码组织.它们看起来像这样:

```Rust
mod spores {
    use cells::Cell;

    /// A cell made by an adult fern. It disperses on the wind as part of
    /// the fern life cycle. A spore grows into a prothallus -- a whole
    /// separate organism, up to 5mm across -- which produces the zygote
    /// that grows into a new fern. (Plant sex is complicated.)
    pub struct Spore {
        ...
    }

    /// Simulate the production of a spore by meiosis.
    pub fn produce_spore(factory: &mut Sporangium) -> Spore {
        ...
    }

    /// Mix genes to prepare for meiosis (part of interphase).
    fn recombine(parent: &mut Cell) {
        ...
    }

    ...
}
```

模块是 *项(items)* 的集合,命名像本示例中的`Spore`结构和两个函数.`pub`关键字使项目成为公有(public)项,因此可以从模块外部访问它.任何未标记为`pub`的内容都是私有的(private).

```Rust
let s = spores::produce_spore(&mut factory);  // ok

spores::recombine(&mut cell);  // error: `recombine` is private
```

模块可以嵌套,看到一个只是子模块集合的模块是很常见的:

```Rust
mod plant_structures {
    pub mod roots {
        ...
    }
    pub mod stems {
        ...
    }
    pub mod leaves {
        ...
    }
}
```

通过这种方式,我们可以在一个源文件中编写一个包含大量代码和整个模块层次结构的整个程序.实际上,以这种方式工作是一种痛苦,所以有另一种选择.

### 单独文件中的模块(Modules in Separate Files)

模块也可以这样写:

```Rust
mod spores;
```

早些时候,我们将`spores`模块的主体包括在花括号中.在这里,我们告诉Rust编译器`spores`模块位于一个名为 *spores.rs* 的单独文件中:

```Rust
// spores.rs

/// A cell made by an adult fern...
pub struct Spore {
    ...
}

/// Simulate the production of a spore by meiosis.
pub fn produce_spore(factory: &mut Sporangium) -> Spore {
    ...
}

/// Mix genes to prepare for meiosis (part of interphase).
fn recombine(parent: &mut Cell) {
    ...
}
```

*spores.rs* 仅包含组成模块的项目.它不需要任何样板来声明它是一个模块.

代码的位置是这个`spores`模块和我们在上一节中显示的版本之间的 *唯一(only)* 区别.什么是公有的和什么是私有的规则都是完全相同的.并且Rust从不单独编译模块,即使它们位于单独的文件中:当你构建Rust crate时,你将重新编译其所有模块.

模块可以拥有自己的目录.当Rust看到`mod spores;`时,它检查 *spores.rs* 和 *spores/mod.rs* ;如果两个文件都不存在,或者两者都存在,那就是错误.对于此示例，我们使用了 *spores.rs* ,因为`spores`模块没有任何子模块.但请考虑我们之前写过的`plant_structures`模块.如果我们决定将该模块及其三个子模块拆分到它们自己的文件,生成的项目将如下所示:

```Rust
fern_sim/
├── Cargo.toml
└── src/
    ├── main.rs
    ├── spores.rs
    └── plant_structures/
        ├── mod.rs
        ├── leaves.rs
        ├── roots.rs
        └── stems.rs
```

在 *main.rs* 中,我们声明 *plant_structures* 模块:

```Rust
pub mod plant_structures;
```

这会导致Rust加载 *plant_structures/mod.rs* ,它声明了三个子模块:

```Rust
// in plant_structures/mod.rs
pub mod roots;
pub mod stems;
pub mod leaves;
```

这三个模块的内容存储在名为 *leaves.rs* , *roots.rs* 和 *stems.rs* 的单独文件中,这些文件位于 *plant_structures* 目录中的 *mod.rs* 旁边.

### 路径和导入(Paths and Imports)

`::`运算符用于访问模块的功能.项目中任何位置的代码都可以通过写出其 *绝对路径(absolute path)* 来引用任何标准库功能:

```Rust
if s1 > s2 {
    ::std::mem::swap(&mut s1, &mut s2);
}
```

这个函数名称`::std::mem::swap`是一个绝对路径,因为它以双冒号开头.路径`::std`指的是标准库的顶级模块.`::std::mem`是标准库中的子模块,`::std::mem ::swap`是该模块中的公有函数.

你可以用这种方式编写所有代码,每次想要一个圆或字典时拼写出`::std::f64::consts::PI`和`::std::collections::HashMap::new`,但它输入繁琐,难以阅读.另一种方法是将功能 *导入(import)* 到使用它们的模块中:

```Rust
use std:mem;

if s1 > s2 {
    mem::swap(&mut s1, &mut s2)
}
```

`use`声明使名称`mem`成为整个封闭块或模块中`::std::mem`的本地别名.`use`声明中的路径自动就是绝对路径,因此不需要前导`::`.

我们可以编写`use std::mem::swap;`导入`swap`函数本身而不是`mem`模块.但是,我们上面所做的通常被认为是最好的风格:导入类型,trait和模块(如`std::mem`),然后使用相对路径来访问函数,常量和其他成员.

可以一次导入多个名称:

```Rust
use std::collections::{HashMap, HashSet};  // import both

use std::io::prelude::*;  // import everything
```

这只是写出所有单个导入的简写:

```Rust
use std::collections::HashMap;
use std::collections::HashSet;

// all the public items in std::io::prelude:
use std::io::prelude::Read;
use std::io::prelude::Write;
use std::io::prelude::BufRead;
use std::io::prelude::Seek;
```

模块 *不会(not)* 自动从其父模块继承名称.例如,假设我们的 *proteins/mod.rs* 中有这个:

```Rust
// proteins/mod.rs
pub enum AminoAcid { ... }
pub mod synthesis;
```

那么 *synthesis.rs* 中的代码不会自动看到类型`AminoAcid`:

```Rust
// proteins/synthesis.rs
pub fn synthesize(seq: &[AminoAcid])  // error: can't find type `AminoAcid`
    ...
```

相反,每个模块都以空白板开始,必须导入它使用的名称:

```Rust
// proteins/synthesis.rs
use super::AminoAcid;  // explicitly import from parent

pub fn synthesize(seq: &[AminoAcid])  // ok
    ...
```

关键字`super`在导入中具有特殊含义:它是父模块的别名.同样,self是当前模块的别名.

```Rust
// in proteins/mod.rs

// import from a submodule
use self::synthesis::synthesize;

// import names from an enum,
// so we can write `Lys` for lysine, rather than `AminoAcid::Lys`
use self::AminoAcid::*;
```

虽然默认情况下导入中的路径被视为绝对路径,但是`self`和`super`允许覆盖它并从相对路径导入.

(当然,这里的`AminoAcid`示例与我们之前提到的仅导入类型,trait和模块的样式规则不同.如果我们的程序包含长氨基酸序列,这在Orwell的第六条规则中是合理的:"破坏任何这些规则比说任何直截了当的野蛮都要快(Break any of these rules sooner than say anything outright barbarous).")

子模块可以访问其父模块中的私有项,但是它们必须按名称导入每个项.`use super::*;`仅导入标记为`pub`的项目.

模块与文件不同,但模块与Unix文件系统的文件和目录之间存在自然的类比.`use`关键字创建别名,就像`ln`命令创建链接一样.路径,如文件名,以绝对和相对形式出现.`self`和`super`就像`.`和`..`特殊目录.而`extern crate`将另一个crate的根模块移植到你的项目中.这很像安装文件系统.

### 标准前置(The Standard Prelude)

我们刚才说,就导入名称而言,每个模块都以"空白板(blank slate)"开头.但石板并非 *完全(completely)* 空白.

首先,标准库`std`自动链接到每个项目.就像你的 *lib.rs* 或 *main.rs* 包含一个看不见的声明:

```Rust
extern crate std;
```

此外,一些特别方便的名称,如`Vec`和`Result`,都包含在 *标准前置(standard prelude)* 中并自动导入.Rust的行为就好像每个模块(包括根模块)都以以下导入开头:

```Rust
use std::prelude::v1::*;
```

标准前置包含几十种常用的trait和类型.它 *不(not)* 包含`std`.因此,如果你的模块引用`std`,则必须显式导入它,如下所示:

```Rust
use std;
```

通常,导入你正在使用的`std`的特定功能更有意义.

在第2章中,我们提到库有时会提供名为`prelude`的模块.但是`std::prelude::v1`是唯一自动导入的前置. 命名模块`prelude`只是一种约定,告诉用户它应该使用`*`导入.

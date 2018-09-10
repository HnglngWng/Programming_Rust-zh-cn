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

这里的词 *dependencies* 意味着该项目使用的其他crate:我们依赖的代码.我们在[crates.io](https://crates.io/)上发现了这些crate,这是Rust社区开源crate的网站.例如,我们通过访问crates.io并搜索图像库(image library)来了解`image`库.crates.io上的每个crate页面都提供了文档和源代码的链接,以及一行配置,如`image ="0.6.1"`,你可以复制并添加到你的 *Cargo.toml* .这里显示的版本号只是我们编写程序时这三个软件包的最新版本.

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

### 项目,Rust的构建块(Items,the Building Blocks of Rust)

模块由 *项目(items)* 组成.有几种项目,以下列表实际上是语言的主要功能列表:

*函数(Functions)*

我们已经见过很多了.

*类型(Types)*

使用`struct`,`enum`和`trait`关键字引入用户定义的类型.我们将在适当的时候给它们每个一章; 一个简单的结构看起来像这样:

```Rust
pub struct Fern {
    pub roots: RootSet,
    pub stems: StemSet
}
```

结构的字段,甚至是私有字段,都可以在声明结构的模块中访问.在模块外部,只能访问公有字段,

事实证明,通过模块而不是像Java或C++中通过类来强制执行访问控制对于软件设计来说是非常有用的.它减少了样板"getter"和"setter"方法,并且在很大程度上消除了像C++`friend`声明之类的需求.单个模块可以定义几个紧密协作的类型,例如`frond::LeafMap`和`frond::LeafMapIter`,根据需要访问彼此的私有字段,同时仍然从程序的其余部分隐藏这些实现细节.

*类型别名(Type aliases)*

正如我们所见,`type`关键字可以像C++中的`typedef`一样使用,为现有类型声明一个新名称:

```Rust
type Table = HashMap<String, Vec<String>>;
```

我们在这里声明的类型`Table`是这种特殊类型的`HashMap`的简写.

```Rust
fn show(table: &Table) {
    ...
}
```

`impl` *块*(`impl` *blocks*)

使用`impl`块将方法附加到类型:

```Rust
impl Cell {
    pub fn distance_from_origin(&self) -> f64 {
        f64::hypot(self.x, self.y)
    }
}
```

语法将在第9章中介绍.impl块不能标记为`pub`.而是将单个方法标记为`pub`,以使它们在当前模块外部可见.

私有方法(和私有结构字段类似)在声明它们的整个模块中都是可见的.

*常量(Constants)*

`const`关键字引入了一个常量.语法就像`let`一样,只是它可以标记为`pub`,并且类型是必需的.此外,`UPPERCASE_NAMES`是常量的常规形式:

```Rust
pub const ROOM_TEMPERATURE: f64 = 20.0;  // degrees Celsius
```

`static`关键字引入了一个静态项,几乎是一样的:

```Rust
pub static ROOM_TEMPERATURE: f64 = 20.0;  // degrees Celsius
```

常量有点像C++的`#define`:在每个使用的地方将值编译到代码中.静态是在程序开始运行之前设置的变量,持续到退出为止.在代码中使用常量来表示魔数(magic bunbers)和字符串.对大量数据使用静态,或者在需要借用对常量值的引用时.

没有`mut`常量.静态可以标记为`mut`,但正如第5章所讨论的,Rust无法强制执行有关`mut`静态独占访问的规则.因此,它们本质上是非线程安全的,安全代码根本不能使用它们:

```Rust
static mut PACKETS_SERVED: usize = 0;

println!("{} served", PACKETS_SERVED);  // error: use of mutable static
```

Rust阻止全局可变状态.有关备选方案的讨论,请参见第496页的"全局变量(Global Variable)".

*模块(Modules)*

我们已经谈过很多了.正如我们所见,模块可以包含子模块,子模块可以是公有的或私有的,就像任何其他命名项一样.

*导入(Imports)*

`use`和`extern crate`声明也是项目.即使它们只是别名,它们也可以是公有的:

```Rust
// in plant_structures/mod.rs
...
pub use self::leaves::Leaf;
pub use self::roots::Root;
```

这意味着`Leaf`和`Root`是`plant_structures`模块的公有项.它们仍然是`plant_structures::leaves::Leaf`和`plant_structures::roots::Root`的简单别名.

标准前置就是这样一系列的`pub`导入.

`extern` *块*(`extern` *blocks*)

这些声明了一些用其他语言(通常是C或C++)编写的函数集合,因此Rust代码可以调用它们.我们将在第21章介绍`etern`块.

Rust警告声明但从未使用过的项目:

```Rust
warning: function is never used: `is_square`
  --> src/crates_unused_items.rs:23:9
   |
23 | /         pub fn is_square(root: &Root) -> bool {
24 | |             root.cross_section_shape().is_square()
25 | |         }
   | |_________^
   |
```

这个警告可能令人费解,因为有两种截然不同的可能原因.也许这个函数目前真的是死代码.或者,也许你打算在其他crate中使用它.在这种情况下,你需要将其和 *所有封闭模块(all enclosing modules)* 标记为公有的.

## 将程序转换成库(Turning a Program into a Library)

当你的蕨类植物模拟器开始起飞时,你决定你需要的不仅仅是一个程序.假设你有一个运行模拟的命令行程序并将结果保存在文件中.现在,你想要编写其他程序来对保存的结果进行科学分析,实时显示正在生长的植物的3D渲染,渲染逼真的图片等等.所有这些程序都需要共享基本的蕨类模拟代码.你需要建一个库.

第一步是将现有项目分为两部分:一个是包含所有共享代码的库crate,另一个是可执行文件,其中包含现有命令行程序只需要的代码.

为了说明如何做到这一点,让我们使用一个非常简化的示例程序:

```Rust
struct Fern {
    size: f64,
    growth_rate: f64
}

impl Fern {
    /// Simulate a fern growing for one day.
    fn grow(&mut self) {
        self.size *= 1.0 + self.growth_rate;
    }
}

/// Run a fern simulation for some number of days.
fn run_simulation(fern: &mut Fern, days: usize) {
    for _ in 0 .. days {
        fern.grow();
    }
}

fn main() {
    let mut fern = Fern {
        size: 1.0,
        growth_rate: 0.001
    };
    run_simulation(&mut fern, 1000);
    println!("final fern size: {}", fern.size);
}
```

我们假设这个程序有一个简单的 *Cargo.toml* 文件:

```Toml
[package]
name = "fern_sim"
version = "0.1.0"
authors = ["You <you@example.com>"]
```

将此程序转换为库很容易.以下是步骤:

1. 将文件 *src/main.rs* 重命名为 *src/lib.rs* .
2. 将`pub`关键字添加到`src/lib.rs`中的项目,这些项目将成为我们库的公共功能.
3. 将`main`函数移动到某个临时文件.我们马上就会回来.

生成的 *src/lib.rs* 文件如下所示:

```Rust
pub struct Fern {
    pub size: f64,
    pub growth_rate: f64
}

impl Fern {
    /// Simulate a fern growing for one day.
    pub fn grow(&mut self) {
        self.size *= 1.0 + self.growth_rate;
    }
}

/// Run a fern simulation for some number of days.
pub fn run_simulation(fern: &mut Fern, days: usize) {
    for _ in 0 .. days {
        fern.grow();
    }
}
```

请注意,我们不需要在 *Cargo.toml* 中更改任何内容.这是因为我们的最小 *Cargo.toml* 文件将Cargo保留为默认行为.默认情况下,`cargo build`会查看源目录中的文件并确定要构建的内容.当它看到文件 *src/lib.rs* 时,它知道构建一个库.

*src/lib.rs* 中的代码构成了库的 *根模块(root module)* .使用我们的库的其他crate只能访问该根模块的公有项目.

## src/bin目录(The src/bin Directory)

让原始命令行`fern_sim`程序再次运行也很简单:Cargo对与程序库存在相同代码库的小程序有一些内置支持.

事实上,Cargo本身就是这样编写的.大部分代码都在Rust库中.我们在本书中使用的`cargo`命令行程序是一个薄的包装程序,可以向库中调用所有繁重的工作.库和命令行程序都[位于同一个源存储库中](https://github.com/rust-lang/cargo).

我们也可以将我们的程序和库放在同一个代码库中.将此代码放入名为 *src/bin/efern.rs* 的文件中:

```Rust
extern crate fern_sim;
use fern_sim::{Fern, run_simulation};

fn main() {
    let mut fern = Fern {
        size: 1.0,
        growth_rate: 0.001
    };
    run_simulation(&mut fern, 1000);
    println!("final fern size: {}", fern.size);
}
```

`main`函数是我们之前预留的.我们添加了一个`extern crate`声明,因为这个程序将使用`fern_sim`库crate,我们从库中导入`Fern`和`run_simulation`.

因为我们已将此文件放入 *src/bin*,所以Cargo将在下次运行`cargo build`时编译`fern_sim`库和此程序.我们可以使用`cargo run --bin efern`来运行`efern`程序.这就是它的样子,使用`--verbose`显示Cargo正在运行的命令:

```Shell
$ cargo build --verbose
   Compiling fern_sim v0.1.0 (file:///.../fern_sim)
     Running `rustc src/lib.rs --crate-name fern_sim --crate-type lib ...`
     Running `rustc src/bin/efern.rs --crate-name efern --crate-type bin ...`
$ cargo run --bin efern --verbose
       Fresh fern_sim v0.1.0 (file:///.../fern_sim)
     Running `target/debug/efern`
final fern size: 2.7169239322355985
```

我们仍然没有对 *Cargo.toml* 进行任何更改,因为Cargo的默认设置是查看源文件并解决问题.它会自动将 *src/bin* 中的 *.rs* 文件视为要构建的额外程序.

当然，既然`fern_sim`是一个库,我们还有另一种选择.我们可以将这个程序放在单独的项目中,放在一个完全独立的目录中,有它自己的 *Cargo.toml* ,将`fern_sim`列为依赖:

```Toml
[dependencies]
fern_sim = { path = "../fern_sim" }
```

也许这就是你将要做的其他蕨类植物模拟计划.*src/bin* 目录只适用于像`efern`这样的简单程序.

## 属性(Attributes)

Rust程序中的任何项都可以使用 *属性(attributes)* 进行修饰.属性是Rust用于向编译器编写各种指令和建议的全能语法.例如,假设你收到此警告:

```Rust
libgit2.rs: warning: type `git_revspec` should have a camel case name
    such as `GitRevspec`, #[warn(non_camel_case_types)] on by default
```

但是你选择这个名字是有原因的,你希望Rust能够关闭它.你可以通过在类型上添加`#[allow]`属性来禁用警告:

```Rust
#[allow(non_camel_case_types)]
pub struct git_revspec {
    ...
}
```

条件编译是使用属性编写的另一个功能,`#[cfg]`属性:

```Rust
// Only include this module in the project if we're building for Android.
#[cfg(target_os = "android")]
mod mobile;
```

`#[cfg]`的完整语法在[Rust Reference](https://doc.rust-lang.org/reference.html#conditional-compilation)中指定;这里列出了最常用的选项:

|`#[cfg]`选项|何时启用|
|:--|:--|
|`test`|启用测试(使用`cargo test`或`rustc --test`编译时.|
|`debug_assertions`|启用调试断言(通常在非优化的构建中).|
|`unix`|为Unix编译,包括macOS.|
|`windows`|为Windows编译.|
|`target_pointer_width = "64"`|针对64位平台.另一个可能的值是"32".|
|`target_arch = "x86_64"`|特别针对x86-64.其他值:`"x86"`,`"arm"`,`"aarch64"`,`"powerpc"`,`"powerpc"`,`"mips"`.|
|`target_os = "macos"`|为macOS编译.其他值:`"windos"`,`"ios"`,`"android"`,`"linux"`,`"openbsd"`,`"netbsd"`,`"dragonfly"`,`"bitrig"`.|
|`feature = "robots"`|启用名为`"robots"`的用户定义功能(使用`cargo build --feature robots or rustc --cfg feature='"robots"'`编译).功能在 [*Cargo.toml* 的`[features]`部分](http://doc.crates.io/manifest.html#the-features-section)中声明.|
|`not(A)`|A不满足.要提供函数的两种不同实现,请使用`#[cfg(X)]`标记一个,使用`#[cfg(not(X))]`标记另一个.|
|`all(A,B)`|A和B都满足(相当于`&&`).|
|`any(A,B)`|A或B满足(相当于`||`).|

```Rust
/// Adjust levels of ions etc. in two adjacent cells
/// due to osmosis between them.
#[inline]
fn do_osmosis(c1: &mut Cell, c2: &mut Cell) {
    ...
}
```

有一种情况是没有`#[inline]`就 *不会(won't)* 发生内联.当在另一个包中调用一个包中定义的函数或方法时,除非它是泛型的(它有类型参数)或者它显式地标记为`#[inline]`,否则Rust不会内联它.

否则,编译器会将`#[inline]`视为建议.Rust还支持更坚持的`#[inline(always)]`,请求在每个调用站点内联扩展函数,以及`#[inline(never)]`,以请求函数永远不会被内联.

某些属性(如`#[cfg]`和`#[allow]`)可以附加到整个模块并应用于其中的所有内容.其他的,如`#[test]`和`#[inline]`,必须附加到单个项目.正如你可能期望的那样,每个属性都是自定义的,并且具有自己的一组受支持的参数.Rust Reference详细记录了完整的[受支持属性集](https://doc.rust-lang.org/reference/attributes.html).

要将属性附加到整个包,请将其添加到 *main.rs* 或 *lib.rs* 文件的顶部,在任何项之前,然后编写`#!`而不是`#`,像这样:

```Rust
// libgit2_sys/lib.rs
#![allow(non_camel_case_types)]

pub struct
git_revspec {
    ...
}

pub struct git_error {
    ...
}
```

`#!`告诉Rust将一个属性附加到封闭项而不是接下来的内容:在这种情况下,`#![allow]`属性附加到整个`libgit2_sys`包,而不仅仅是`struct git_revspec`.

`#!`也可以在函数,结构体等内部使用,但它通常仅在文件的开头使用,以将属性附加到整个模块或crate.有些属性总是使用`#!`语法因为它们只能应用于整个crate.

例如,`#![feature]`属性用于打开Rust语言和库的 *不稳定(unstable)* 功能,这些功能是实验性的,因此可能会出现错误,或者将来可能会更改或删除.例如,正如我们写的那样,Rust实验支持128位整数类型`i128`和`u128`;但由于这些类型是实验性的,你只能通过以下方式使用它们:(1)安装Rust的Nightly版本和(2)显式声明你的crate使用它们:

```Rust
#![feature(i128_type)]

fn main() {
    // Do my math homework, Rust!
    println!("{}", 9204093811595833589_u128 * 19973810893143440503_u128);
}
```

随着时间的推移,Rust团队有时会 *稳定(stabilizes)* 实验性功能,因此它成为语言的标准部分.然后`#![feature]`属性变得多余,Rust会生成警告,建议你将其删除.

## 测试和文档(Tests and Documentation)

正如我们在第11页的"编写和运行单元测试(Writing and Running Unit Tests)"中所看到的,Rust中内置了一个简单的单元测试框架.测试是标有`#[test]`属性的普通函数.

```Rust
#[test]
fn math_works() {
    let x: i32 = 1;
    assert!(x.is_positive());
    assert_eq!(x + 1, 2);
}
```

`cargo test`运行项目中的所有测试.

```Shell
$ cargo test
   Compiling math_test v0.1.0 (file:///.../math_test)
     Running target/release/math_test-e31ed91ae51ebf22
running 1 test
test math_works ... ok
test result: ok. 1 passed; 0 failed; 0 ignored; 0 measured
```

(你还会看到一些关于"doc-tests"的输出,我们将在一分钟内得到它).

无论你的crate是可执行文件还是库,这都是一样的.你可以通过将参数传递给Cargo来运行特定测试:`cargo test math`运行所有在其名称中包含`math`的测试.

测试通常使用Rust标准库中的`assert!`和`assert_eq!`宏.如果`expr`为真,则`assert!(expr)`成功.否则,它会引起恐慌,导致测试失败.`assert_eq!(v1, v2)`就像`assert!(v1 == v2)`一样,如果断言失败,错误消息显示两个值.

你可以在普通代码中使用这些宏来检查不变量,但请注意`assert!`和`assert_eq!`即使在发布版本中也包含在内.使用`debug_assert!`和`debug_assert_eq!`去替代,编写仅在调试版本中检查的断言.

要测试错误情况,请将`#[should_panic]`属性添加到测试中:

```Rust
/// This test passes only if division by zero causes a panic,
/// as we claimed in the previous chapter.
#[test]
#[should_panic(expected="divide by zero")]
fn test_divide_by_zero_error() {
    1 / 0;  // should panic!
}
```

标有`#[test]`的函数是有条件编译的.当你运行`cargo test`时,Cargo会在你的测试和测试工具启用的情况下构建程序的副本.普通`cargo build`或`cargo build --release`跳过测试代码.这意味着你的单元测试可以与他们测试的代码一起使用,如果需要,可以访问内部实现细节,但是没有运行时成本.然而,它可能会导致一些警告.例如:

```Rust
fn roughly_equal(a: f64, b: f64) -> bool {
    (a - b).abs() < 1e-6
}

#[test]
fn trig_works() {
    use std::f64::consts::PI;
    assert!(roughly_equal(PI.sin(), 0.0));
}
```

在测试版本中,这很好.在非测试版本中,`approximate_equal`未使用,Rust会抱怨:

```Shell
$ cargo build
   Compiling math_test v0.1.0 (file:///.../math_test)
warning: function is never used: `roughly_equal`
 --> src/crates_unused_testing_function.rs:7:1
  |
7 | / fn roughly_equal(a: f64, b: f64) -> bool {
8 | |     (a - b).abs() < 1e-6
9 | | }
  | |_^
  |
   = note: #[warn(dead_code)] on by default
```

因此,当你的测试变得足够大以至于需要支持代码时,惯例是将它们放在`test`模块中并使用`#[cfg]`属性声明整个模块仅进行测试:

```Rust
#[cfg(test)]
// include this module only when testing
mod tests {
    fn roughly_equal(a: f64, b: f64) -> bool {
        (a - b).abs() < 1e-6
    }

    #[test] fn trig_works() {
        use std::f64::consts::PI;
        assert!(roughly_equal(PI.sin(), 0.0));
    }
}
```

Rust的测试工具使用多个线程一次运行多个测试,默认情况下,Rust代码的一个很好的附带好处是线程安全. (要禁用此功能,请运行单个测试,`cargo test` *`testname`* ;或将环境变量`RUST_TEST_THREADS`设置为1.)这意味着,从技术上讲,我们在第2章中展示的Mandelbrot程序不是该章中的第二个多线程程序,而是第三个!第11页的"编写和运行单元测试(Writing and Running Unit Tests)"中的`cargo test`是第一个.

### 集成测试(Integration Tests)

你的蕨类植物模拟器继续增长.你决定将所有主要功能放入可由多个可执行文件使用的库中.如果使用 *fern_sim.rlib* 作为外部包,那么使用最终用户的方式与库链接的测试会很不错.此外,你还有一些测试通过从二进制文件加载已保存的模拟开始,并且在 *src* 目录中有这些大型测试文件很不方便.集成测试有助于解决这两个问题.

集成测试是 *.rs* 文件,它们位于项目的 *src* 目录旁边的 *tests* 目录中.当你运行`cargo test`时,Cargo会将每个集成测试编译为单独的,独立的crate,与你的库和Rust测试工具链接.这是一个例子:

```Rust
// tests/unfurl.rs - Fiddleheads unfurl in sunlight

extern crate fern_sim;
use fern_sim::Terrarium;
use std::time::Duration;

#[test]
fn test_fiddlehead_unfurling() {
    let mut world = Terrarium::load("tests/unfurl_files/fiddlehead.tm");
    assert!(world.fern(0).is_furled());
    let one_hour = Duration::from_secs(60 * 60);
    world.apply_sunlight(one_hour);
    assert!(world.fern(0).is_fully_unfurled());
}
```

请注意,集成测试包括`extern crate`声明,因为它使用`fern_sim`作为库.集成测试的重点在于,他们可以从外部看到你的crate,就像用户一样.他们测试crate的公有API.

`cargo test`同时运行单元测试和集成测试.要仅在特定文件中运行集成测试--例如, *tests/unfurl.rs* --使用命令`cargo test --test unfurl`.

### 文档(Documentation)

命令`cargo doc`为你的库创建HTML文档:

```Shell
$ cargo doc --no-deps --open
 Documenting fern_sim v0.1.0 (file:///.../fern_sim)
```

`--no-deps`选项告诉Cargo仅为`fern_sim`本身生成文档,而不是为它所依赖的所有crate生成文档.

`--open`选项告诉Cargo之后在浏览器中打开文档.

你可以在图8-2中看到结果.Cargo将新文档文件保存在 *target/doc* 中.起始页面是 *target/doc/fern_sim/index.html* .

*图8-2. rustdoc生成的文档示例*

该文档从你的库的pub功能,以及你附加到它们的任何 *文档注释(doc comments)* 生成.我们已经在本章中看到了一些文档注释.他们看起来像注释:

```Rust
/// Simulate the production of a spore by meiosis.
pub fn produce_spore(factory: &mut Sporangium) -> Spore {
    ...
}
```

但是当Rust看到以三个斜杠开头的注释时,它会将它们视为`#[doc]`属性.Rust将前面的示例视作与此完全相同:

```Rust
#[doc = "Simulate the production of a spore by meiosis."]
pub fn produce_spore(factory: &mut Sporangium) -> Spore {
    ...
}
```

编译或测试库时,将忽略这些属性.生成文档时,公有功能上的doc注释包含在输出中.

同样,以`//!`开头的评论被视为`#![doc]`属性,并附加到封闭功能,通常是模块或crate.例如,你的 *fern_sim/src/lib.rs* 文件可能开始如下所示:

```Rust
//! Simulate the growth of ferns, from the level of
//! individual cells on up.
```

文档注释的内容被视为Markdown,简单HTML格式的简写表示法.星号用于`*斜体*`和`**粗体**`,空行被视为段落,等等.但是,你也可以依赖HTML;你的文档注释中的任何HTML标记都会逐字复制到文档中.

你可以使用`` `backticks` ``在运行文本的中间设置代码位.在输出中,这些片段将以固定宽度字体格式化.可以通过缩进四个空格来添加更大的代码示例.

```Rust
/// A block of code in a doc comment:
///
///     if everything().works() {
///         println!("ok");
///     }
```

你还可以使用Markdown防护代码块.这具有完全相同的效果.

```Rust
/// Another snippet, the same code, but written differently:
///
/// ```
/// if everything().works() {
///     println!("ok");
/// }
/// ```
```

无论你使用哪种格式,当你在文档注释中包含一段代码时,都会发生一件有趣的事情.Rust会自动将其转换为测试.

### 文档测试(Doc-Tests)

当你在Rust库crate中运行测试时,Rust会检查文档中显示的所有代码是否实际运行并正常运行.它通过获取文档注释中出现的每个代码块,将其编译为单独的可执行crate,将其链接到你的库并运行它来实现此目的.

这是文档测试的独立示例.通过`cargo new ranges`创建一个新项目,并将此代码放在 *range/src/lib.rs* 中:

```Rust
use std::ops::Range;
/// Return true if two ranges overlap.
///
///     assert_eq!(ranges::overlap(0..7, 3..10), true);
///     assert_eq!(ranges::overlap(1..5, 101..105), false);
///
/// If either range is empty, they don't count as overlapping.
///
///     assert_eq!(ranges::overlap(0..0, 0..10), false);
///
pub fn overlap(r1: Range<usize>, r2: Range<usize>) -> bool {
    r1.start < r1.end && r2.start < r2.end &&
        r1.start < r2.end && r2.start < r1.end
}
```

文档注释中的两个小代码块出现在`cargo doc`生成的文档中,如图8-3所示.

*图8-3. 展示一些文档测试的文档*

它们也成为两个独立的测试:

```Shell
$ cargo test
   Compiling ranges v0.1.0 (file:///.../ranges)
...
   Doc-tests ranges
running 2 tests
test overlap_0 ... ok
test overlap_1 ... ok
test result: ok. 2 passed; 0 failed; 0 ignored; 0 measured
```

如果你将`--verbose`标志传递给Cargo,你会发现它正在使用`rustdoc --test`运行这两个测试.Rustdoc将每个代码示例存储在一个单独的文件中,添加几行样板代码,以生成两个程序.这是第一个:

```Rust
extern crate ranges;
fn main() {
    assert_eq!(ranges::overlap(0..7, 3..10), true);
    assert_eq!(ranges::overlap(1..5, 101..105), false);
}
```

这是第二个:

```Rust
extern crate ranges;
fn main() {
    assert_eq!(ranges::overlap(0..0, 0..10), false);
}
```

如果这些程序编译并成功运行,则测试通过.

这两个代码示例包含断言,但这只是因为在这种情况下.断言会提供合适的文档.文档测试背后的想法不是将所有测试都放入注释中.相反,你编写了最好的文档,Rust确保文档中的代码示例实际编译和运行.

通常,最小的工作示例包括使代码编译所必需的一些细节,例如导入或设置代码,但仅仅在文档中显示的重要性不够.要隐藏代码示例的一行,请在该行的开头添加一个`#`后跟一个空格:

```Rust
/// Let the sun shine in and run the simulation for a given
/// amount of time.
///
///     # use fern_sim::Terrarium;
///     # use std::time::Duration;
///     # let mut tm = Terrarium::new();
///     tm.apply_sunlight(Duration::from_secs(60));
///
pub fn apply_sunlight(&mut self, time: Duration) {
    ...
}
```

有时在文档中显示完整的示例程序是有帮助的,包括`main`函数和`estern crate`声明.显然,如果这些代码片段出现在你的代码示例中,你也不希望Rustdoc自动添加它们.结果不会编译.因此,Rustdoc将包含确切字符串`fn main`的任何代码块视为完整程序,并且不向其添加任何内容.

可以针对特定代码块禁用测试.要告诉Rust编译你的示例,但没有实际运行它,请使用带有`no_run`注解的防护代码块:

```Rust
/// Upload all local terrariums to the online gallery.
///
/// ```no_run
/// let mut session = fern_sim::connect();
/// session.upload_all();
/// ```
pub fn upload_all(&mut self) {
    ...
}
```

如果甚至不希望编译代码,请使用`ignore`而不是`no_run`.如果代码块根本不是Rust代码,请使用该语言的名称,如`c++`或`sh`,或纯文本的`text`.`rustdoc`不知道数百种编程语言的名称;相反,它会将其无法识别的任何注解视为表示代码块不是Rust.这会禁用代码高亮以及文档测试.

## 指定依赖(Specifying Dependencies)

我们已经看到了一种告诉Cargo在哪里获取你的项目所依赖的crates的源代码的方法:按版本号.

```Toml
image = "0.6.1"
```

有几种方法可以指定依赖,还有一些相当微妙的事情你可能想要说明要使用哪些版本,因此值得花几页(篇幅).

首先,你可能希望使用未在crates.io上发布的依赖.一种方法是通过指定Git存储库URL和修订版本:

```Toml
image = { git = "https://github.com/Piston/image.git", rev = "528f19c" }
```

这个特殊的包是开源的,托管在GitHub上,但你可以轻松地指向企业网络上托管的私有Git存储库.如此处所示,你可以指定要使用的特定`rev`,`tag`或`branch`.(这些都是告诉Git要检出的源代码修订版的所有方法.)

另一种方法是指定包含crate源代码的目录:

```Toml
image = { path = "vendor/image" }
```

当你的团队有一个单一的版本控制存储库,其中包含多个crate的源代码或整个依赖关系图时,这很方便.每个crate可以使用相对路径指定其依赖关系.

对依赖关系进行这种级别的控制非常强大.如果你决定使用的任何开源crate并不完全符合你的喜好,你可以轻松地fork:只需点击GitHub上的Fork按钮并更改 *Cargo.toml* 文件中的一行即可.你的下一次`cargo build`将无缝地使用你的crate的fork而不是正式版本.

### 版本(Versions)

当你在 *Cargo.toml* 文件中写出像`image ="0.6.1"` 之类的内容时,Cargo会相当宽泛地解释它.它使用与版本0.6.1兼容的最新版本的`image`.

兼容性规则适用于[语义化版本](http://semver.org/).

- 以0.0开头的版本号是如此原始,以至于Cargo从不认为它与任何其他版本兼容.

- 以0.x开头的版本号,其中x非零,被认为与0.x系列中的其他点发行版本兼容.我们指定了`image`版本0.6.1,但Cargo将使用0.6.3(如果可用).(这不是语义化版本标准所说的关于0.x版本号的内容,但事实证明该规则太有用了.)

- 一旦项目达到1.0,只有新的主要版本会破坏兼容性.因此,如果你要求版本2.0.1,Cargo可能会使用2.17.99,而不是3.0.

默认情况下,版本号是灵活的,否则使用哪个版本的问题很快就会过度约束.假设一个库`libA`使用`num ="0.1.31"`,而另一个库`libB`使用`num ="0.1.29"`.如果版本号需要精确匹配,则没有项目可以将这两个库一起使用.允许Cargo使用任何兼容版本是一个更实用的默认值.

不过,在涉及到依赖关系和版本控制时,不同的项目有不同的需求.你可以使用运算符指定确切的版本或版本的范围:

|Cargo.toml行|含义|
|:--|:--|
|`image = "=0.10.0"`|仅使用精确版本0.10.0|
|`image = ">=1.0.5"`|使用1.0.5或 *任何(any)* 更高版本(甚至2.9,如果可用)|
|`image = ">1.0.5 <1.1.9"`|使用高于1.0.5但低于1.1.9的版本|
|`image = "<=2.7.10"`|使用最高2.7.10的任何版本|

你偶尔会看到的另一个版本规范是通配符`*`.这告诉Cargo任何版本都可以.除非某些其他 *Cargo.toml* 文件包含更具体的约束,否则Cargo将使用最新的可用版本.[*doc.crates.io* 上的Cargo文档](http://doc.crates.io/crates-io.html)更详细地介绍了版本规范.

请注意,兼容性规则意味着无法仅出于营销原因选择版本号.他们实际意味着什么.它们是crate维护者和用户之间的合同.如果你维护的crate是1.7版本,并且你决定删除某函数或进行任何其他不完全向后兼容的更改,则必须将版本号提升至2.0.如果你将其称为1.8,那么你将声称新版本与1.7兼容,但是你的用户可能会发现自己的构建已损坏.

### Cargo.lock(Cargo.lock)

*Cargo.toml* 中的版本号是故意灵活的,但我们不希望Cargo每次构建时都将我们升级到最新的库版本.想象一下,当`cargo build`突然升级到一个新版本的库时,正处于激烈的调试会话中.这可能是令人难以置信的破坏性.在调试过程中发生任何变化都是不好的.事实上,谈到库时,从来没有一个意外变化的好时机.

因此Cargo有一个内置机制来防止这种情况发生.第一次构建项目时,Cargo输出一个 *Cargo.lock* 文件,该文件记录了它使用的每个crate的确切版本.以后的版本将查阅此文件并继续使用相同的版本.只有当你通过手动提升 *Cargo.toml* 文件中的版本号或通过运行`cargo update`时,Cargo才能升级到更新的版本:

```Shell
$ cargo update
    Updating registry `https://github.com/rust-lang/crates.io-index`
    Updating libc v0.2.7 -> v0.2.11
    Updating png v0.4.2 -> v0.4.3
```

`cargo update`仅升级到与你在 *Cargo.toml* 中指定的内容兼容的最新版本.如果你指定了`image ="0.6.1"`,并且想要升级到版本`0.10.0`,则必须在 *Cargo.toml* 中更改它.下次构建时,Cargo将更新到新版本的`image`库并将新版本号存储在 *Cargo.lock* 中.

上面的示例显示Cargo更新了托管在[crates.io](https://crates.io/)上的两个crate.对于存储在Git中的依赖,会发生类似的事情.假设我们的 *Cargo.toml* 文件包含以下内容:

```Toml
image = { git = "https://github.com/Piston/image.git", branch = "master" }
```

如果看到我们有一个 *Cargo.lock* 文件,`cargo build`将不会从Git存储库中提取(pull)新的更改.相反,它读取 *Cargo.lock* 并使用与上次相同的修订版.但`cargo update`将从`master`中提取(pull),以便我们的下一次构建使用最新版本.

*Cargo.lock* 会自动为你生成,你通常不会手动编辑它.尽管如此,如果你的项目是可执行文件,则应将 *Cargo.lock* 提交(commit)到版本控制.这样,构建项目的每个人都将始终获得相同的版本. *Cargo.lock* 文件的历史记录将记录你的依赖更新.

如果你的项目是普通的Rust库,请不要操心地提交 *Cargo.lock* .你的库的下游用户将拥有 *Cargo.lock* 文件,其中包含整个依赖图的版本信息;他们会忽略你的库的 *Cargo.lock* 文件.在极少数情况下,你的项目是共享库(即输出是 *.dll* , *.dylib* 或 *.so* 文件,没有这样的下游`cargo`用户,因此你应该提交 *Cargo.lock* .

*Cargo.toml* 灵活的版本说明符使你可以轻松地在项目中使用Rust库,并最大限度地提高库之间的兼容性. *Cargo.lock* 的簿记支持跨机器的一致性,可重复的构建.它们共同帮助你避免依赖地狱.

## 将Crates发布到crates.io(Publishing Crates to crates.io)

你决定将你的蕨类模拟库发布为开源软件.恭喜!这部分很容易.

首先,确保Cargo可以为你打包crate.

```Shell
$ cargo package
warning: manifest has no description, license, license-file, documentation,
homepage or repository. See http://doc.crates.io/manifest.html#package-metadata
for more info.
   Packaging fern_sim v0.1.0 (file:///.../fern_sim)
   Verifying fern_sim v0.1.0 (file:///.../fern_sim)
   Compiling fern_sim v0.1.0 (file:///.../fern_sim/target/package/fern_sim-0.1.0)
```

`cargo package`命令创建一个文件(在本例中为 *target/package/fern_sim-0.1.0.crate* ),其中包含所有库的源文件,包括 *Cargo.toml* .这是你要上传到[crates.io](https://crates.io/)以与世界分享的文件.(你可以使用`cargo package --list`来查看包含哪些文件.)然后,Cargo通过从 *.crate* 文件构建库来仔细复核其工作,就像最终用户一样.

Cargo警告说, *Cargo.toml* 的`[package]`部分缺少一些对下游用户很重要的信息,例如你分发代码的许可证.警告中的URL是一个很好的资源,因此我们不会在此详细解释所有字段.简而言之,您可以通过向 *Cargo.toml* 添加几行来修复警告:

```Toml
[package]
name = "fern_sim"
version = "0.1.0"
authors = ["You <you@example.com>"]
license = "MIT"
homepage = "https://fernsim.example.com/"
repository = "https://gitlair.com/sporeador/fern_sim"
documentation = "http://fernsim.example.com/docs"
description = """
Fern simulation, from the cellular level up.
"""
```

Note:一旦你在crates.io上发布此包，任何下载你的包的人都可以看到 *Cargo.toml* 文件.因此,如果`authors`字段包含你更想保密的电子邮件地址,那么现在是时候更改它了.

此阶段有时会出现的另一个问题是你的 *Cargo.toml* 文件可能按`path`指定其他crate的位置,如"指定依赖(Specifying Dependencies)"(第185页)中所示:

```Toml
image = { path = "vendor/image" }
```

对于你和你的团队,这可能会很好.但很自然,当其他人下载`fern_sim`库时,他们的计算机上没有相同的文件和目录.因此Cargo会 *忽略(ignores)* 自动下载的库中的`path`键,这可能会导致构建错误.但是,修复很简单:如果你的库将在crates.io上发布,它的依赖项也应该在crates.io上.指定版本号而不是`path`:

```Toml
image = "0.6.1"
```

如果你愿意,可以指定一个`path`,它优先于你自己的本地构建,和一个给所有其他用户的`version`:

```Toml
image = { path = "vendor/image", version = "0.6.1" }
```

当然,在这种情况下,你有责任确保两者保持同步.

最后,在发布crate之前,你需要登录crates.io并获取API密钥.这一步很简单:一旦你在crates.io上拥有一个帐户,你的"帐户设置(Account Settings)"页面将显示`cargo login`命令,如下所示:

```Shell
$ cargo login 5j0dV54BjlXBpUUbfIj7G9DvNl1vsWW1
$
```

Cargo将密钥保存在配置文件中,API密钥应该保密,就像密码一样.因此,只能在你控制的计算机上运行此命令.

完成后,最后一步是运行`cargo publish`:

```Shell
$ cargo publish
    Updating registry `https://github.com/rust-lang/crates.io-index`
   Uploading fern_sim v0.1.0 (file:///.../fern_sim)
```

有了这个,你的库就加入了crates.io上成千上万的其他库中.

## 工作区(Workspaces)

随着你的项目不断发展,你最终会编写许多crate.它们并排存在于单个源存储库中:

```Shell
fernsoft/
├── .git/...
├── fern_sim/
│   ├── Cargo.toml
│   ├── Cargo.lock
│   ├── src/...
│   └── target/...
├── fern_img/
│   ├── Cargo.toml
│   ├── Cargo.lock
│   ├── src/...
│   └── target/...
└── fern_video/
    ├── Cargo.toml
    ├── Cargo.lock
    ├── src/...
    └── target/...
```

Cargo的工作方式,每个crate都有自己的构建目录,`target`,它包含所有crate的依赖项的单独构建.这些构建目录完全独立.即使两个crates具有共同的依赖,它们也不能共享任何已编译的代码.这很浪费.

你可以使用Cargo工作区(共享公共构建目录和 *Cargo.lock* 文件的包集合)来节省编译时间和磁盘空间.

你需要做的就是在存储库的根目录中创建一个 *Cargo.toml* 文件,并将这些行放入其中:

```Toml
[workspace]
members = ["fern_sim", "fern_img", "fern_video"]
```

其中`fern_sim`等是包含你的crates的子目录的名称.删除这些子目录中存在的任何剩余的 *Cargo.lock* 文件和 *target* 目录.

完成此操作后,任何crate中的`cargo build`都将自动创建并使用根目录下的共享构建目录(在本例中为 *fernsoft/target* ).命令`cargo build --all`在当前工作区中构建所有包.`cargo test`和`cargo doc`也接受`--all`选项.

## 更多美好的事情(More Nice Things)

如果你还不高兴的话,Rust社区还有更多的零碎的东西给你:

- 当你在[crates.io](https://crates.io/)上发布开源crate时,感谢Onur Aslan,你的文档会自动呈现并托管在 *docs.rs* 上.

- 如果你的项目在GitHub上,Travis CI可以在每次推送(push)时构建和测试你的代码.设置起来非常简单;有关详细信息,请参阅[travis-ci.org](https://travis-ci.org/).如果你已经熟悉Travis,这个 *.travis.yml* 文件将帮助你入门:

```Yaml
    language: rust
    rust:
      - stable
```

- 你可以从crate的顶级文档注释中生成 *README.md* 文件.此功能由Livio Ribeiro作为第三方Cargo插件提供.运行`cargo install readme`安装插件,然后`cargo readme --help`学习如何使用它.

我们可以继续.

Rust是新的,但它旨在支持大型,雄心勃勃的项目.它有很棒的工具和活跃的社区.系统程序员 *可以(can)* 拥有美好的东西.
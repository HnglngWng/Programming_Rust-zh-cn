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

# Rust之旅(A Tour of Rust)

> 个人的经验完全是根据他的语言建立的.
> --Henri Delacroix

原文

> Toute l'expérience d'un individu est construit sur le plan de son langage.(法语)
> (An individual’s experience is built entirely in terms of his language.)
> --Henri Delacroix

在本章中，我们将介绍几个简短的程序,以了解Rust的语法，类型和语义是如何组合在一起,以支持安全,并发和高效的代码.我们将介绍下载和安装Rust的过程,展示一些简单的数学代码,尝试一个基于第三方库的Web服务器,并使用多线程来加速绘制曼德勃罗集(Mandelbrot set)的过程.

## 下载,安装Rust

安装Rust的最佳方式是使用`rustup`,这是Rust的安装器.访问[https://rustup.rs](https://rustup.rs)并按照那里的说明操作.

您也可以访问[https://www.rust-lang.org](https://www.rust-lang.org),单击"下载"，获取适用于Linux,macOS和Windows的预构建软件包.Rust也包含在某些操作系统发行版中.我们更倾向于`rustup`因为它是用来管理Rust安装的工具,就像RVM之于Ruby或NVM之于Node.比如,当发布新版本的Rust时,你可以通过键入`rustup update`来升级,不需要任何点击.

在任何情况下,一旦安装完成,你的命令行应该有三个新命令:

```shell
$ cargo --version
cargo 0.18.0 (fe7b0cdcf 2017-04-24)
$ rustc --version
rustc 1.17.0 (56124baa9 2017-04-24)
$ rustdoc --version
rustdoc 1.17.0 (56124baa9 2017-04-24)
$
```

这里,`$`是命令提示符;在Windows上,这将是`C:\>`或类似的东西.在这个脚本中,我们运行我们安装的三个命令,要求每个命令报告它是哪个版本.依次介绍每个命令:

- `cargo`是Rust的编译管理器,包管理器和通用工具.你可以使用Cargo启动新项目,构建和运行程序,以及管理代码所依赖的任何外部库.
- `rustc`是Rust的编译器.通常我们让Cargo为我们调用编译器,但有时直接运行它也很有用.
- `rustdoc`是Rust的文档工具.如果你在程序的源代码中使用相应形式的注释编写文档,那么rustdoc可以由它们构建出格式良好的HTML. 像`rustc`一样，我们通常会让Cargo为我们运行`rustdoc`.

为方便起见,Cargo可以为我们创建一个新的Rust包，并妥善地准备好一些标准元数据：

```shell
$ cargo new --bin hello
     Created binary (application) `hello` projec
```

这个命令创建了一个名为*hello*新包目录, `--bin`标识让Cargo准备一个可执行文件而不是一个库.看下包中一级目录:

```shell
$ cd hello
$ ls -la
total 24
drwxrwxr-x.  4 jimb jimb 4096 Sep 22 21:09 .
drwx------. 62 jimb jimb 4096 Sep 22 21:09 ..
drwxrwxr-x.  6 jimb jimb 4096 Sep 22 21:09 .git
-rw-rw-r--.  1 jimb jimb    7 Sep 22 21:09 .gitignore
-rw-rw-r--.  1 jimb jimb   88 Sep 22 21:09 Cargo.toml
drwxrwxr-x.  2 jimb jimb 4096 Sep 22 21:09 src
$
```

我们可以看到Cargo已经创建了一个文件`Cargo.toml`来保存包的元数据.目前这个文件包含的东西不多:

```toml
[package]
name = "hello"
version = "0.1.0"
authors = ["You <you@example.com>"]

[dependencies]
```

如果我们的程序获得了对其他库的依赖,我们可以将它们记录在这个文件中,Cargo将负责为我们下载,构建和更新这些库.我们将在第8章详细介绍`Cargo.toml`文件.

Cargo已经用`git`版本控制系统设置了我们的包,创建了 *.git* 元数据子目录和 *.gitignore* 文件.你可以通过在命令行上指定`--vcs none`来告诉Cargo跳过此步骤.

*src* 子目录包含了实际的Rust代码:

```shell
$ cd src
$ ls -l
total 4
-rw-rw-r--. 1 jimb jimb 45 Sep 22 21:09 main.rs
```

似乎Cargo已经代我们编写了程序. *main.rs* 文件包含文本:

```Rust
fn main() {
    println!("Hello, world!");
}
```

在Rust中,你甚至不需要编写自己的"Hello,World!"程序.这是新建Rust程序的模板：两个文件,总共九行.

我们可以在包中的任何目录调用`cargo run`命令去构建,运行我们的程序:

```shell
$ cargo run
   Compiling hello v0.1.0(file:///home/jimb/rust/hello)
    Finished dev [unoptimized + debuginfo] target(s) in 0.27 secs
     Running `/home/jimb/rust/hello/target/debug/hello`
Hello, world!
$
```

在这里,Cargo调用了Rust编译器`rustc`,然后运行它生成的可执行文件.Cargo将可执行文件放在包顶部的 *target* 子目录中：

```shell
$ ls -l ../target/debug
total 580
drwxrwxr-x. 2 jimb jimb   4096 Sep 22 21:37 build
drwxrwxr-x. 2 jimb jimb   4096 Sep 22 21:37 deps
drwxrwxr-x. 2 jimb jimb   4096 Sep 22 21:37 examples
-rwxrwxr-x. 1 jimb jimb 576632 Sep 22 21:37 hello
-rw-rw-r--. 1 jimb jimb    198 Sep 22 21:37 hello.d
drwxrwxr-x. 2 jimb jimb     68 Sep 22 21:37 incremental
drwxrwxr-x. 2 jimb jimb   4096 Sep 22 21:37 native
$ ../target/debug/hello
Hello, world!
$
```

当我们想时,Cargo可以为我们清理生成的文件:

```shell
$ cargo clean
$ ../target/debug/hello
bash: ../target/debug/hello: No such file or directory
$
```

## 一个简单的函数

Rust的语法故意不独创.如果你熟悉C,C++,Java或JavaScript,则可以通过Rust程序的一般结构找到自己的方法.这是一个使用[欧几里得算法](https://en.wikipedia.org/wiki/Euclidean_algorithm) (Euclid's algorithm)计算两个整数的最大公约数的函数:

```Rust
fn gcd(mut n: u64, mut m: u64) -> u64 {
    assert!(n != 0 && m != 0);
    while m != 0 {
        if m < n {
            let t = m;
            m = n;
            n = t;
        }
        m = m % n;
    }
    n
}
```

`fn`关键字(发音为"fun")引入了一个函数.这里,我们定义了一个名为`gcd`的函数,该函数接收两个参数`n`和`m`,两者的类型都是`u64`--无符号64位整数.`->`标记位于返回类型之前:我们的函数返回一个`u64`值. 4空格缩进是标准的Rust风格.

Rust的机器整数类型名称反映了它们的大小和符号：`i32`是有符号的32位整数;`u8`是无符号的8位整数(用于"字节(byte)"值),等等.`isized`和`usized`类型包含指针大小的有符号和无符号整数,在32位平台上是32位长,在64位平台上是64位长.Rust也有两种浮点类型,`f32`和`f64`,也就是IEEE标准的单精度和双精度浮点类型,就像C和C++中的`float`和`double`.

默认情况下,一个变量一旦被初始化,它的值就不可改变.但是将`mut`关键字(发音"mute", *mutable* 的简写)在放参数`n`和`m`的前面,就允许我们的函数体给它们赋值.在实践中,大多数变量都没有被赋值;在这样的情况下,`mut`关键字对于阅读代码是一个有用的提示.

函数体以调用`assert!`宏开始,验证两个参数都不为零.`!`字符标示着这是宏调用,而不是函数调用.就像C和C++中的`assert`宏一样,Rust的`assert!`检查其参数是否为真,如果不是,则终止程序,并给出包含失败检查的源位置的有用消息;这种突然终止称之为 *pinac*.与C和C++(断言可以被跳过)不同的是,Rust总是检查断言,而无论程序是如何编译的.还有一个`debug_assert!` 宏,当程序追求编译速度时,可以跳过其断言.

我们函数的核心是一个包含`if`语句和赋值的`while`循环.与C和C++不同,Rust在条件表达式周围不需要圆括号,但它需要使用大括号包围它们控制的语句.

`let`语句声明一个局部变量,就像我们函数中的`t`一样.我们不需要写出`t`的类型,只要Rust可以根据变量的使用方式推断出它.在我们的函数中，唯一适用于`t`的类型是`u64`,匹配`m`和`n`.Rust只推断函数体内的类型:你必须写出函数参数和返回值的类型,就像我们之前做的那样.如果我们想给出`t`的类型,我们可以写:

```Rust
let t: u64 = m;
```

Rust有`return`语句,但是`gcd`函数不需要这个.如果函数体以一个 *没有* 分号的表达式结束,那它就是函数的返回值.实际上，任何由大括号包围的块都可以用作表达式.例如,这是一个打印消息然后将`x.cos()`作为其值的表达式：

```Rust
{
    println!("evaluation cos x");
    x.cos()
}
```

在Rust中,通常在函数的控制结构"直达末尾(falls
off the end)"时使用这种形式建立函数的值,仅在从函数中间显式提前返回时使用`return`语句.

## 编写,运行单元测试

Rust语言内置了对测试的简单支持.为了测试我们的`gcd`函数,可以这样写:

```Rust
#[test]
fn test_gcd() {
    assert_eq!(gcd(14, 15), 1);
    assert_eq!(gcd(2 * 3 * 5 * 11 * 17,
                   3 * 7 * 11 * 13 * 19),
                3 * 11);
}
```

这里我们定义一个名为`test_gcd`的函数,它调用`gcd`并检查它是否返回正确的值.定义在上面的`[#test]`标记`test_gcd`作为一个测试函数,在正常编译中会跳过,但如果我们使用`cargo test`命令运行我们的程序,它将自动包含和调用.假如我们将`gcd`和`test_gcd`的定义编辑到我们在本章开头创建的 *hello* 包中.如果我们当前的目录位于包的子树中,我们可以按如下方式运行测试:

```shell
$ cargo test
   Compiling hello v0.1.0 (file:///home/jimb/rust/hello)
    Finished dev [unoptimized + debuginfo] target(s) in 0.35 secs
     Running /home/jimb/rust/hello/target/debug/deps/hello-2375a82d9e9673d7

running 1 test
test test_gcd ... ok

test result: ok. 1 passed; 0 failed; 0 ignored; 0 measured

$
```

我们可以将测试函数分散在我们的源代码树中,放在他们要测试代码旁边,`cargo test`将自动收集它们并运行.

`[#test]`标记是 *属性(attribute)* 的一个例子.属性是一个开放式系统,用于使用额外信息标记函数和其它声明,就像C++和C#中的属性,或Java中的注解.

它们用来控制编译警告和代码风格检查,有条件地包含代码(像C和C++中的`#ifdef`),告诉Rust如何与用其他语言编写的代码进行交互,等等.我们将会看到更多属性示例.

## 处理命令行参数

如果我们希望程序从命令行参数中获取一系列数字并打印出它们最大的公约数,我们可以用以下代码替换`main`函数:

```Rust
use std::io::Write;
use std::str::FromStr;

fn main(){
    let mut numbers = Vec::new();

    for arg in std::env::args().skip(1) {
        numbers.push(u64::from_str(&arg).expect("error parsing argument"));
    }

    if numbers.len() == 0 {
        writeln!(std::io::stderr(), "Usage: gcd NUMBER ...").unwrap();
        std::process::exit(1);
    }

    let mut d = numbers[0];
    for m in &numbers[1..] {
        d = gcd(d, *m);
    }

    println!("The greatest common divisor of {:?} is {}", numbers, d);
}
```

这是一大块代码，所以让我们一小块一小块来:

```Rust
use std::io::Write;
use std::str::FromStr;
```

`use`声明引入`Write`和`FromStr`两个 *traits* 作用域.我们将在第11章详细介绍traits,但是这里简单书一下,trait是一个类型可以实现的方法的集合.虽然我们在程序中哪里都没有使用`Write`和`FromStr`的名字,但是为了使用它的方法trait必须在作用域中.在本例中:

- 任何实现`Write`trait的类型都有`write_fmt`方法,去格式化文本到流中.`std::io::Stderr`类型实现了`Write`,我们使用`writeln!`宏去打印错误消息,这宏展开代码使用了`write_fmt`方法.

- 任何实现`FromStr`trait的类型都有一个`from_str`方法,去从字符串中解析一个值.`u64`类型实现了`FromStr`,我们调用`u64::from_str`去解析我们的命令行参数.

继续学习程序的`main`函数:

```Rust
fn main() {
```

我们的`main`函数没有返回值,所以我们可以简单地省略`->`和通常遵循的参数列表的类型.

```Rust
let mut numbers = Vec::new();
```

我们声明了一个可变的局部变量`numbers`,将其初始化为一个空的向量(vector).`Vec`是Rust的可变长向量类型,类似与C++的`std::vector`,Python的列表(list)或是JavaScript的数组(array).即使向量被设计为动态增长和收缩的,我们仍然必须标记变量`mut`,以便Rust允许我们将数字推入它的末尾.

`numbers`的类型是`Vec<u64>`,一个`u64`值的向量,但是和以前一样,我们不需要写出来.Rust会帮我们推断,部分是因为我们推入向量的是`u64`值,还因为我们将向量的元素传递给了`gcd`,它只接受u64值.

```Rust
for arg in std::env::args().skip(1) {
```

我们使用`for`循环来处理我们的命令行参数,依次将变量`arg`设置为每个参数,然后对循环体求值.

`std::env::args`函数返回一个 *迭代器(iterator)* ,按需生成每个参数的值,并指示我们何时完成.迭代器在Rust中无处不在;标准库包含其它迭代器,它们生成向量的元素,文件的行,在通信通道上接收的消息,以及几乎任何有意义的循环.Rust的迭代器非常高效:编译器通常能够将它们转换为与手写循环相同的代码.我们将在第15章中展示它的工作原理并举例说明.

除了使用`for`循环之外,迭代器还包括可以直接使用的各种方法.例如,`std::env::args`返回的迭代器生成的第一个值始终是正在运行的程序的名称.我们想跳过它,所以我们调用迭代器的`skip`方法来生成一个省略第一个值的新迭代器.

```Rust
numbers.push(u64::from_str(&arg)
        .expect("error parsing argument"));
```

这里我们调用`u64::from_str`试图将我们的命令行参数解析为64位无符号整数.`u64::from_str`是一个与`u64`类型相关的函数,而不是我们在某些`u64`值上调用的方法,类似于C++或Java中的静态方法.`from_str`函数不直接返回`u64`,而是返回一个指示解析是成功还是失败的`Result`值.`Result`值是两个变量之一:

- 一个值是`Ok(v)`,表示解析成功,`v`是生成的值.
- 一个值是`Err(e)`,表示解析失败,`e`是解释原因的错误值.

执行输入或输出或以其他方式与操作系统交互的函数都返回`Result`,它的`Ok`变量带有成功结果-传输的字节数,打开的文件等等,`Err`变量带有来自系统的错误码.
和大部分现代语言不一样,Rust没有异常:所有的错误都使用`Result`或者panic处理,如第7章所述.

我们使用`Result`的`expect`方法检查解析是否成功.如果结果是`Err(e)`,`expect`打印一条包含`e`描述的消息,并立即退出程序.如果结果是`Ok(v)`,`expect`简单地返回`v`本身,我们最终就可以将其推入数字向量的末尾.

```Rust
if numbers.len() == 0{
    writenln!(std::io::stderr(), "Usage: gcd NUMBER ...").unwrap();
    std::process::exit(1);
}
```

数字的空集没有最大公约数,所以我们检查我们的向量是否至少有一个元素,如果没有,则退出程序并返回错误.我们使用`writenln!`宏将错误信息写出到`std::io::stderr()`提供的标准错误流.`.unwrap`调用是一种简洁的方法来检查打印错误消息的尝试本身是否失败;调用`expect`也可以,但这可能没必要.

```Rust
let mut d = numbers[0];
for m in &numbers[1..] {
    d = gcd(d, *m);
}
```

这个循环使用`d`作为其运行值,更新它以保存到目前为止我们处理的所有数字的最大公约数.和以前一样,我们必须将`d`标记为可变,以便我们可以在循环中为其赋值.

这个`for`循环有两个出乎意料的点.首先,我们编写的是`for m in &numbers[1..]`;`&`是什么操作符?其次,我们编写了`gcd(d, *m)`;`*m`中的`*`又是什么?这两个细节是相互补充的.

到目前为止,我们的代码只对简单的值进行操作,例如固定大小的内存块的整数.但是现在我们要迭代一个向量,它可以是任何大小--可能非常大.Rust在处理这些值时非常谨慎:它希望让程序员控制内存消耗,明确每个值的存在时间,同时确保在不再需要时立即释放内存.

因此,当我们迭代时,我们想告诉Rust,向量的 *所有权(ownership)* 应该保留在`numbers`中;我们只是为了循环 *借用(borrowing)* 它的元素.`&numbers[1..]`中的`&`操作符借了从向量第二个元素开始的 *引用(reference)* .`for`循环遍历引用的元素,让`m`连续借用每个元素.`*m`中的`*`操作符 *解引用(dereferences)* `m`,产生它引用的值.这是我们想要传递给`gcd`的下一个`u64`.最后，由于``numbers`拥有向量,当`numbers`超出`main`结尾的范围时,Rust会自动释放它.

Rust关于所有权和应用的规则是Rust内存管理和安全并发的关键;我们将在第4章和第5章中详细讨论它们和有关的东西.你需要适应规则才能适应Rust,但是对于这个介绍性的游览,你需要知道的是`&x`借了对`x`的引用,并且`*r`是`r`所引用的值.

继续我们的程序:

```Rust
println!("The greatest common divisor of {:?} is {}",
    numbers, d);
```

迭代了`numbers`的元素后,程序将结果打印到标准输出流.`println!`宏接受模板字符串,替换模板字符串中出现的`{...}`形式为剩余参数的格式化版本,并将结果写入标准输出流.

与C和C++不同的是,它们的程序成功结束后,`main`函数需要返回0,如果出错了,返回非0的退出状态.Rust假定如果`main`返回,则程序成功完成.只有通过显式调用`expect`或`std::process::exit`等函数,我们才能使程序以错误状态码终止.

`cargo run`命令允许我们向程序传递参数,所以我们可以尝试我们的命令行处理:

```shell
$ cargo run 42 56
   Compiling hello v0.1.0 (file:///home/jimb/rust/hello)
    Finished dev [unoptimized + debuginfo] target(s) in 0.38 secs
     Running `/home/jimb/rust/hello/target/debug/hello 42 56`
The greatest common divisor of [42, 56] is 14
$ cargo run 799459 28823 27347
    Finished dev [unoptimized + debuginfo] target(s) in 0.0 secs
     Running `/home/jimb/rust/hello/target/debug/hello 799459 28823 27347`
The greatest common divisor of [799459, 28823, 27347] is 41
$ cargo run 83
    Finished dev [unoptimized + debuginfo] target(s) in 0.0 secs
     Running `/home/jimb/rust/hello/target/debug/hello 83`
The greatest common divisor of [83] is 83
$ cargo run
    Finished dev [unoptimized + debuginfo] target(s) in 0.0 secs
     Running `/home/jimb/rust/hello/target/debug/hello`
Usage: gcd NUMBER ...
$
```

在这一节,我们使用了一些标准库中的特性.如果你想知道还有什么可用的,我们强烈建议您尝试使用Rust的在线文档.它有一个实时搜索功能,使探索变得容易,甚至包括到源代码的链接.当你安装Rust时,`rustup`命令会自动在你的电脑上安装一个副本.你可以用以下命令在自己的浏览器中查看标准库文档:

```shell
$ rustup doc --std
$
```

你也可以在网上查看[https://doc.rust-lang.org/](https://doc.rust-lang.org/).

## 一个简单的Web服务器

Rust的优势之一是发布在[crates.io](https://crates.io/)网站上的免费库包集合.`cargo`使得我们自己的代码可以轻松使用crates.io的包:它将下载正确版本的软件包,构建它,并按要求更新.Rust的软件包不管是库还是可执行文件都被称为 *crate* ; Cargo和crates.io的名字都来源于这一术语.

为了说明这是如何工作的,我们将使用`iron`web框架,`hyper`HTTP服务器以及它们所依赖的各种其它crates拼凑一个简单的web服务器.如图2-1所示,我们的网站将提示用户输入两个数字,然后计算他们最大的公约数.

*图2-1.计算GCD的Web页*

首先,我们使用Cargo为我们创建一个新包,名字叫做`iron-gcd`:

```shell
$ cargo new --bin iron-gcd
     Created binary (application) `iron-gcd` project
$ cd iron-gcd
$
```

然后,我们将编辑新项目的`Cargo.toml`文件以列出我们想要使用的包;其内容应如下：

```Toml
[package]
name = "iron-gcd"
version = "0.1.0"
authors = ["You <you@example.com>"]

[dependencies]
iron = "0.5.1"
mime = "0.2.3"
router = "0.5.1"
urlencoded = "0.5.0"
```

`Cargo.toml`的`[dependencies]`部分的每一行都给出了crates.io上一个crate的名字,以及我们想要使用的crate的版本.Crates.io上的这些crate的版本可能比这里显示的更新,但通过命名我们测试此代码的特定版本,我们可以确保即使新版本的软件包已发布,代码仍能继续编译.我们将在第8章讨论版本管理的更多细节.

注意我们只需要命名我们将直接使用的包;`cargo`负责引入其它需要的包.

对于我们的第一次迭代,我们将保持Web服务器简单:它将仅提供提示用户输入要计算的数字的页面.在 *iron-gcd/src/main.rs* 中,我们输入以下文本:

```Rust
extern crate iron;
#[macro_use] extern crate mime;

use iron::prelude::*;
use iron::status;

fn main(){
    println!("Serving on http://localhost:3000...");
    Iron::new(get_form).http("local:3000").unwrap();
}

fn get_form(_request: &mut Request) -> IronResult<Response> {
    let mut response = Response::new();
    response.set_mut(status::Ok);
    response.set_mut(mime!(Text/Html; Charset=Utf8));
    response.set_mut(r#"
        <title>GCD Calculator</title>
        <form action="/gcd" method="post">
          <input type="text" name="n"/>
          <input type="text" name="n"/>
          <button type="submit">Compute GCD</button>
        </form>
    "#);

    Ok(response)
}
```

我们从两个`extern crate`指令开始,这些指令使我们在 *Cargo.toml* 文件中引用的`iron`和`mime`crates可用于我们的程序.`extern crate mime`项之前的`#[macro_use]`属性警告Rust我们计划使用此crate导出的宏.

接下来,我们使用`use`声明来引入这些crates的公共功能.`iron::prelude::*`声明使`iron::prelude`模块所有的公开名字都能直接显示在我们的代码中.一般来说,最好拼出你想要使用的名字,就像我们为`iron::status`所做的那样;但按照惯例,当一个模块被命名为prelude时,这意味着它的导出旨在提供该crate的任何用户都可能需要的通用设施.在这种情况下,通配符`use`指令更有意义.

我们的`main`函数很简单:它会打印一条消息,提醒我们如何连接到我们的服务器,调用`Iron::new`创建一个服务器,然后将其设置为监听本机上的TCP端口3000.我们将`get_form`函数传递给`Iron::new`,表明服务器应该使用该函数来处理所有请求;我们很快就会对此进行改进.

`get_form`函数本身接收一个可变引用(写为`&mut`)引用到一个`Request`值,该值表示我们已经调用的HTTP请求.虽然这个特殊的处理函数没有使用它的`_request`参数,但我们稍后会看到它.暂时给参数一个以`_`开头的名称告诉Rust我们希望该变量未被使用,因此它不应该警告我们.

在函数体中,我们构建了一个`Response`值.`set_mut`方法使用它的参数类型来决定设置响应的那个部分,所以每个`set_mut`的调用实际设置了`response`的不同部分:传递`status::Ok`设置HTTP状态;传递内容的媒体类型(使用我们从`mime`crate导入的方便的`mime!`宏),设置`Content-Type`头;传递字符串设置响应体.

因为响应文本中包含许多双引号,所以我们使用Rust的"原始字符串(raw string)"语法:字母`r`,零个或多个哈希标记(即`#`字符),双引号,然后是字符串的内容,由另一个双引号后跟相同数量的哈希标记终止.任何字符都可以出现在原始字符串中而不会被转义,包括双引号;实际上,像`\"`这样转义序列不会被识别.

我们函数的返回类型`IronResult<Response>`是`Result`类型的另一个变体,我们之前遇到过:对于某些成功的`Response`值`r`,这是`Ok(r)`,对于某些错误值`e`,这是Err(e).我们构建我们的返回值`Ok(response)`在函数体的底部,使用"最后表达式"语法隐式地指定函数的返回值.

编写完 *main.rs* ,我们可以使用`cargo run`命令去做运行它所需的一切:获取需要的包,编译它们,构建我们自己的程序,将所有内容链接在一起,然后启动它:

```Rust
$ cargo run
    Updating registry `https://github.com/rust-lang/crates.io-index`
 Downloading iron v0.5.1
 Downloading urlencoded v0.5.0
 Downloading router v0.5.1
  Downloading hyper v0.10.8
 Downloading lazy_static v0.2.8
 Downloading bodyparser v0.5.0
...
   Compiling conduit-mime-types v0.7.3
   Compiling iron v0.5.1
   Compiling router v0.5.1
   Compiling persistent v0.3.0
   Compiling bodyparser v0.5.0
   Compiling urlencoded v0.5.0
   Compiling iron-gcd v0.1.0 (file:///home/jimb/rust/iron-gcd)
     Running `target/debug/iron-gcd`
Serving on http://localhost:3000...
```

此时,我们可以在浏览器中访问给定的URL,并查看前面图2-1中显示的页面.

不幸的是,单击Compute GCD不会执行任何操作,只会将浏览器导航到URL *http://localhost:3000/gcd* ,然后显示相同的页面;实际上,我们服务器上的每个URL都是这样做的.接下来让我们解决这个问题,使用`Router`类型将不同的处理程序与不同的路径相关联.

首先,让我们将如下声明添加到 *iron-gcd/src/main.rs* 来使用`Router`:

```Rust
extern crate router;
use router::Router;
```

Rust程序员通常会将他们所有的`extern crate`和`use`声明一起放在文件顶部.但这不是强制要求:Rust允许声明以任何顺序出现,只要它们出现在适当的嵌套级别.(宏定义和具有`#[macro_use]`属性的`extern crate`是此规则的例外:它们必须在使用之前出现.)

然后我们可以修改我们的`main`函数,如下所示:

```Rust
fn main() {
    let mut router = Router ::new();

    router.get("/", get_form , "root");
    router.post("/gcd", post_gcd , "gcd");

    println!("Serving on http://localhost:3000...");
    Iron::new(router).http("localhost:3000").unwrap();
}
```

我们创建了一个`Router`,为两个特定路径建立处理函数,然后将此`Router`作为请求处理器传递给`Iron::new`,从而建立一个Web服务器,该服务器参考URL路径来决定调用哪个处理函数.

现在我们开始编写`post_gcd`函数:

```Rust
extern crate urlencoded;

use std::str::FromStr;
use urlencoded::UrlEncodedBody;

fn post_gcd(request: &mut Request) -> IronResult<Response> {
    let mut response = Response::new();
    let form_data = match request.get_ref::<UrlEncodedBody>() {
        Err(e) => {
            response.set_mut(status::BadRequest);
            response.set_mut(format!("Error parsing form data: {:?}\n", e));
            return Ok(response);
        }
        Ok(map) => map
    };

    let unparsed_numbers = match form_data.get("n") {
        None => {
            response.set_mut(status::BadRequest);
            response.set_mut(format!("form data has no 'n' parameter\n"));
            return Ok(response);
        }
        Some(nums) => nums
    };

    let mut numbers = Vec::new();
    for unparsed in unparsed_numbers {
        match u64::from_str(&unparsed) {
            Err(_) => {
                response.set_mut(status::BadRequest);
                response.set_mut(format!("Value for 'n' parameter not a number: {:?}\n", unparsed));
                return Ok(response);
            }
            Ok(n) => { numbers.push(n); }
        }
    }

    let mut d = numbers[0];
    for m in &numbers[1..] {
        d = gcd(d, *m);
    }

    response.set_mut(status::Ok);
    response.set_mut(mime!(Text/Html; Charset=Utf8));
    response.set_mut(format!("The greatest common divisor of the numbers {:?} is <b>{}</b>\n", numbers, d));
    Ok(response)
```

这个函数的大部分是一系列`match`表达式,C,C++,Java和JavaScript程序员可能对此不太熟悉,但对于那些使用Haskell和OCaml的人来说是一个受欢迎的景象. 我们提到过`Result`对于某些成功值`s`是值`Ok`(s),或者对于某些误差值`e`是`Err(e)`.给定一些`Result res`,我们可以检查它是哪个变量,并使用如下形式的`match`表达式访问它所拥有的任何值:

```Rust
match res{
    Ok(success) => { ... },
    Err(error)  => { ... }
}
```

这是视条件而定的,像C中的`if`语句或是`switch`语句:如果`res`是`Ok(v)`,运行第一个分支,变量`success`设置为`v`.类似地,如果`res`是`Err(e)`,运行第二个分支,`error`设置为`e`.`success`和`error`变量都是其分支的本地变量.整个匹配表达式的值是运行的分支的值.

`match`表达式的优点在于程序只能通过首先检查`Result`的变量来访问其值;绝不会将失败值误解为成功完成.在C和C++中,忘记检查错误码或空指针是一个常见错误,在Rust中,这些错误在编译时被捕获.这个简单的措施是可用性的重大进步.

Rust允许您用带有值的变量定义自己的类似与`Result`的类型,并使用`match`表达式来分析它们.Rust称这些类型为 *枚举(enums)* ;你可能会从其他语言中得知它们,称为*代数数据类型(algebraic data types)* . 我们将在第10章详细介绍枚举.

既然你可以阅读`match`表达式,`post_gcd`的结构应该很清楚了：

- 它调用`request.get_ref::<UrlEncodedBody>()`来解析请求体,将查询参数名称映射到值数组;如果解析失败,它会将错误报告给客户端.方法的`::<UrlEncodedBody>`部分是一个 *类型参数(type parameter)* ,指示`get_ref`应该检索`Request`的哪个部分.在这种情况下,`UrlEncodedBody`类型引用请求正文,解析为URL编码的查询字符串.我们将在下一节中详细讨论类型参数.
- 在该表中,它找到名为"`n`"的参数的值,这是HTML表单将输入的数字放入了网页.此值不是单个字符串,而是一个字符串向量,因为可以查询参数名字可以重复.
- 它遍历字符串向量,将每个字符串解析为无符号的64位数字,并在任一字符串无法解析时返回相应的失败页面.
- 最后,和之前一样计算数字的最大公约数,而后构造一个描述结果的响应.`format!`宏使用同`writeln!`,`println!`宏一样的字符串模板,但是返回一个字符串值,而不是将文本写入流.

剩下的最后一块是我们之前写的`gcd`函数.有了这些,你可以中断可能已经运行的任何服务器,重新构建并重启该程序：

```shell
$ cargo run
   Compiling iron-gcd v0.1.0 (file:///home/jimb/rust/iron-gcd)
    Finished dev [unoptimized + debuginfo] target(s) in 0.0 secs
     Running `target/debug/iron-gcd`
Serving on http://localhost:3000...
```

这时,访问 *http://localhost:3000* ,输入一些数字,点击计算GCD按钮,你应该看到一些结果(图2-2).

*图2-2. 显示计算GCD结果的Web页*

## 并发

Rust的一大优势是它支持并发编程.确保Rust程序没有内存错误的相同规则也确保线程只能以避免数据竞争的方式共享内存.例如:

- 如果使用互斥锁来协调线程更改共享数据结构,Rust会确保你不能访问数据,除非您持有锁,并在操作后自动释放锁.在C和C++中,互斥锁与其保护的数据之间的关系留给了注释.
- 如果要在多个线程之间共享只读数据,Rust会确保您不会意外修改数据.在C和C++中，类型系统可以帮助解决这个问题,但很容易弄错.
- 如果你将数据结构的所有权从一个线程转移到另一个线程,Rust确保您确实放弃了对它的所有访问权限.在C和C++中,由您决定发送线程上的任何内容都不会再次触及数据.如果你做得不对,效果可能取决于处理器缓存中发生的事情以及你最近对内存的写入次数.这样我们很痛苦.

在本节中,我们将引导您完成编写第二个多线程程序的过程.虽然你可能没有意识到这一点，但你已经写完第一个了:你用来实现Greatest Common Divisor服务器的Iron Web框架使用线程池来运行请求处理函数.如果服务器同时收到请求,它可以同时在多个线程中运行`get_form`和`post_gcd`函数.这可能有点令人震惊,因为当我们编写这些函数时,我们确实没有考虑并发性.但Rust保证这是安全的,无论你的服务器有多精细:如果你的程序可以编译,它就没有数据竞争.所有Rust函数都是线程安全的.

本节的程序绘制了Mandelbrot集,它是一个通过在复数上迭代一个简单函数而产生的分形.绘制Mandelbrot集通常被称为 *尴尬的并行(embarrassingly parallel
)* 算法,因为线程之间的通信模式非常简单;我们将在第19章介绍更复杂的模式，但是这个任务演示了一些基本要素.

首先，我们将创建一个新的Rust项目：

```Rust
$ cargo new --bin mandelbrot
     Created binary (application) `mandelbrot` project
```

所有代码都将放在 *mandelbrot/src/main.rs中,我们将向 *mandelbrot/Cargo.toml* 添加一些依赖项.

在我们进入并发Mandelbrot实现之前,需要描述下要执行的计算.

### Mandelbrot实际上是什么

在阅读代码时,对它试图做什么有一个具体的概念是有帮助的.那么让我们进入一些纯数学的短暂旅行.我们将从一个简单的例子开始,然后添加复杂的细节,直到我们到达Mandelbrot集的核心计算.

这是一个无限循环,使用Rust的专用语法编写,`loop`语句:

```Rust
fn square_loop(mut x: f64){
    loop{
        x = x * x;
    }
}
```

实际上,Rust可以看出`x`从未作它用,因此可能不会费心计算它的值.但就目前而言,先假设代码按所写的方式运行.x的值会有什么变化?平方计算任何小于1的数字会使其变小,因此它接近零;平方计算1得到的还是1;平方计算大于1的数字会使其变大,因此它接近无穷大;并且对负数进行平方计算使其成为正数,之后它表现为先前的情况之一(图2-3).

*图2-3.反复计算一个数平方的效果*

因此,根据您传递给`square_loop`的值,`x`接近零,保持为1或接近无穷大.

现在考虑一个稍微不同的循环:

```Rust
fn square_add_loop(c: f64) {
    let mut x = 0.;
    loop {
        x = x * x + c;
    }
}
```

这一次,`x`从零开始,我们通过在平方后添加`c`来调整每次迭代的进度.这使得很难看出`x`如何变化,但是一些实验表明,如果`c`大于0.25,或者小于-2.0,则`x`最终变得无限大;否则,它会停留在零附近的某个地方.

下一个问题:不使用`f64`值,而是使用复数来考虑相同的循环.crates.io上的`num`crate提供了我们可以使用的复数类型,因此我们必须在我们程序的 *Cargo.toml* 文件的`[dependencies]`部分添加`num`行.这是整个文件,直到目前为止(稍后还会添加):

```Toml
[package]
name = "mandelbrot"
version = "0.1.0"
authors = ["You <you@example.com>"]

[dependencies]
num = "0.1.27"
```

现在我们可以编写循环的倒数第二个版本:

```Rust
extern crate num;
use num::Complex;

#[allow(dead_code)]
fn complex_square_add_loop(c: Complex<f64>) {
    let mut z = Complex { re: 0.0, im: 0.0 };
    loop {
        z = z * z + c;
    }
}
```

使用`z`表示复数是惯例,所以我们重命名了循环变量.`Complex { re: 0.0, im: 0.0 }`表达式是`num`crate的`Complex`类型表示复数零的方式.`Complex`是Rust的结构体类型(或 *结构(struct)* ),像这样定义:

```Rust
struct Complex<T> {
    /// Real portion of the complex number
    re: T,
    /// Imaginary portion of the complex number
    im: T
}
```

上面的代码定义了一个名为`Complex`的结构,它有两个字段`re`和`im`.`Complex`是一个泛型结构体:你可以读`<T>`于类型名后"对于任何类型T".例如,`Complex<f64>`是一个复数,它的`re`和`im`是`f64`值,`Complex<f32>`想使用32位浮点数,等等.根据这个定义,像`Complex {re：R, im：I}`这样的表达式产生一个`Complex`值,其`re`字段初始化为`R`,其`im`字段初始化为`I`.

`num`crate安排`*`,`+`和其他算术运算符处理`Complex`值,因此函数的其余部分与先前版本一样,除了它在复平面上的点上操作，而不仅仅是在实数线上的点.我们将在第12章解释如何让Rust的操作符可以在你自己的类型上工作.

最后,我们已经到达了纯数学游览的目的地.Mandelbrot集被定义为复数`c`的集合,其中`z`不会飞到无穷大.我们最初的简单平方循环足够可预测:任何大于1或小于-1的数字都会飞走.在每次迭代中加入`+c`会使行为更难以预测:正如我们之前所说,`c`值大于0.25或小于-2会导致`z`飞走.但是将游戏扩展到复数会产生真正奇异而美丽的图案,这就是我们想要绘制的图案.

由于复数`c`具有实部和虚部`c.re`和`c.im`,我们将这些视为笛卡尔平面上某点的`x`和`y`坐标,如果`c`在Mandelbrot集合中,则将该点着色为黑色,否则为浅色.因此,对于我们图像中的每个像素,我们必须在复平面上的相应点上运行前一个循环,看它是否永远逃逸到无限远或绕原点旋转,并相应地对其进行着色.

无限循环需要一段时间才能运行,但对于急切的人有两个技巧.首先,如果我们放弃永远运行循环并仅尝试一些有限数量的迭代,事实证明我们仍然得到了相当大的集合近似值.我们需要多少次迭代取决于我们想要绘制边界的精确程度.其次,已经证明,如果`z`一旦离开以原点为中心的半径为2的圆,它最终肯定会远离原点飞得无限远.

所以这就是我们循环的最终版本,也是程序的核心:

```Rust
extern crate num;
use num::Complex;

/// Try to determine if `c` is in the Mandelbrot set, using at most `limit`
/// iterations to decide.
///
/// If `c` is not a member, return `Some(i)`, where `i` is the number of
/// iterations it took for `c` to leave the circle of radius two centered on the
/// origin. If `c` seems to be a member (more precisely, if we reached the
/// iteration limit without being able to prove that `c` is not a member),
/// return `None`.
fn escape_time(c: Complex<f64>, limit: u32)-> Option<u32> {
    letmut z = Complex { re: 0.0, im: 0.0 };
    for i in 0..limit {
        z = z*z + c;
        if z.norm_sqr() > 4.0 {
            return Some(i);
        }
    }
    None
}
```

此函数接收我们要在Mandelbrot集中测试成员资格的复数c,和在放弃声明`c`可能是成员之前尝试的迭代次数限制.

函数返回一个`Option<u32>`,Rust标准库如下定义`Option`类型:

```Rust
enum Option<T> {
    None,
    Some(T),
}
```

`Option`是一个 *可枚举(enumerated)* 类型,一般称之为 *枚举(enum)* ,因为它的定义列举了几种变体,这种类型的值可能是:对于任意的类型`T`,类型`Option<T>`的值是`Some(v)`,这里`v`是`T`类型的一个值;或者`None`,表示没有`T`值可用.类似于我们之前讨论的`Complex`类型,`Option`是个泛型类型:你可以使用`Option<T>`表示你喜欢的任何类型`T`的可选值.

在我们的例子中,`escope_time`返回一个`Option<u32>`表示`c`是否在Mandelbrot集中--如果不在,我们需要多长时间才能找到它.如果`c`不在集合中,`escope_time`返回`Some(i)`,这里`i`是`z`离开半径为2的圆的迭代次数.否则,`c`显然在集合中,`escope_time`返回`None`.

```Rust
for i in 0..limit {
```

前面的例子显示`for`循环迭代命令行参数和向量元素;这个for循环只是遍历从0开始最大到limit(但不包括)的整数范围.

`z.norm_sqr()`方法调用返回`z`到原点的距离的平方.要确定`z`是否已离开半径为2的圆,我们只需将平方距离与4.0进行比较而不是计算平方根,这样会更快.

你可能已经注意到了我们使用`///`来标记函数定义上方的注释行;`Complex`结构体成员上面的注释也以`///`开头.这些是 *文档注释(documents comments))* ;`rustdoc`工具知道如何解析它们以及它们描述的代码,并生成在线文档.Rust标准库的文档就是用这个形式编写的.我们将在第8章中详细描述文档注释.

程序的其余部分涉及决定集合的哪个部分以什么分辨率绘制,并将工作分布在多个线程上以加速计算.

### 解析一对命令行参数

该程序需要获取几个命令行参数来控制我们将要编写的图像的分辨率,并且Mandelbrot集合图像显示的部分.由于这些命令行参数都遵循一个共同的形式,这里是一个解析它们的函数:

```Rust
use std::str::FromStr;
/// Parse the string `s` as a coordinate pair, like `"400x600"` or `"1.0,0.5"`.
///
/// Specifically, `s` should have the form <left><sep><right>, where <sep> is
/// the character given by the `separator` argument, and <left> and <right> are both
/// strings that can be parsed by `T::from_str`.
///
/// If `s` has the proper form, return `Some<(x, y)>`. If it doesn't parse
/// correctly, return `None`.
fn parse_pair<T: FromStr>(s: &str, separator: char) -> Option<(T, T)> {
    match s.find(separator) {
        None => None,
        Some(index) => {
            match (T::from_str(&s[..index]), T::from_str(&s[index + 1..])) {
                (Ok(l), Ok(r)) => Some((l, r)),
                _ => None
            }
        }
    }
}

#[test]
fn test_parse_pair() {
    assert_eq!(parse_pair::<i32>("",        ','), None);
    assert_eq!(parse_pair::<i32>("10,",     ','), None);
    assert_eq!(parse_pair::<i32>(",10",     ','), None);
    assert_eq!(parse_pair::<i32>("10,20",   ','), Some((10, 20)));
    assert_eq!(parse_pair::<i32>("10,20xy", ','), None);
    assert_eq!(parse_pair::<f64>("0.5x",    'x'), None);
    assert_eq!(parse_pair::<f64>("0.5x1.5", 'x'), Some((0.5, 1.5)));
}
```

`parse_pair`定义为一个 *泛型函数(generic function)* :

```Rust
fn parse_pair<T: FromStr>(s: &str, separator: char) -> Option<(T, T)> {
```

你可以将<T：FromStr>子句大声地读作,"对于实现`FromStr`trait的任何类型`T`...".这有效地让我们一次定义整个函数族：`parse_pair::<i32>`是一个解析`i32`值对的函数;`parse_pair::<f64>`解析浮点值对;等等.这非常像C++中的函数模板.Rust程序员将`T`称为`parse_pair`的 *类型参数(type parameter)* .当你使用泛型函数时,Rust通常都能帮你推断类型参数,你不需要像在测试代码中那样把它们写出来.

我们的返回类型时`Option<(T, T)>`:`None`或是一个值`Some((v1, v2))`,`(v1, v2)`是个两个值组成的元组,类型都是`T`.`parse_pair`函数没有使用显示返回语句,因此返回值是函数体中最后一个(只有一个)表达式的值:

```Rust
match s.find(separator){
    None => None,
    Some(index) =>{
        ...
    }
}
```

`String`类型的`find`方法再字符串中搜索与`separator`相匹配的字符.如果`find`返回`None`,意味着字符串中没有出现`separator`字符,整个`match`表达式的值就是`None`,表示解析失败.否则,我们将`index`作为分隔符在字符串中的位置.

```Rust
match (T::from_str(&s[..index], T::from_str(&s[index + 1..]))){
    (Ok(l), Ok(r)) => Some((l, r)),
    _ =>None
}
```

这开始展示`match`表达式的力量.匹配的参数是这个元组表达式:

```Rust
(T::from_str(&s[..index]), T::from_str(&s[index + 1..]))
```

表达式`&s[..index]`和`[index + 1..]`是在分隔符之前和之后的字符串切片.类型参数`T`的关联`from_str`函数接受其中的每一个,并尝试将它们解析为类型`T`的值,从而产生结果元组.我们再次匹配:

```Rust
(Ok(l), Ok(r)) => Some((l, r)),
```

这个模式只有元组的两个元素都是`Result`类型的`Ok`变量时才匹配,表示两个都解析成功.若此,`Some((l, r))`就是匹配表达式的值,因此是函数的返回值.

```Rust
_ => None
```

通配符模式`_`匹配任何内容,并忽略其值.如果我们达到这一点,那么`parse_pair`失败了,所以我们计算为None,再次提供函数的返回值.

现在我们有了`parse_pair`,很容易编写一个函数来解析一对浮点坐标并将它们作为`Complex <f64>`值返回:

```Rust
/// Parse a pair of floating-point numbers separated by a comma as a complex
/// number.
fn parse_complex(s: &str) -> Option<Complex<f64>> {
    match parse_pair(s, ',') {
        Some((re, im)) => Some(Complex { re, im }),
        None => None
    }
}

#[test]
fn test_parse_complex() {
    assert_eq!(parse_complex("1.25,-0.0625"),
            Some(Complex { re: 1.25, im: -0.0625 }));
    assert_eq!(parse_complex(",-0.0625"), None);
}
```

`parse_complex`函数调用`parse_pair`,如果成功解析了坐标,则构建`Complex`值,否则将失败传递给其调用者.

如果您仔细阅读,您可能已经注意到我们使用简写符号来构建`Complex`值.使用和字段相同名称的变量初始化结构的字段是很常见的，而不是强迫你编写`Complex { re: re, im: im }`,Rust允许你简写`Complex { Re, im }`.这是用JavaScript和Haskell中的类似符号建模的.

### 从像素到复数的映射

程序需要在两个相关的坐标空间中工作:输出图像中的每个像素对应于复平面上的一个点. 这两个空间之间的关系取决于我们要绘制Mandelbrot集合的哪个部分,以及所需要的图像的分辨率,这由命令行参数确定.下面的函数从图像空间转换为复数空间:

```Rust
/// Given the row and column of a pixel in the output image, return the
/// corresponding point on the complex plane.
///
/// `bounds` is a pair giving the width and height of the image in pixels.
/// `pixel` is a (column, row) pair indicating a particular pixel in that image.
/// The `upper_left` and `lower_right` parameters are points on the complex
/// plane designating the area our image covers.
fn pixel_to_point(bounds: (usize, usize),
                pixel: (usize, usize),
                upper_left: Complex<f64>,lower_right: Complex<f64>)
    -> Complex<f64>
{
    let (width, height) = (lower_right.re - upper_left.re,
                        upper_left.im - lower_right.im);
    Complex {
        re: upper_left.re + pixel.0 as f64 * width  / bounds.0 as f64,
        im: upper_left.im - pixel.1 as f64 * height / bounds.1 as f64
        // Why subtraction here? pixel.1 increases as we go down,
        // but the imaginary component increases as we go up.
    }
}

#[test]
fn test_pixel_to_point() {
    assert_eq!(pixel_to_point((100, 100), (25, 75),
                            Complex { re: -1.0, im:  1.0 },
                            Complex { re:  1.0, im: -1.0 }),
            Complex { re: -0.5, im: -0.5 });
}
```

图2-4说明了`pixel_to_point`执行的计算.

*图2-4. 复平面和图像像素之间的关系*

`pixel_to_point`的代码只是计算,所以我们不会详细解释.但是,有几点需要指出.用这种形式的表达式引用元组元素:

```Rust
pixel.0
```

这是引用元组`pixel`的第一个元素.

```Rust
pixel.0 as f64
```

这是Rust的类型转换语法:转换`pixel.0`到一个`f64`值.和C以及C++不同,Rust通常拒绝隐式地在数字类型之间进行转换;你必须写出你需要的转换.这可能很乏味，但是明确哪些转换发生以及何时发生是非常有用的.隐式整数转换似乎是无辜的,但从历史上看,它们是真实世界的C和c++代码中经常出现的bug和安全漏洞的来源.

### 绘制集合

要绘制Mandelbrot集,对于图像中的每个像素,我们只需将`escape_time`应用于复平面上的对应点,并根据结果为像素着色:

```Rust
/// Render a rectangle of the Mandelbrot set into a buffer of pixels.
///
/// The `bounds` argument gives the width and height of the buffer `pixels`,
/// which holds one grayscale pixel per byte. The `upper_left` and `lower_right`
/// arguments specify points on the complex plane corresponding to the upper-
/// left and lower-right corners of the pixel buffer.
fn render(pixels: &mut [u8],
        bounds: (usize, usize),
        upper_left: Complex<f64>,
        lower_right: Complex<f64>)
{
    assert!(pixels.len() == bounds.0 * bounds.1);
    for row in 0 .. bounds.1 {
        for column in 0 .. bounds.0 {
            let point = pixel_to_point(bounds, (column, row),
                                    upper_left, lower_right);
            pixels[row * bounds.0 + column] =
                match escape_time(point, 255) {
                    None => 0,
                    Some(count) => 255 - count as u8
                };
        }
    }
}
```

此时这一切看起来都很熟悉.

```Rust
pixels[row * bounds.0 + column] =
    match escape_time(point, 255) {
        None => 0,
        Some(count) => 255 - count as u8
    };
```

如果`escope_time`表示`point`属于集合,则`render`着色相应的像素为黑色(`0`).否则,`render`会为需要更长时间才能逃脱圆圈的数字指定较暗的颜色.

### 写入图像文件

`image`crate提供了用于读取和写入各种图像格式的函数,以及一些基本的图像处理函数.特别是,它包括PNG图像文件格式的编码器,程序使用该编码器来保存最终的计算结果.要使用`image`,请将以下行添加到 *Cargo.toml* 的`[dependencies]`部分:

```Rust
image = "0.13.0"
```

有了这个,我们可以写:

```Rust
extern crate image;
use image::ColorType;
use image::png::PNGEncoder;
use std::fs::File;

/// Write the buffer `pixels`, whose dimensions are given by `bounds`, to the
/// file named `filename`.
fn write_image(filename: &str, pixels: &[8], bounds: (usize, usize))
    -> Result<(), std::io::Error>
{
    let output = File::crate(filename)?;

    let encoder = PNGEncoder::new(output);
    encoder.encode(&pixels,
                bounds.0 as u32, bound.1 as u32,
                ColorType::Gray(8))?;
    Ok(())
}
```

这个函数的操作非常简单:它打开一个文件并尝试将图像写入其中,我们向编码器传递来自`pixels`的实际像素数据,以及来自`bounds`的像素宽度和高度,然后最后一个参数说明如何解释`pixels`中的字节:值`ColorType::Gray(8)`表示每个字节都是一个8位的灰度值.

这一切都很简单.这个函数的有趣之处在于它在出错时是如何处理的.如果遇到错误,我们需要将其报告给调用者.正如我们之前提到的,Rust中的易出错函数应该返回一个`Result`值,该值在成功时是`Ok(s)`,其中`s`是成功的值,在失败时为`Err(e)`,其中`e`是错误码.那么`write_image`的成功和错误类型是什么呢?

当一切顺利时,`write_image`函数没有有用的值可以返回;它把所有有趣的东西都写入了文件.因此,它的成功类型是 *单元(unit)* 类型`()`,之所以这么叫是因为它只有一个值,也就是`()`.单元类型类似于C和c++中的`void`.

当发生错误时,这是因为`File::create`无法创建文件,或者`encoder.encode`无法将图像写入其中;I/O操作返回错误代码.`File::create`的返回类型是`Result<std::fs::File, std::io::Error>`,而`encoder.encode`的返回类型是`Result<(), std::io::Error>`,所以两者共享相同的错误类型,`std::io::Error`.我们的write_image函数也可以这样做.

考虑对`File::create`的调用,如果对于成功打开的`File`值`f`返回`Ok(f)`,则`write_image`可以继续向`f`写入图像数据.但是如果`File::create`错误代码`e`的`Err(e)`,则`write_image`应立即返回`Err(e)`作为它自己的返回值.对`encoder.encode`的调用必须以类似地处理:失败应该导致立即返回,并传递错误代码.

`?`运算符的存在使这些检查变得方便.而不是拼写出所有内容,并写下：

```Rust
let output = match File::create(filename) {
    Ok(f) => { f }
    Err(e) => { return Err(e); }
};
```

你可以使用等效的,更清晰的:

```Rust
let output = File::create(filename)?;
```

> **注意**
> 尝试在`main`函数中使用`?`是一个常见的初学者错误.然而,由于`main`没有返回值,所以这不起作用;你应该使用`Result`的`expect`方法.`?`运算符仅在返回`Result`的函数中有用.

我们可以在这里使用另一种速记.因为某些类型`T`的`Result<T, std::io::Error>`形式的返回类型非常常见--这通常是执行I/O的函数的正确类型--Rust标准库为它定义了一个简写.在`std::io`模块中,我们有以下定义:

```Rust
// The std::io::Error type.
struct Error { ... };
// The std::io::Result type, equivalent to the usual `Result`, but
// specialized to use std::io::Error as the error type.
type Result<T> = std::result::Result<T, Error>
```

如果我们使用`std::io::Result`声明将此定义带入范围,我们可以更简洁地将`write_image`的返回类型写为`Result<()>`.这是你在阅读`std::io`,`std::fs`和其他地方的函数文档时经常看到的形式.

### 并行Mandelbrot程序

最后,所有部分都已就绪,我们可以向你展示下`main`主函数,在这里我们可以将并发性用于工作.首先,为了简单起见,一个非并发版本:

```Rust
use std::io::Write;
fn main() {
    let args: Vec<String> = std::env::args().collect();
    if args.len() != 5 {
        writeln!(std::io::stderr(),
                "Usage: mandelbrot FILE PIXELS UPPERLEFT LOWERRIGHT")
            .unwrap();
        writeln!(std::io::stderr(),
                "Example: {} mandel.png 1000x750 -1.20,0.35 -1,0.20",
                args[0])
            .unwrap();
        std::process::exit(1);
    }
    let bounds = parse_pair(&args[2], 'x')
        .expect("error parsing image dimensions");
    let upper_left = parse_complex(&args[3])
        .expect("error parsing upper left corner point");
    let lower_right = parse_complex(&args[4])
        .expect("error parsing lower right corner point");

    let mut pixels = vec![0; bounds.0 * bounds.1];

    render(&mut pixels, bounds, upper_left, lower_right);

    write_image(&args[1], &pixels, bounds)
        .expect("error writing PNG file");
}
```

在将命令行参数收集到`String`向量中之后,我们解析每个参数然后开始计算.

```Rust
let mut pixels = vec![0; bounds.0 * bounds.1];
```

宏调用`vec![v; n]`创建一个`n`个元素长的向量,其元素初始化为`v`,因此前面的代码创建一个零的向量,其长度为`bounds.0 * bounds.1`,其中`bounds`是从命令行解析的图像分辨率.我们将此向量用作单字节灰度像素值的矩形阵列,如图2-5所示.

*图2-5. 使用向量作为像素的矩形阵列*

下一行感兴趣的是:

```Rust
render(&mut pixels, bounds, upper_left, lower_right);
```

这会调用`render`函数来实际计算图像.表达式`&mut pixels`借用对我们的像素缓冲区的可变引用,允许`render`用计算出的灰度值填充它,即使`pixels`仍然是向量的所有者.其余参数传递图像的尺寸,以及我们选择绘制的复平面的矩形.

```Rust
write_image(&args[1], &pixels, bounds)
        .expect("error writing PNG file");
```

最后,我们将像素缓冲区,作为PNG文件写入磁盘.在这种情况下,我们传递一个缓冲区的共享(不可变)引用,因为`write_image`不需要修改缓冲区的内容.

将这种计算分配到多个处理器上的自然方法是将图像分成几个部分,每个处理器一个,并让每个处理器为分配给它的像素着色.为简单起见,我们将其分解为水平条带,如图2-6所示.当所有处理器完成后,我们可以将像素写出到磁盘.

*图2-6.为并行渲染将像素缓冲区划分为条带*

`crosssbeam`crate提供了许多有价值的并发工具,包括一个 *scoped thread* 工具,它完全满足我们这里的需要.要使用它,我们必须将以下行添加到我们的 *Cargo.toml* 文件中:

```Toml
crossbean = "0.2.8"
```

然后,我们必须将以下行添加到`main.rs`文件的顶部:

```Rust
extern crate crossbeam
```

然后我们需要取出调用`render`的这一行,并用以下内容替换它:

```Rust
let threads = 8;
let rows_per_band = bounds.1 / threads + 1;

{
    let bands: Vec<&mut [u8]> =
        pixels.chunks_mut(rows_per_band * bounds.0).collect();
    crossbeam::scope(|spawner| {
        for (i, band) in bands.into_iter().enumerate() {
            let top = rows_per_band * i;
            let height = band.len() / bounds.0;
            let band_bounds = (bounds.0, height);
            let band_upper_left =
                pixel_to_point(bounds, (0, top), upper_left, lower_right);
            let band_lower_right =
                pixel_to_point(bounds, (bounds.0, top + height),
                            upper_left, lower_right);
            spawner.spawn(move || {
                render(band, band_bounds, band_upper_left, band_lower_right);
            });
        }
    });
}
```

按照通常的方式进行分解:

```Rust
let threads = 8;
let rows_per_hand = bounds.1 / threads + 1
```

这里我们决定使用八个线程.^[1] 然后我们计算每个条带应该有多少行像素.由于带的高度是`rows_per_band`并且图像的总宽度是`bounds.0`.因此带的面积(以像素为单位)是`rows_per_band * bounds.0`.我们向上四舍五入计算行数,以确保条带覆盖整个图像,即使高度不是线程的倍数.

```Rust
let bands: Vec<&mut [u8]> =
        pixels.chunks_mut(rows_per_band * bounds.0).collect();
```

这里我们将像素缓冲区划分为带.缓冲区的`chunks_mut`方法返回一个迭代器,该迭代器生成缓冲区的可变的,不重叠的切片,每个切片都包含`rows_per_band * bounds.0`个像素--换句话说,`rows_per_band`完整的像素行.`chunks_mut`生成的最后一个切片可能包含较少的行,但是每行将包含相同数量的像素.最后,迭代器的`collect`方法构建一个向量来保存这些可变的,不重叠的切片.

现在我们可以让`crossbeam`库工作:

```Rust
crossbeam::scope(|spawner| { ... });
```

参数`|spawner| { ... }`是Rust的 *闭包(closure)* 表达式.闭包是一个可以被调用的值,就像它是一个函数一样.这里,`|spawner|`是参数列表,`{ ... }`是函数体.注意,与用`fn`声明的函数不同,我们不需要声明闭包的参数类型;Rust会推断它们以及它的返回类型.

在这种情况下,`crossbeam::scope`调用闭包,传递给`spawner`参数一个值,闭包可用其创建新线程.`crossbeam :: scope`函数在返回之前等待所有此类线程完成执行.这种行为允许Rust确保这些线程在超出作用域后不会访问它们的像素部分,并允许我们确保当`crossbeam::scope`返回时,图像计算完成.

```Rust
for (i, band) in bands.into_iter().enumerate() {
```

在这里,我们遍历像素缓冲区的条带.`into_iter()`迭代器为循环体的每次迭代提供一个带的独占所有权,确保每次只能有一个线程对其进行写入.我们将在第5章详细解释它的工作原理.然后,枚举适配器生成元组,将每个向量元素与其索引配对.

```Rust
let top = rows_per_band * i;
let height = band.len() / bounds.0;
let band_bounds = (bounds.0, height);
let band_upper_left =
    pixel_to_point(bounds, (0, top), upper_left, lower_right);
let band_lower_right =
    pixel_to_point(bounds, (bounds.0, top + height),
                upper_left, lower_right);
```

给定索引和带的实际大小(回想一下最后一个可能比其它的短),我们可以生成一个`render`需要的包围框,但只能引用缓冲区的这个带,而不是整个图像.类似地,我们重新调整渲染器的`pixel_to_point`函数,以找到带的左上角和右下角落在复平面上的位置.

```Rust
spawner.spawn(move || {
    render(band, band_bounds, band_upper_left, band_lower_right);
});
```

最后,我们创建了一个线程,运行闭包`move || { ... }`.这个语法看起来有点奇怪:它表示一个没有参数的闭包,其主体是`{ ... }`形式.前面的`move`关键字表示这个闭包拥有它使用的变量的所有权;特别是,只有闭包可以使用可变切片`band`.

正如我们前面提到的,`crossbeam::scope`调用确保所有线程在它返回之前都已完成,这意味着将图像保存到文件是安全的,这是我们的下一步操作.

### 运行Mandelbrot绘制器

我们在这个程序中使用了几个外部crates:`num`用于复数运算;`image`用于编写PNG文件;和`crossbeam`用于scoped thread创建原语.这是最终的 *Cargo.toml* 文件,包括所有这些依赖项:

```Toml
[package]
name = "mandelbrot"
version = "0.1.0"
authors = ["You <you@example.com>"]

[dependencies]
crossbeam = "0.2.8"
image = "0.13.0"
num = "0.1.27"
```

有了这个,我们就可以构建并运行该程序:

```Shell
$ cargo build --release
    Updating registry `https://github.com/rust-lang/crates.io-index`
   Compiling bitflags v0.3.3
   ...
   Compiling png v0.4.3
   Compiling image v0.13.0
   Compiling mandelbrot v0.1.0 (file:///home/jimb/rust/mandelbrot)
    Finished release [optimized] target(s) in 42.64 secs
$ time 
target/release/mandelbrot mandel.png 4000x3000 -1.20,0.35 -1,0.20
real    0m1.750s
user    0m6.205s
sys     0m0.026s
$
```

在这里,我们使用Unix`time`程序来查看程序运行了多长时间;请注意,即使我们花费六秒多的处理器时间来计算图像,但实际经过的时间还不到两秒钟.你可以通过注释掉执行此操作的代码来验证该真正时间的大部分内容是否用于编写图像文件;在测试此代码的笔记本电脑上,并发版本将Mandelbrot计算时间缩短了近四倍.我们将在第19章中展示如何在此基础上进一步改进.

此命令应该创建一个名为 *mandel.png* 的文件,你可以使用系统的图像查看程序或在Web浏览器中查看该文件.如果一切顺利,它应该如图2-7所示.

*图2-7. 并行Mandelbrot程序的结果*

### 安全是无形的

最后,我们最终得到的并行程序与我们用任何其它语言编写的程序没有太大的不同:我们在将像素缓冲区的小块分配给各个处理器;让每个分开工作;当他们全部完成时,呈现结果。那么Rust的并发支持有什么特别之处呢?

我们这里没有展示的是所有我们无法编写的Rust程序.我们在本章中看到的代码正确地将缓冲区划分到线程之间,但是该代码有许多小的变体,它们并不正确(因此引入了数据竞争);这些变体中没有一个能通过Rust编译器的静态检查.C或C++编译器会愉快地帮助你探索具有微妙数据竞争的程序的广阔空间;而Rust预先告诉你什么时候会出问题.

在第4章和第5章中,我们将描述Rust的内存安全规则.第19章解释了这些规则如何确保适当的并发卫生.但是为了理解这些,在有必要对Rust的基本类型有所了解,我们将在下一章中介绍.

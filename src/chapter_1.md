# 第一章 基础  

当你深入到 Rust 的更高级的角落时，重要的是你要确保你对基础知识有一个坚实的理解。在 Rust 中，就像任何编程语言一样，当你开始以更复杂的方式使用该语言时，各种关键字和概念的确切含义变得非常重要。在本章中，我们将浏览 Rust 的许多基元，并试图更清楚地定义它们的含义，它们是如何工作的，以及为什么它们是这样的。具体来说，我们将看看变量和值有什么不同，它们在内存中是如何表示的，以及一个程序有哪些不同的内存区域。然后，我们将讨论一些所有权、借用和寿命的微妙之处，在你继续阅读本书之前，你需要掌握这些知识。  

如果你愿意，你可以从头到尾地阅读这一章，也可以把它作为参考，来复习那些你觉得不太确定的概念。我建议你只有在对本章的内容感到完全满意时才继续阅读，因为对这些基元如何工作的误解会很快妨碍你理解更高级的主题，或者导致你错误地使用它们。  

## 谈谈内存  

不是所有的内存都是平等的。在大多数编程环境中，你的程序可以访问栈（stack）、堆（heap）、寄存器（register）、文本段（text segment）、内存映射的寄存器（memory-mapped register）、内存映射的文件（memory-mapped file），也许还有非易失性 RAM（Nonvolatile RAM）。在特定情况下，你选择使用哪一个，对你能在那里存储什么，它能保持多长时间，以及你用什么机制来访问它都有影响。这些内存区域的具体细节因平台而异，超出了本书的范围，但有些内存区域对你如何推理 Rust 代码非常重要，因此值得在此介绍。  

### 内存术语  

在我们深入研究内存区域之前，你首先需要了解值、变量和指针之间的区别。Rust 中的值是一个类型和该类型的值域的一个元素的组合。一个值可以使用其类型的表示法变成一串字节，但就其本身而言，你可以认为一个值更像是你这个程序员的意思。例如，`u8` 类型中的数字 `6` 是数学整数 `6` 的一个实例，它在内存中的表示是字节 `0x06`。同样，字符串 "Hello world" 是所有字符串域中的一个值，其表示方法是 UTF-8 编码。一个值的意义与这些字节存储的位置无关。  

一个值被存储在一个地方，这是 Rust 的术语，意思是 "一个可以容纳一个值的位置"。这个位置可以在栈中，也可以在堆上，或者在其他一些位置。最常见的存储值的地方是一个变量，它是栈上的一个命名值槽。

指针是一个持有内存区域地址的数值，所以指针指向一个地方。指针可以被解引用，以访问存储在它所指向的内存位置的值。我们可以在一个以上的变量中存储同一个指针，因此有多个变量间接地指向内存中的同一个位置，从而指向同一个底层值。

考虑清单 1-1 中的代码，它说明了这三个要素。

```rust
let x = 42;
let y = 43;
let var1 = &x;
let mut var2 = &x;
var2 = &y; // (1)

// 清单 1-1：值、变量和指针
```

这里，有四个不同的值。`42`（一个`i32`），`43`（一个`i32`），`x` 的地址（一个指针），以及 `y` 的地址（一个指针）。还有四个变量：`x`、`y`、`var1` 和 `var2`。后两个变量都持有指针类型的值，因为引用是指针。虽然 `var1` 和 `var2` 最初存储的是同一个值，但它们分别存储该值的独立副本；当我们改变 `var2` (1) 中存储的值时，`var1` 中的值不会改变。特别是，`=` 运算符将右侧表达式的值存储在左侧命名的地方。

变量、值和指针之间的区别变得很重要的一个有趣的例子是在一个语句中，比如说：

```rust
let string = "Hello world";
```

尽管我们给变量 `string` 分配了一个字符串值，但该变量的实际值是一个指向字符串值 "Hello world "中第一个字符的指针，而不是字符串值本身。在这一点上，你可能会说："但是等一下，那么字符串的值是在哪里存储的？指针指向哪里？" 如果是这样的话，你的眼光就很敏锐了--我们一会儿就会说到这一点。

### 深入了解变量

我前面给出的变量定义很宽泛，本身不太可能有什么用。当你遇到更复杂的代码时，你将需要一个更准确的心智模型来帮助你推理出程序的真正作用。我们可以利用许多这样的模型。详细描述它们会占用好几章的篇幅，也超出了本书的范围，但广义上讲，它们可以分为两类：高层模型（High-level）和低层模型（low-level）。高层模型在思考生存期和借用层面的代码时很有用，而低层模型在推理不安全代码和原始指针时很有用。下面两节描述的变量模型对于本书的大部分材料来说已经足够了。

#### 高层模型

在高级模型中，我们不认为变量是存放字节的地方。当你给一个变量赋值的时候，这个值就被这个变量命名了。当一个变量后来被访问时，你可以想象从该变量的前一次访问到新的访问画一条线，这在两次访问之间建立了一种依赖关系。如果一个变量中的值被移动了，就不能再从它那里画线了。

在这个模型中，一个变量只有在它持有一个合法的值时才存在；你不能从一个值未被初始化或已被移动的变量上画线，所以实际上它不存在。使用这个模型，你的整个程序由许多这样的依赖线组成，通常称为流，每个流都追踪一个值的特定实例的生存期。当有分支时，流可以分叉和合并，每一个分叉都追踪该值的一个不同的生存期。编译器可以检查在程序的任何给定点，所有可以相互平行存在的流都是兼容的。例如，不能有两个并行的流对一个值进行可变的访问。也不能有一个流借用一个值，而没有一个流拥有该值。清单 1-2 显示了这两种情况的例子。

```rust
let mut x;
// 这是非法的，没有地方可以获取流
// assert_eq!(x, 42);
x = 42;     // (1)
// 这是好的，可以从上面分配的值中画出一个流程。
let y = &x; // (2)
// 这就建立了第二个来自 x 的、可变的流。
x = 43;     // (3)
// 这样就继续从 y 那里获得流，而 y 又从 x 那里获得流。 
// 但这条流与对 x 的分配相冲突！
assert_eq!(*y, 42); // (4)

// 清单 1-2：借用检查器会发现的非法流
```

首先，在 `x` 被初始化之前，我们不能使用它，因为我们没有地方可以绘制流。只有当我们给 `x` 赋值时，我们才能从它那里提取流。这段代码有两个流：一个从 `(1)` 到 `(3)` 的独占（`&mut`）流，一个从 `(1)` 到 `(2)` 到 `(4)` 的共享（`&`）流。 借阅检查器检查每个流的每个顶点，并检查是否有其他不兼容的流同时存在。在这个例子中，当借用检查器检查 `(3)` 处的独占流时，它看到了终止于 `(4)` 处的共享流。由于你不能同时对一个值进行独占和共享使用，借用检查器（正确地）拒绝了该代码。请注意，如果没有 `(4)`，这段代码会编译得很好。共享流将在 `(2)` 处终止，而当独占流在 `(3)` 处被检查时，就不会有冲突的流存在。

如果一个新的变量与之前的变量同名，它们仍然被认为是不同的变量。这被称为 "遮蔽"--后一个变量 "遮蔽"了前一个同名的变量。这两个变量共存，尽管随后的代码不再有办法命名先前的变量。这个模型与编译器，特别是借用检查器，对你的程序的推理大致吻合，并且实际上在编译器的内部使用，以产生高效代码。

#### 底层模型

变量命名了可能持有或不持有合法数值的内存位置。你可以把一个变量看作是一个 "值槽"。当你给它赋值时，槽被填满，它的旧值（如果它有一个的话）被丢弃和替换。当你访问它时，编译器会检查该槽是否为空，因为这意味着该变量未被初始化或其值已被移动。一个变量的指针指的是该变量的后备内存，可以被解引用以获得其值。例如，在语句 `let x: usize` 中，变量 `x` 是堆栈上一个内存区域的名称，该区域有空间容纳一个 `usize` 大小的值，尽管它没有一个明确的值（其槽是空的）。如果你给这个变量赋值，比如 `x = 6`，那么这个内存区域就会容纳代表值 `6` 的比特。这个模型与 C 和 C++以及其他许多低级语言所使用的内存模型相匹配，当你需要对内存进行明确的推理时，这个模型很有用。

> 注意：在这个例子中，我们忽略了 CPU 寄存器，并将其视为一种优化。在现实中，如果一个变量不需要内存地址，编译器可能会使用一个寄存器来支持该变量，而不是一个内存区域。

你可能会发现其中一个比另一个更适合你之前的模型，但我建议你试着仔细理解这两个模型。它们都是同样有效的，而且都是简化的，就像任何有用的心智模型一样。如果您能够从这两个角度考虑一段代码，您就会发现处理复杂的代码段并理解它们为什么要或不按照预期进行编译和工作要容易得多。

### 内存区域

现在你已经掌握了我们对内存的称呼，我们需要谈谈内存到底是什么。内存有许多不同的区域，也许令人惊讶的是，并不是所有的区域都存储在你的计算机的 DRAM 中。你使用哪一部分内存，对你如何编写代码有很大影响。就编写 Rust 代码而言，三个最重要的区域是栈、堆和静态内存。

#### 栈

栈是一段内存，程序使用它作为函数调用的临时空间。每次调用一个函数时，都会在栈的顶部分配一个称为帧的连续内存块。靠近栈底部的是主函数的帧，当函数调用其他函数时，额外的帧被推送到栈中。函数的帧包含该函数中的所有变量，以及该函数接受的任何参数。当函数返回时，它的栈帧被回收。

构成函数局部变量值的字节不会立即被清除，但访问它们是不安全的，因为它们可能被随后的函数调用覆盖，而该函数调用的帧与回收的帧重叠。即使它们没有被覆盖，它们也可能包含非法使用的值，例如在函数返回时被移动的值。

栈帧，以及它们最终消失的关键事实，与 Rust 中的生存期概念密切相关。任何存储在栈上的帧中的变量在该帧消失后都不能被访问，所以任何对它的引用都必须有一个最长与帧生存期一样长的生存期。

#### 堆

堆是一个内存池，它没有绑定到程序的当前调用栈。堆内存中的值会一直存在，直到它们被显式地回收。当您希望一个值存在超过当前函数帧的生存期时，这是很有用的。如果该值是函数的返回值，则调用函数可以在其栈上留下一些空间，以便被调用函数在返回之前将该值写入其中。但是，如果你想，比如说，发送这个值到一个不同的线程，而当前线程可能根本不共享栈帧，你可以把它存储在堆上。

堆允许您显式地分配连续的内存段。当你这样做的时候，你会得到一个指向该内存段开始的指针。这个内存段是为你保留的，直到你以后释放它；这个过程通常被称为释放，以 C 标准库中相应函数的名称命名。由于函数返回时 heap 的分配不会消失，所以您可以在一个位置为一个值分配内存，将指向它的指针传递给另一个线程，并让该线程安全地继续操作该值。或者，换句话说，当你堆分配内存时，结果指针有一个不受约束的生存期——它的生存期是你的程序保持它存活的时间。

在 Rust 中与堆交互的主要机制是 `Box` 类型。当您写入 `Box::new(value)` 时，该值被放在堆上，而返回给您的 (`Box<T>`) 是一个指向堆上该值的指针。当 `Box` 最终被丢弃时，内存将被释放。

如果你忘记删除堆内存，它将永远存在，你的应用程序最终会吃掉你机器上的所有内存。这被称为泄漏内存，通常是你想要避免的。然而，在有些情况下，你会明确地想要泄漏内存。例如，假设你有一个只读的配置，整个程序都应该能够访问。你可以在堆上分配这个配置，然后用 `Box::leak` 显式地泄露它，以获得一个 "静态引用"。

#### 静态内存

静态内存实际上是一个包揽一切的术语，指的是程序被编译成的文件中几个密切相关的区域。当程序执行时，这些区域会自动加载到程序的内存中。静态内存中的值在程序执行的整个过程中都是有效的。你的程序的静态内存包含程序的二进制代码，它通常被映射为只读。当您的程序执行时，它将遍历文本段指令中的二进制代码，并在调用函数时跳转。静态内存还保存了使用`static` 关键字声明的变量的内存，以及代码中的某些常量值，比如字符串。

特殊的生存期 `'static`，它的名称来自静态内存区域，标志着一个引用在静态内存存在的时间内是有效的，也就是直到程序关闭。因为静态变量的内存是在程序启动时分配的，所以对静态内存中变量的引用定义为 `'static`，因为在程序关闭之前它不会被释放。反之则不然，可能会有不指向静态内存的 `'static` 引用，但这个名字仍然是合适的：一旦你创建了一个具有静态寿命的引用，就程序的其他部分而言，它所指向的东西可能就在静态内存中，因为它可以被使用多长时间，你的程序就会使用多长时间。

在使用 Rust 时，您将更经常地遇到 `'static`生存期，而不是真正的静态内存（例如，通过 `static` 关键字）。这是因为 `static` 经常出现在类型参数的 trait 限定中。像 `T: 'static` 这样的绑定表示类型参数 `T` 能够存活，我们就保留它多久，包括程序的剩余执行时间。本质上，这个限定要求 `T` 是拥有（owned）的和自给自足（self-sufficient）的，要么它不借用其他（非静态）值，要么它借用的任何东西也是 `'static`，因此会一直保留到程序结束。`'static` 作为限定的一个很好的例子是 `std::thread::spawn` 函数，它创建了一个新的线程，它要求传递给它的闭包是 `'static` 。由于新线程的生存期可能比当前线程长，因此新线程不能引用存储在旧线程堆栈上的任何内容。新线程只能引用在其整个生存期内（可能是在程序的剩余时间内）存在的值。

> 注意：您可能想知道 `const` 与 `static` 有何不同。 `const` 关键字将下面的项声明为常量。可以在编译时完全计算常数项，任何引用它们的代码都将在编译期间替换为常数的计算值。常量没有与之关联的内存或其他存储（它不是一个位置）。您可以将常量看作是特定值的一个方便名称。

## 所有权

Rust 的内存模型的核心思想是，所有值都有一个所有者，也就是说，只有一个位置（通常是一个作用域）负责最终回收每个值。这是通过借用检查器强制执行的。如果值被移动，例如将其赋值给一个新变量、将其推入一个向量或将其放在堆上，则值的所有权将从旧位置移动到新位置。在这一点上，您不能再通过来自原始所有者的变量访问值，即使从技术上来说，组成值的位仍然存在。相反，您必须通过引用其新位置的变量来访问被移动的值。

有些类型是反叛者，不遵守这条规则。如果一个值的类型实现了特殊的 `Copy` 特性，即使它被重新分配到一个新的内存位置，它也不会被认为已经移动了。相反，该值被复制，新旧位置仍然可访问。从本质上说，在移动的目的地构造了另一个相同值的相同实例。Rust 中大多数基本类型（如整数和浮点类型）都是 `Copy` 类型。要成为 Copy 类型，必须能够简单地通过复制其比特来复制该类型的值。这排除了所有包含非 `Copy` 类型的类型，以及在值被删除时它必须释放的拥有资源的任何类型。

要知道为什么，请考虑一下如果像 `Box` 这样的类型被复制会发生什么。如果我们执行 `box2 = box1`，那么 `box1` 和 `box2` 都会认为他们拥有为 `box` 分配的堆内存，当他们超出范围时，他们都会试图释放它。释放两次内存可能会产生灾难性的后果。

当一个值的所有者不再使用它时，该所有者有责任通过删除该值来对该值进行任何必要的清理。在 Rust 中，当保存值的变量不再在作用域中时，删除将自动发生。类型通常递归地删除它们所包含的值，因此删除复杂类型的变量可能会导致许多值被删除。由于 Rust 的离散所有权要求，我们不能意外地多次丢弃相同的价值。保存对另一个值的引用的变量并不拥有另一个值，因此当变量删除时，该值不会被删除。

清单 1-3 中的代码给出了围绕所有权、移动和复制语义以及放弃的规则的快速总结。

```rust
let x1 = 42;
let y1 = Box::new(84);
{ // 开始一个新的作用域
let z = (x1, y1); // (1)
// z 离开作用域，并被析构；
// 它一次析构了 x1 和 y1 中的值
} // (2)
// x1 的值是 Copy 语义， 所以它不会移动给 z
let x2 = x1; // (3)
// y1 的值不是 Copy 语义，所以它会移动给 z
// let y2 = y1; // (4)

// 清单 1-3: 移动和复制语义
```

我们从两个值开始，数字 `42` 和包含数字 `84` 的 `Box（一个堆分配的值）。前者是复制，而后者不是。当我们将` `x1` 和 `y1` 放入元组 `z1` `时，x1` 被复制到 `z` 中，而 `y1` 被移动到 `z` 中。此时，`x1` 继续可访问，并可以再次使用 (3) 。另一方面，一旦 `y1` 的值被移动到 (4)，它就变得不可访问，任何访问它的尝试都将导致编译器错误。当 `z` 超出范围 (2) 时，它所包含的元组值将被删除，这将依次删除从 `x1` 复制的值和从 `y1` 移动的值。当 `y1` 中的 `Box` 被丢弃时，它还释放用于存储 `y1` 值的堆内存。

>析构顺序
>
>当值超出作用域时，Rust 会自动丢弃它们，比如清单 1-3 中内部作用域的 x1 和 y1。丢弃顺序的规则相当简单：变量（包括函数参数）按相反的顺序丢弃，嵌套值按源代码的顺序丢弃。
>
>这听起来可能很奇怪，为什么会有这样的差异？不过，如果我们仔细观察，就会发现它有很大的意义。假设你写了一个函数，声明了一个字符串，然后将该字符串的引用插入到一个新的哈希表中。当函数返回时，哈希表必须先被删除；如果字符串先被删除，那么哈希表就会持有一个无效的引用 一般来说，后来的变量可能包含对早期值的引用，而由于 Rust 的生存期规则，反之则不能发生。出于这个原因，Rust 以相反的顺序丢弃变量。
>
>现在，我们可以对嵌套的值有同样的行为，比如元组、数组或结构中的值，但这可能会让用户感到惊讶。如果你构建了一个包含两个值的数组，如果数组的最后一个元素先被丢弃，那就显得很奇怪。这同样适用于元组和结构，最直观的行为是第一个元组元素或字段先被丢弃，然后是第二个，以此类推。与变量不同的是，在这种情况下没有必要颠倒丢弃顺序，因为 Rust（目前）不允许在单个值中进行自我引用。所以，Rust 采用了直观的选项。

## 借用和生存期

Rust 允许一个值的所有者通过引用将该值借给其他人，而不放弃所有权。引用是一个指针，它有一个额外的契约，规定了它们的使用方式，比如引用是否提供了对被引用值的唯一访问，或者被引用值是否也可以有其他引用指向它。

### 共享引用

顾名思义，是一个可以被共享的指针。对于相同的值，可以存在任意数量的其他引用，并且每个共享引用都是 `Copy`，因此您可以轻松地创建更多的共享引用。共享引用背后的值不是可变的；您不能修改或重新分配共享引用指向的值，也不能将共享引用强制转换为可变引用。

Rust 编译器允许假设共享引用指向的值在引用存在期间不会改变。例如，如果 Rust 编译器看到共享引用后面的值在函数中被多次读取，那么它有权只读取一次并重用该值。更具体地说，清单 1-4 中的断言应该永远不会失败。

```rust
fn cache(input: &i32, sum: &mut i32) {
    *sum = *input + *input;
    assert_eq!(*sum, 2 * *input);
} 

// 清单 1-4: Rust 假设共享引用是不可变的。
```

编译器是否选择应用给定的优化或多或少是无关的。编译器的启发式会随着时间的推移而改变，所以你通常想要根据编译器被允许做什么来编写代码，而不是根据它在特定的情况下在特定的时间点实际做了什么。

### 可变引用

共享引用的替代方案是可变引用：&mut T。对于可变引用，Rust 编译器又被允许充分利用引用所带来的契约：编译器假设没有其他线程访问目标值，无论是通过共享引用还是可变引用。换句话说，它假定可变引用是独占的。这使得一些有趣的优化成为可能，这些优化在其他语言中是不容易实现的。以清单 1-5 中的代码为例。

```rust
fn noalias(input: &i32, output: &mut i32) {
    if *input == 1 {
        *output = 2; // (1)
    } if *input != 1 {  // (2)
        *output = 3;
    }
}

// 清单 1-5:  Rust 假设可变借用是独占的
```

在 Rust 中，编译器可以假设输入和输出不指向同一内存。因此，(1) 处输出的重新分配不能影响 (2) 处的检查，整个函数可以被编译为一个单一的 `if-else` 块。如果编译器不能依赖排他性可变性契约，那么这种优化就会失效，因为在 `noalias(&x, &mut x)` 这样的情况下，(1) 的输入可能导致 (3) 的输出。

一个可改变的引用只允许你改变该引用所指向的内存位置。你是否可以改变直接引用之外的值，取决于位于两者之间的类型所提供的方法。用一个例子可能更容易理解，所以考虑清单 1-6。

```rust
let x = 42;
let mut y = &x; // y &i32 类型
let z = &mut y; // z 是 &mut &i32 类型

// 清单 2-6: 可变性只适用于直接引用的内存
```

在这个例子中，你能够通过使指针 `y` 引用不同的变量来改变它的值（也就是不同的指针），但你不能改变被指向的值（也就是 `x` 的值）。同样地，你可以通过 `z` 来改变 `y` 的指针值，但你不能改变 `z` 本身，使其持有不同的引用。

拥有一个值和拥有一个对它的可变引用之间的主要区别是，当不再需要这个值时，所有者要负责丢弃这个值。除此之外，你可以通过一个可改变的引用做任何事情，如果你拥有这个值的话，有一个注意事项：如果你把这个值移到可改变的引用后面，那么你必须在它的位置上留下另一个值。如果你不这样做，所有者仍然会认为它需要放弃这个值，但是没有任何值可以让它放弃。

清单 1-7 给出了将值移动到可变引用后面的方法示例。

```rust
fn replace_with_84(s: &mut Box<i32>) {
    // 这是不可能的，因为 *s 会变成空值 :
    // let was = *s; // (1)
    // 但是这可以：
    let was = std::mem::take(s); // (2)
    // 这也可以：
    *s = was; // (3)
    // 可以在 &mut 后面交换值：
    let mut r = Box::new(84);
    std::mem::swap(s, &mut r); // (4)
    assert_ne!(*r, 84);
}

let mut s = Box::new(42);
replace_with_84(&mut s);
// (5)

// 清单 2-7：可变性仅适用于直接引用的内存。
```

你不能简单地将值移出 `1`，因为调用者仍然认为他们拥有该值，并会在 `5` 处再次释放它，导致双重释放。如果你只是想留下一些有效的值，`std::mem::take` (2) 是一个不错的选择。它相当于 `std::mem::replace(&mut value, Default::default())`；它将值从可变引用后面移出，但为该类型留下一个新的、默认的值。默认值是一个单独的、自有的值，所以当作用域在 `5` 处结束时，调用者可以安全地析构它。

另外，如果你不需要引用后面的旧值，你可以用一个你已经拥有的值覆盖它 (3)，让调用者以后再丢弃这个值。当你这样做的时候，原来在可变引用后面的值会被立即丢弃。

最后，如果你有两个可变的引用，你可以在不拥有任何一个引用的情况下交换它们的值 (4)，因为两个引用最后都会有一个合法拥有的值，供它们的主人最终释放。

### 内部可变性

有些类型提供内部可变性，这意味着它们允许您通过共享引用改变值。这些类型通常依赖于额外的机制（如原子 CPU 指令）或不变量来提供安全的可变性，而不依赖于独占引用的语义。它们通常分为两类：一类允许您通过共享引用获得可变引用，另一类允许您替换仅给定共享引用的值。

第一类包括 `Mutex` 和 `RefCell` 这样的类型，它们包含安全机制，以确保对于它们提供的任何可变引用，同一时刻只能存在一个可变引用（没有共享引用）。在本质上，这些类型（以及类似的类型）都依赖于一个名为 `UnsafeCell` 的类型，该类型的名称会立即让您犹豫是否使用它。我们将在第 9 章中更详细地介绍 `UnsafeCell`，但现在你应该知道，这是通过共享引用进行变异的唯一正确方法。

提供内部可变性的其他类型是那些不给出内部值的可变引用，而只是提供适当操作该值的方法的类型。`std::sync::atomic` 和 `std::cell::cell` 类型中的原子整数就属于这一类。您不能直接获取此类类型后面的 `usize` 或 `i32` 的引用，但是可以在给定的时间点读取和替换它的值。

> 注意：标准库中的 Cell 类型是通过不变量实现安全内部可变性的一个有趣的例子。它不能跨线程共享，并且永远不会给出对 Cell 中包含的值的引用。相反，这些方法要么完全替换该值，要么返回所包含值的副本。由于内部值不能存在任何引用，所以移动它总是可以的。而且，由于 Cell 不能在线程之间共享，因此即使通过共享引用发生突变，内部值也永远不会并发地发生突变。

### 生存期

如果您正在阅读这本书，您可能已经熟悉了生存期的概念，这可能是由于编译器对生存期规则违反的反复通知。这种程度的理解将为您编写的大多数 Rust 代码提供良好的服务，但是随着我们深入研究 Rust 更复杂的部分，您将需要一个更严格的心智模型来工作。

较新的 Rust 开发人员经常被教导将生存期与作用域相对应：生存期开始于对某个变量的引用，结束于该变量被移动或超出作用域。这通常是正确的，通常也是有用的，但实际情况要复杂一些。生存期实际上是某个引用必须有效的代码区域的名称。虽然生存期经常与作用域相一致，但这并不是必须的，正如我们将在本节后面看到的那样。

#### 生存期和借用检查器

Rust 生存期的核心是借用检查器。每当一个具有某种生存期的引用被使用时，借用检查器就会检查 `'a` 是否仍然活着。它通过追踪路径回到 `'a` 开始的地方--引用被取走的地方--从使用点开始，并检查该路径上是否有冲突的使用。这可以确保引用仍然指向一个可以安全访问的值。这类似于我们在本章前面讨论的高级 "数据流 " 心智模型；编译器检查我们正在访问的引用的流不会与任何其他并行流相冲突。

清单 1-8 显示了一个简单的代码例子，其中有对 `x` 的引用的生存期注释。

```rust
let mut x = Box::new(42);
let r = &x;   // (1)            // 'a
if rand() > 0.5 {
    *x = 84;  // (2)
} else {
    println!("{}", r); // (3)   // 'a
}
// (4)

// 清单 1-8：生存期不需要是连续的
```

当我们对 `x` 进行引用时，生存期从 (1) 开始。在第一个分支 (2) 中，我们立即尝试修改 `x`，将其值更改为 `84`，这需要一个 `&mut x`。借用检查器取出 `x` 的可变引用并立即检查其使用情况。它发现在获取引用和使用引用之间没有冲突，所以它接受代码。这是个令人惊讶的消息如果你习惯于思考生存期范围，因为 `r` 仍在范围 (2)（超出范围在 (4))。但是借用检查器足够聪明，它意识到如果这个分支被选中，以后就不会再使用 `r`，因此 `x` 在这里被可变访问是没有问题的。或者，换一种说法，在 (1) 处创建的生存期不会扩展到这个分支：没有来自 `r` 超过 (2) 的流，因此没有冲突流。然后借用检查器在打印语句 (3) 中找到了对 `r` 的使用。它沿着路径返回到 (1)，并发现没有冲突的用途 ((2) 不在该路径上），所以它也接受这种用途。

如果我们在清单 1-8 中在 `4` 处添加 `r` 的另一个使用，代码将不再编译。生存期 `'a` 将从 (1) 一直持续到 (4) (`r` 的最后一次使用），当借用检查器检查 `r` 的新的使用时，它会在 (2) 处发现一个冲突的使用。

生存期可以变得相当复杂。在清单 1-9 中，你可以看到一个有漏洞的生存期的例子，它在开始和最终结束的地方间歇性地失效了

```rust
let mut x = Box::new(42);
let mut z = &x;        // (1)   // 'a
for i in 0..100 {
    println!("{}", z); // (2)   // 'a
    x = Box::new(i);   // (3)
    z = &x;            // (4)   // 'a
} 
println!("{}", z);              // 'a

// 清单 1-9: 生存期有漏洞
```

当我们对 `x` 进行引用时，生存期从 (1) 开始。然后我们在 (3) 处离开 `x`，这将结束生存期 `'a`，因为它不再有效。借用检查器通过考虑 `'a` 结束于 (2)，这使得 `x` 和 (3) 之间没有冲突流”来接受这一移动。然后，通过更新 `z` (4) 中的引用，重新启动生存期。无论代码现在是循环回到 `2` 还是继续到最后的 `println!` 语句，这两个用途现在都有一个有效的值可以流出来，而且没有冲突的流，所以借用检查器接受了这段代码。

同样，这与我们之前讨论的内存的数据流模型完全一致。当 `x` 被移动时，`z` 停止存在。当我们稍后重新分配 `z` 时，我们创建了一个全新的变量，这个变量只从这一点开始存在。碰巧的是，这个新变量也被命名为 `z`。

> 注意：借用检查器是，而且必须是，保守的。如果它不确定一个借用是否有效，它就会拒绝它，因为允许一个无效的借用的后果可能是灾难性的。借用检查器越来越聪明，但有时它也需要帮助来理解为什么一个借用是合法的。这就是为什么我们有不安全的 Rust 的部分原因。

#### 泛型生存期

偶尔你需要在自己的类型中存储引用，这些引用需要有一个生存期，这样当它们被用于该类型的各种方法时，借用检查器可以检查它们的有效性。如果你想让你的类型上的一个方法返回一个比对 `self` 的引用更久远的引用，这一点尤其重要。

Rust 允许您在一个或多个生存期内使类型定义泛型，就像它允许您使类型泛型一样。Steve Klabnik 和 Carol Nichols 合著的《Rust 编程语言》(No Starch Press, 2018) 详细介绍了这个主题，所以我在此不再赘述基本内容。但是，当您编写这种性质的更复杂类型时，您应该注意这些类型和生存期之间的交互有两个微妙之处。

首先，如果你的类型也实现了 `Drop`，那么丢弃你的类型也算作使用你的类型的任何生存期或类型的泛型。 基本上，当你的类型的一个实例被析构时，借用检查器将检查在析构它之前使用你的类型的任何泛型生存期是否仍然合法。这是必要的，以防你的析构代码确实使用了任何这些引用。如果你的类型没有实现 `Drop`，析构这个类型就不算是使用，用户只要不再使用你的类型，就可以自由地忽略存储在你的类型中的任何引用，就像我们在清单 1-7 中看到的那样。我们将在第 9 章中更多地讨论这些关于析构的规则。

其次，虽然一个类型可以存在多个泛型生存期，但经常这样做只会使你的类型特征变得不必要的复杂。通常情况下，一个类型只使用一个泛型生存期就可以了，编译器会将插入到你的类型中的任何引用的生存期中较短的一个作为这个生存期。只有当你有一个包含多个引用的类型，并且它的方法返回的引用应该只与其中一个引用的生存期相联系时，你才应该真正使用多个泛型生存期参数。

考虑清单 1-10 中的类型，它为您提供了一个迭代器，迭代器将遍历由特定的其他字符串分隔的字符串部分。

```rust
struct StrSplit<'s, 'p> {
    delimiter: &'p str,
    document: &'s str,
}
impl<'s, 'p> Iterator for StrSplit<'s, 'p> {
    type Output = &'s str;
    fn next(&self) -> Option<Self::Output> {
        todo!()
    }
}
fn str_before(s: &str, c: char) -> Option<&str> {
    StrSplit { document: s, delimiter: &c.to_string() }.next()
}

// 清单 1-10： 一个需要多个泛型生存期的类型
```

当你构造这个类型时，你必须给出 `delimiter` 和要搜索的 `document` ，这两个都是对字符串值的引用。 当你要求下一个字符串时，你会得到一个对 `document` 的引用。考虑一下如果你在这个类型中使用一个单一的生存期会发生什么。迭代器产生的值将与 `document` 的生存期和分隔符相联系。这将使 `str_before` 无法编写：返回类型将有一个与函数本地变量相关的生存期-- `to_string` 产生的 `String`--借用检查器将拒绝该代码。

#### 生存期型变

型变 (Variance) 是程序员经常接触到的一个概念，但很少知道它的名字，因为它大多是看不见的。型变描述了什么类型是其他类型的子类型，以及什么时候可以用子类型代替父类型（反之亦然）。 广义上讲，如果一个类型 `A` 至少和 `B` 一样有用，那么它就是另一个类型 `B` 的子类型。在 Java 中，如果 `Turtle` 是 `Animal` 的子类型，你可以把 `Turtle` 传给接受 `Animal` 的函数，或者在 Rust 中，你可以把一个 `&'static str` 传给接受 `&'a str` 的函数，这就是型变。

虽然型变通常隐藏在视线之外，但它经常出现，我们需要对它有一个工作上的了解。乌龟是动物的一个子类型，因为乌龟比某些未指定的动物更 "有用"--乌龟可以做任何动物能做的事，而且可能更多。同样，`'static` 是 `'a` 的一个子类型，因为 `'static` 的寿命至少与任何 `'a` 一样长，所以更有用。或者，更一般地说，如果 `'b:'a`（`'b` 比 `'a` 长寿），那么 `'b` 就是 `'a` 的一个子类型。这显然不是正式的定义，但是它已经足够接近实际用途了。

所有类型都有一个型变，它定义了哪些其他类似的类型可以用于该类型的位置。有三种型变：协变（covariant）、不变（invariant）和逆变（contravariant）。如果你可以只使用一个子类型来代替该类型，那么该类型就是协变的。例如，如果一个变量是 `&'a T` 类型，你可以给它提供一个 `&'static T` 类型的值，因为 `&'a T` 在 `'a` 上是协变的。`&'a T` 在 `T` 上也是协变的，所以你可以把一个 `&Vec<&'static str>` 传递给一个接受 `&Vec<&'a str>` 的函数。

有些类型是不变的，这意味着你必须准确提供给定的类型。`&mut T` 就是一个例子--如果一个函数接受一个 `&mut Vec<&'a str>`，你不能把一个 `&mut Vec<&'static str>` 传给它。也就是说，`&mut T` 在 `T` 上是不变的。如果你可以，函数可以在 `Vec` 中放入一个短暂的字符串，然后调用者会继续使用它，认为它是一个 `Vec<&'static str>`，从而认为包含的字符串是 `'static` ！任何提供可变性的类型一般都是不变的，原因也是如此--例如，`Cell<T>` 在 `T` 上是不变的。

最后一类，逆变，出现在函数参数上。如果函数类型可以接受其参数不那么有用，那么它们就会更有用。如果你将参数类型本身的型变与它们作为函数参数时的型变进行对比，这一点就更清楚了：

```rust
let x: &'static str; // 更有用，活的更长
let x: &'a str; // 不太有用，活得更短
fn take_func1(&'static str) // 更严格，所以不那么有用
fn take_func2(&'a str) // 不太严格，所以更有用
```

这种翻转的关系表明，`Fn(T)` 在 `T`上是逆变的。

那么，当涉及到生存期时，为什么需要学习型变呢？当您考虑通用生存期参数如何与借用检查器交互时，型变就变得很重要了。考虑清单 1-11 所示的类型，它在一个字段中使用多个生存期。

```rust
struct MutStr<'a, 'b> {
    s: &'a mut &'b str
}
let mut s = "hello";
*MutStr { s: &mut s }.s = "world"; // (1)
println!("{}", s);

// 清单 1-11: 需要多个泛型生存期的类型
```

乍一看，在这里使用两个生存期似乎是不必要的--我们没有需要区分结构中不同部分的借用的方法，就像我们在清单 1-10 中的 `StrSplit` 那样。 但是如果你把这里的两个生存期换成一个 `'a`，代码就不再能被编译了！这就是为什么我们在这里使用了两个生命周期。而这一切都是因为型变。

> 注意：(1) 处的语法可能看起来很奇怪。它相当于定义了一个持有 `MutStr` 的变量 `x`，然后写 `*x.s = "world"`，只是没有变量，所以 `MutStr` 被立即删除了。

在 (1) 处，编译器必须确定生存期参数应该被设置为什么生存期。如果有两个生命期，`'a` 被设置为有待确定的 `s` 的借用生存期，`'b` 被设置为 `'static`，因为那是提供的字符串 "hello" 的生命期。如果只有一个生命周期 `'a` ，编译器推断该生命周期必须是 `'static`。

当我们后来试图通过共享引用访问字符串引用 `s` 来打印它时，编译器试图缩短 `MutStr` 使用的 `s` 的可变借用，以允许 `s` 的共享借用。

在双生存期的情况下，`'a` 只是在 `println!` 之前结束，`'b` 保持不变。另一方面，在单生存期的情况下，我们遇到了问题。编译器想缩短 `s` 的借用，但要做到这一点，它也必须缩短 `str` 的借用。虽然 `&'static str` 一般来说可以缩短为任何 `&'a str` （ `&'a T` 在 `'a` 中是协变的），但这里它在 `&mut T` 后面，而 `&mut T` 在 `T` 中是不变量的。不变要求相关类型永远不会被子类型或父类型取代，所以编译器缩短借用的尝试失败了，它报告说该清单仍然是可变的借用。哎哟！

由于型变带来的灵活性的降低，你想确保你的类型在尽可能多的泛型参数上保持协变（或在适当情况下保持逆变）。如果这需要引入额外的生命期参数，你需要仔细权衡增加一个参数的认知成本和型变的人机工程成本。

## 总结

本章的目的是建立一个坚实的、共享的基础，我们可以在接下来的章节中建立这个基础。到现在，我希望你觉得你已经牢牢掌握了 Rust 的内存和所有权模型，那些你可能从借用检查器中得到的错误似乎不那么神秘了。你可能已经知道了我们在这里所涉及的零星内容，但希望这一章能给你一个更全面的印象，让你知道这一切是如何结合起来的。在下一章中，我们将为类型做一些类似的事情。我们将讨论类型是如何在内存中表示的，看看泛型和特质（trait）是如何产生运行代码的，并看看 Rust 为更高级的用例提供的一些特殊类型和特质结构。

你不知道的JavaScript（上卷）
=========================

## 作用域和闭包

### 作用域是什么

几乎所有的编程语言最基本的功能之一，就是能够存储变量当中的值，并且能在之后对这个值进行访问和修改。事实上，正式这种存储和访问变量的值的能力将状态带给了程序。

若没有了状态这个概念，程序虽然也能够执行一些简单的任务，但它会受到高度限制，做不到非常有趣。

但是将变量引入程序会引起很有意思的问题，也正式我们将要讨论：这些变量住在哪里？换句话说，他们储存在哪里？最重要的是，程序需要时如何找到他们？

这些问题说明需要一套良好的规则来存储变量，并且之后可以方便地找到这些变量。这套规则被称为作用域。

但是究竟在哪里而且怎么设置这些作用域规则呢？

#### 1.1编译原理

尽管通常将JavaScript归类为“动态”或“解释执行”语言，但事实上它是一门编译语言非常相似，在某些环节可能比预想的要复杂。

在传统编译语言的流程中，程序中的一段源代码在执行之前会经历三个步骤，统称为“编译”。

* 分词/词法分析
> 这个过程会将由字符组成的字符串分解成（对编程语言来说）有意义的代码块，这些代码块被称为词法单元（token）。例如，考虑程序`var a = 2;`。这段程序通常会被分解成为下面的词法单元：`var 、 a、 =、 2、 ；`。空格是否会被当做词法单元，取决于空格在这门语言中是否具有意义。

> **分词和词法分析之间的区别是非常微妙、晦涩的，主要差异在于词法单元的识别是通过有状态还是无状态的方式进行的。简单来说，如果词法单元生成器在判断a是一个独立的词法单元还是其他词法单元的一部分时，调用的是有状态的解析规则，那么这个过程就被称为词法分析。**

* 解析/语法分析

> 这个过程是将词法单元流转换为一个由元素逐级嵌套所组成的代表了程序语法结构的树。这个树被称为“抽象语法树”。`var a = 2;`的抽象语法树中可能会有一个叫做VariableDeclaration的顶级节点，接下来是一个叫做Identifier（它的值是a）的子节点，以及一个叫做AssignmentExpression的子节点。AssignmentExpression节点有一个叫做Numericliteral（它的值是2）的子节点。

* 代码生成

> 将AST转换为可执行的代码的过程称为代码生成。这个过程与语言、目标平台等息息相关。
> 
> 抛开具体细节，简单来说就是某种方法可以将`var a = 2;`的AST转化为一组及其指令，用来创建一个叫做a得变量（包括分配内存等），并将一个存储存在a中。
> 
> **关于引擎如何管理系统资源超出了我们的探讨范围，因此只需要简单地了解引擎可以根据需要创建并存储变量即可。**

比起那些辨析过程只有三个步骤的语言编译器，JavaScript引擎要复杂的多。例如，在语法分析和代码生成阶段特定的步骤来对运行性能进行优化，包括冗余元素进行优化等。

因此在这里只进行宏观、简单的介绍，接下来你就会发现我们介绍的这些开起来有点高深的内容与所要讨论的事情有什么关联。

首先，JavaScript引擎不会有大量的（像其他语言编译器那么多）时间用来进行优化，因为与其他语言不同，JavaScript的编译过程不是发生在构建之前的。

对于JavaScript来说，大部分情况下编译发生在代码执行前的几微秒（甚至更短！）的时间内。在我们所要讨论的作用域背后，JavaScript引擎用尽了各种办法（比如JIT，可以贪吃编译甚至实施重编译）来保证性能最佳。

简单地来说，任何JasvaScript代码片段在执行前都要进行编译（通常就在执行前）。因此，JavaScript编译器首先会对`var a = 2;`这段程序进行编译，然后做好执行它的准备，并且通常马上就会执行它。

### 1.2理解作用域

我们学习作用域的方式是将这个过程模拟成几个任务之间的对话。那么，由谁进行这场对话呢？

#### 1.2.1演员表

首先介绍将要参与到对程序`var a = 2;`进行处理的过程中的演员们，这样才能理解接下来将要被听到的对话。

* 引擎
> 从头到尾负责整个JavaScript的程序编译及执行过程。

* 编译器
> 引擎的好朋友之一，负责语法分析以及代码生成等脏活累活

* 作用域
> 引擎的另一位好朋友，负责收集并维护由所有生命的标识符组成的一系列查询，并实施一套非常严格的规则，确定当前执行的代码对这些标识符的访问权限。

为了能够完全理解JavaScript的工作原理，你需要开始像引擎（和它的朋友们）一样思考，从他们的角度提出问题，并从他们的角度回答这些问题。

#### 1.2.2

当你看见`var a = 2;`这段程序时，很可能认为这是一句声明。但我们的新朋友引擎却不这么看。事实上，引擎认为这里有两个完全不同的声明，一个由编译器在编译时处理，另一个则由引擎运行时处理。

下面我们将`var a = 2;`分解，看看引擎和它的朋友们是如何协同工作的。

编译器首先会将这段程序分解为词法单元，单后将词法单元解析成一个树结构。但是当编译器开始进行代码生成时，它对这段程序的处理方式会和预期的有所不同。

可以合理地假设编译器所产生的代码能够用下面的伪代码进行概括：“为一个变量分配内存，将其命名为a，然后将值2保存金这个变量。”然而，这并不完全正确。

事实上编译器会进行如下处理。

1. 遇到`var a`，编译器会询问所用于是否已经有一个该名称的变量存在于同一个作用域集合中。如果是，编译器会忽略该声明，继续进行编译；否则它会要求作用域在当前作用域的集合中声明一个新的变量，并命名为a。
2. 接下来编译器会为引擎生成运行时所需要的代码，这些代码被用来处理`a = 2`这个复制操作。引擎运行时会首先询问作用域，在当前的作用域集合中是否在一个叫做a的变量。如果是，引擎就会使用这个变量；如果否，引擎会继续查找该变量。

如果引擎最终找到了a变量，就会将2赋值给它。否则引擎会拒收十一跑出一个异常！

**总结：**变量的赋值操作会执行两个动作，首先编译器在单钱作用域中声明一个变量（如果之前没有声明过），然后在运行时引擎会在作用域中查找该变量，如果能够找到就会对它赋值。

#### 1.2.3 编译器有话说

为了进一步理解，我们需要介绍一点编译器的术语

编译器在编译过程的第二步中生成了代码，引擎执行它时，会通过查找变量a来判断它时否已声明过。查找的过程由作用域进行协助，但是引擎执行怎样的查找，会影响最终的查找结果。

在我们的例子中，引擎会为变量a进行HS查询。另外一个查找的类型叫做RHS。我打赌你一定能猜到“L”和“R”的含义，它们分别代表左侧和右侧。

什么东西的左侧和右侧？是一个赋值操作的左侧和右侧。

换句话说，当变量出现在赋值操作的左侧时进行LHS查询，出现在右侧时进行RHS查询。

将的更准确一点，RHS查询与简单地查找某个变量的值别无二致，而LHS查询则是试图找到变量的容器本身，从而可以对其赋值，从这个角度，RHS并不是真正意义的“赋值操作的右侧”，更准确地说“非左侧”。

你可以将RHS理解成retrieve his source value (取到它的原值)，这意味着“得到某某的值”。

让我们继续深入研究。

考虑一下代码：

```js

console.log( a );

```

其中对a的引用是一个RHS引用，因此这里a并没有赋予任何值。相应地，需要查找并取得a的值，这样才能将值传递给`console.log(..)`。

**LHS和RHS的含义是“赋值操作的左侧和右侧”并不一定意味着就是“=赋值操作符的左侧或右侧”。赋值操作还有其他几种形式，因此在概念上最好将其理解为“赋值操作的目标是谁（LHS）”以及“谁是赋值操作的源头（RHS）”。**

考虑下面的程序，其中既有LHS也有RHS引用：

```js

function foo(a) {
	console.log( a ); // 2
};

foo( 2 );

```

最后一行foo(...)函数的调用需要对foo进行RHS引用，这意味着“去找到foo的值，并把它给我”。并且(..)意味着foo的值需要被执行，因此它最好真的是一个函数类型的值！

这里还有一个容易被忽略却非常重要的细节。

代码中隐式的`a = 2`操作可能很容易被你忽略掉。这个曹组哦发生在2被当做参数传递给foo(..)函数时，2会被分配给参数a。为了给参数a（隐式）分配值，需要进行一次LHS查询。

这里还有对a进行RHS引用，并且将得到的值传给console.log(..)。console.log(..)本身也需要一个引用才能执行，因此会对console.log对象进行RHS查询，并且检查得到的值中是否有一个叫做log的方法。

最后，在概念上可以理解为在LHS和RHS之间通过对值2进行交互来讲起传递给log(..)（通过变量a的RHS查询）。假设在log(..)函数的原生实现中它可以接受参数，在将2赋值给其中第一个（也许叫做arg1）参数之前，这个参数需要进行LHS引用查询。

**你可能会倾向于将函数声明function foo(a) {... 概念化为普通的变量声明和负值，比如var foo、 foo = function {...。如果这样理解的话，这个函数声明将需要进行LHS查询。然而还有一个重要的细微差别，编译器可以在代码生成的同事处理声明和值的定义，比如在引擎执行代码时，并不会有线程专门用来将一个函数值“分配给”foo。因此，将函数声明理解成前面讨论的LHS查询和负值的形式并不合适。**

#### 1.2.4 引擎和作用域的对话

```js

function foo(a) {
	console.log( a ); // 2
};

foo( 2 );

```

让我们吧上面这段代码处理过程想象成一段对话，这段对话可能是下面这样的。

> 引擎： 我说作用域，我需要为foo进行RHS引用。你见过吗？
> 
> 作用域： 别说，我还真见过，编译器那小子刚刚声明了它。它是一个函数，给你。
> 
> 引擎： 哥们太够意思啦！好吧，我们来执行一下foo
> 
> 引擎： 作用域，还有个事儿。我需要为a进行LHS引用，这个你见过吗？
> 
> 作用域： 这个也见过，编译器最近把它声明为foo的一个形式参数了，那去吧。
> 
> 引擎：大恩不言谢，你总是这么棒。现在我要把2赋值给a。
> 
> 引擎：哥们，不还意思又来打扰你。我要为console进行RHS引用，你见过它吗？
> 
> 作用域：咱们俩谁跟谁啊，再说我就是干这个。这个我也有，console是个内置对象。给你。
> 
> 引擎：么么哒。我看看这里面是不是有log(..)。太好了，找到了，是一个函数。
> 
> 引擎：哥们，能帮我再找一下a的RHS引用吗？虽然我记得它，但想再确认一次。
> 
> 作用域： 放心吧，这个变量没有变动过，拿走不谢。
> 
> 引擎：真棒。我来把a的值，也就是a，传递给log(..)。

#### 1.2.5 小测验

检验一下到目前的理解程度。把自己当做引擎，并同作用域进行一次“对话”：

```js

function foo(a) {
	var b = a;
	return a + b;
};
var c = foo( 2 );

```

1. 找到其中所有的LHS查询。（这里有3处！）
2. 找到其中所有的RHS查询。（这里有4处！）

### 1.3 作用域嵌套

我们说过，作用域是根据名称查找变量的一套规则。实际情况中，通常需要同时估计几个作用域。

当一个块或函数嵌套在另一个块或函数中时，就发生了作用域的嵌套。因此，在当前作用域中无法找到某个变量时，引擎就会在外层的作用域中继续查找，知道找到该变量，或抵达最外层的作用域（也就是全局作用于）为止。

考虑一下代码：

```js

function foo(a) {
	console.log( a + b );
};
var b = 2;
foo( 2 ); // 4

```

对b进行的RHS引用无法在函数foo内部完成，但可以再上一级作用域（在这个例子中就是全局作用域）中完成。

因此回顾一下引擎和作用域之间的对话，会进一步听到：

> 引擎： foo的作用域兄弟，你见过b吗？我需要对它进行RHS引用。
> 
> 作用域： 听都没听过，走开。
> 
> 引擎： foo的商机作用域兄弟，咦？有眼不识泰山，原来你是全局作用域大哥，太好了。你见过b吗？我需要对它进行RHS引用。
> 
> 作用域：当然了，给你吧。

遍历嵌套作用域链的规则很简单：引擎从当前的执行作用域开始查找变量，如果找不到，就向上一级继续查找。当抵达最外层的全局作用域时，无论找到还是没找到，查找过程都会停止。

#### 把作用域链比喻成一个建筑

为了将作用域处理的过程可视化，我希望你在脑中想象下面这个高大的建筑：

![](http://oqpmmru7y.bkt.clouddn.com/zuoyongyulian.png)

这个建筑代表程序中的嵌套作用域链。第一层楼代表当前的执行作用域，也就是你所处的位置。建筑的顶层代表全局作用域。

LHS和RHS引用都会在当前楼层进行查找，如果没有找到，就会做电梯前往上一层楼，如果还是没有找到就继续向上，以此类推。一旦抵达顶层（全局作用域），可能找到了你所需要的变量，也可能没找到，但无论如何查找过程都将停止。

### 1.4 异常

为什么区分LHS和RHS是一件重要的事情？

因为变量还没有声明（在任何作用域中无法找到该变量）的情况下，这两种查询的行为是不一样的。

考虑如下代码：

```js

function foo(a) {
	console.log( a + b );
	b = a;
};
foo( 2 );

```

第一次对b进行RHS查询时是无法找打变量的。也就是说，这是一个“未声明”的变量，因为在任何相关的作用域中都无法找到它。

如果RHS查询在所有嵌套的作用域中遍寻不到所需的变量，引擎就会跑出ReferenceError异常。值得注意的是，ReferenceError是非常重要的异常类型。

相较之下，当引擎执行LHS查询时，如果在顶层（全局作用域）中也无法找到目标变量，全局作用域中就会创建一个具有该名称的变量，并将其返回给引擎，前提是程序运行在非“严格模式”下。

“不，这个变量之前并不存在，但是我们很热心地帮你创建一个。”

ES5中引入可“严格模式”。同正常模式，或者说宽松/懒惰模式相比，严格模式在行为上有很多不同。其中一个不同的行为是严格模式禁止自动或饮食地创建全局变量。因此，在严格模式中LHS查询失败时，并不会创建并返回一个全局变量，引擎会抛出同RHS查询失败时类似的ReferenceError异常。

接下来，如果RHS查询找到了一个变量，但是你尝试对这个变量的值进行不合理的操作比如试图对一个函数类型的值进行函数调用，或者引用null或undefined类型的值中的属性，那么引擎会抛出另外一种异常，叫做TypeError。

ReferenceError同作用域判别失败相关，而TypeError则代表作用域判别成功了，但是对结果的操作是非法或不合理的。

### 1.5 小结

作用域是一套规则，用于确定在何处以及如何查找变量（标识符）。如果查找的目的是对变量进行赋值，那么就会使用LHS查询；如果目的是获取变量的值，就会使用RHS查询。赋值操作符导致LHS查询。=操作符或调用函数时传入参数的操作都会关联作用域的赋值操作。

JavaScript引擎首先会在代码执行前对其进行编译，在这个过程，像var a = 2这样的声明被分解成两个独立的步骤：

1. 首先，var a 在其作用域中声明新变量。这回在最开始的阶段，想var a = 2这样的声明会被分解成为两个独立的步骤。
2. 接下来，a = 2会查询（LHS查询）变量a并对其进行赋值。

LHS和RHS出去啊新都会在当前执行作用域开始，如果有需要（也就是说他们没有找到所需要的标识符），就会向上级作用域继续查找目标标识符，这样每次上升一级作用域（一层楼），最后抵达全局作用域（顶层），无论找到或没找到都将停止。

不成功的RHS引用会导致抛出ReferenceError异常。不成功的LHS引用会导致自动隐式地创建一个全局变量（非严格模式下），该变量使用LHS引用的目标作为标识符，或者抛出ReferenceError异常（严格模式下）。








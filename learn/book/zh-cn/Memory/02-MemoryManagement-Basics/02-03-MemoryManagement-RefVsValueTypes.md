It seems like you haven't provided any text to translate. Could you please provide the text you want translated into Chinese?

# 引用类型和值类型

现在，当我们加强了对 .NET 中内存管理基础的理解后，让我们来讨论一下引用类型和值类型。如果要谈论它们之间的区别以及每种类型的用处，我首先想提到的是我对它们命名的看法。在我谦虚的观点中，如果在俄语环境中将它们称为引用类型和值类型，而不是说 Value Types 和 Reference Types，那么理解它们之间的区别就会变得更加直观。

> 当人们被问及什么是引用类型和值类型时，他们经常回答说引用类型存在于堆中，而值类型存在于栈中。这是完全错误的。这只是真相的一小部分，根本不能算是真相。

为了理解它们之间的区别，让我们从它们的区别来学习它们：

  - *值类型*：**整个结构**是其值。对于*引用类型*，其值是对对象的**引用**；
  - 在内存结构上：值类型只包含你指定的数据。引用类型还包含两个系统字段。第一个用于存储 `SyncBlockIndex`，第二个用于存储类型信息：包括 Virtual Methods Table (VMT)；
  - 然而，引用类型可以被继承，通过重写方法。值类型没有这样的能力；
  - 但是，要分配引用类型，需要在堆上分配空间。值类型*可以*在栈上工作，不进入堆，也可以成为引用类型的一部分。这个属性可以显著提高某些算法的性能；

然而，它们也有共同之处。

>{.big-quote}  两个子类都继承自 object 类型。<br>
    这意味着，它们可以作为其代表：拥有完整的权利

让我们分别考虑每个特性。

## 复制

类型之间最主要和基本的区别可以大致描述如下：

  - 任何接受引用类型的变量、类/结构的字段或方法参数，实际上存储的是对值的**引用**；
  - 而任何接受值类型的变量、类/结构的字段或方法参数，实际上存储的是值本身。即整个结构；

这对我们意味着什么？这特别意味着，任何赋值或通过方法参数传递都会导致值的复制。更改副本不会更改原始对象。而如果你更改引用类型的字段，所有拥有该类型实例引用的人都会看到这些更改。让我们通过一个例子来看看：

{.wide}
```csharp

 DateTime dt = DateTime.Now;   // 这里首先在调用方法时为 DateTime 变量分配空间，
                               // 但它将被零填充。然后将 Now 属性的所有值复制到 dt 变量中
 DateTime dt2 = dt;            // 这里的值再次被复制

 object obj = new object();    // 创建一个对象，在 SOH 中分配内存，并将对象的指针放在 obj 变量中
 object obj2 = obj;            // 这里我们复制了对该对象的引用。即对象是一个，引用是两个

```     

这个属性产生了一系列看似模棱两可的代码结构。其中之一是在集合中更改值：

```csharp
// 声明一个结构
struct ValueHolder
{
    public int Data;
}

// 创建这样一个结构的数组并初始化 Data 字段为 5
var array = new [] { new ValueHolder { Data = 5 } };

// 通过索引获取结构并将 Data 字段设置为 4
array[0].Data = 4;

// 检查值
Console.WriteLine(array[0].Data);
```

在这段代码中有一个小技巧。代码看起来好像我们首先获取结构的实例，然后在得到的副本上设置新的 Data 字段值。这意味着在检查时我们应该再次得到 `5`。但事实并非如此。问题在于，MSIL 有一个特殊的指令用于设置数组中结构的字段值。它是为了提高性能而引入的。这段代码将按照其作者的意图执行：程序将在控制台上输出数字 `4`。

但是，如果稍微改变示例：

```csharp
// 声明一个结构
struct ValueHolder
{
    public int Data;
}

// 创建这样一个结构的列表并初始化 Data 字段为 5
var list = new List<ValueHolder> { new ValueHolder { Data = 5 } };

// 通过索引获取结构并将 Data 字段设置为 4
list[0].Data = 4;

// 检查值
Console.WriteLine(list[0].Data);
```

这样我们什么也编译不了。原因是当你写 `list[0].Data = 4` 时，你首先得到了结构的副本。因为你实际上是在调用 `List<T>` 类型的实例方法，它隐藏在索引访问背后。它从内部数组中获取结构的副本，该副本从索引访问方法返回给你。然后你尝试修改副本，该副本随后不会被使用。这不是错误，但绝对是毫无意义的代码。编译器知道人们对值类型感到困惑，因此禁止这种行为。因此，示例应该被重写如下：

{.wide}
```csharp
// 声明一个结构
struct ValueHolder
{
    public int Data;
}

// 创建这样一个结构的列表并初始化 Data 字段为 5
var list = new List<ValueHolder> { new ValueHolder { Data = 5 } };

// 通过索引获取结构并将 Data 字段设置为 4，然后保存回去
var copy = list[0];
copy.Data = 4;
list[0] = copy;

// 检查值
Console.WriteLine(list[0].Data);
```

尽管看起来有些冗长，但它是正确的。当程序执行时，控制台将输出数字 `4`。

下一个例子我想向你展示的是，"结构的值是整个结构"这一概念的含义：

```csharp
// 方案 1
struct PersonInfo
{
    public int Height;
    public int Width;
    public int HairColor;
}

int x = 5;
PersonInfo person;
int y = 6;

// 方案 2
int x = 5;
int Height;
int Width;
int HairColor;
int y = 6;
```

从数据在内存中的位置来看，这两个例子是相同的。因为结构的值是整个结构。它占据了它所在的空间，并为自己定义了内存。

```csharp
// 方案 1
struct PersonInfo
{
    public int Height;
    public int Width;
    public int HairColor;
}

class Employee
{
    public int x;
    public PersonInfo person;
    public int y;
}

// 方案 2
class Employee
{
    public int x;
    public int Height;
    public int Width;
    public int HairColor;
    public int y;
}
```

这些示例在内存中元素位置的角度来看也是相同的，因为结构只是被放置在它被定义的地方：在类的字段中。我并不是说这完全相同：毕竟在结构中你可以通过结构的方法来操作它的字段。

如果我们谈论引用类型，那么情况显然不同。实例本身位于不可达的 Small Object Heap (SOH) 或 Large Object Heap (LOH) 中，而在类的字段中只会记录实例的指针值：32位或64位数字。

最后一个例子，我希望不会让你感到困惑。但我想在这个问题上画上句号。

```csharp
// 方案 1
struct PersonInfo
{
    public int Height;
    public int Width;
    public int HairColor;
}

void Method(int x, PersonInfo person, int y);

// 方案 2
void Method(int x, int HairColor, int Width, int Height, int y);
```

你完全正确地理解了：从内存工作的角度来看，这两种方案将以相同的方式工作（但不是从架构的角度来看！这并不是可变参数数量的替代）。为什么顺序改变了？因为方法参数是一个接一个地声明的，并且按照这个顺序放入线程的栈中。然而，栈是从高地址向低地址增长的，这意味着按顺序放置的顺序将与整个结构放置的顺序不同。

## 可重写方法和继承

它们之间的第二个全球性差异是结构中没有虚方法表。这意味着：

  1. 在结构中不能声明 `virtual` 方法，也不能重写它们；
  2. 结构原则上不能相互继承。唯一的继承模拟方法是将基本类型结构作为第一个字段放置。然后它们的偏移将匹配，"继承"结构的字段将位于"基本"字段之后，从逻辑上你将实现继承；
  3. 与类不同，结构可以被传递到非托管代码中。我是指它的值。类型信息当然会丢失。因为结构只是一个数据填充的内存段，没有类型信息。这意味着它可以不加更改地传递给非托管方法，例如，用 C++ 编写的方法。

>{.big-quote}   虽然缺少虚方法表剥夺了结构一些继承带来的"魔法"，但也赋予了它们一系列优势。

首先也是最重要的一点已经提到了：我们可以轻松地将结构的实例传递给外部世界（超出 .NET Framework 的范围）。这只是一段内存！或者我们可以从非托管代码接收一段内存并将其转换为我们的结构，以便更方便地访问其字段。使用类这样做是不行的：类有两个不对外公开的字段：这是 SyncBlockIndex 和虚方法表的地址。如果这两个字段进入非托管代码，这将是非常危险的。因为通过虚方法表，有可能访问任何类型并更改它，对应用程序发起攻击。

让我们证明这只是一段没有任何附加逻辑的内存：

{.wide}
```csharp
unsafe void Main()
{
    int secret = 666;
    HeightHolder hh;
    hh.Height = 5;

    WidthHolder wh;
    unsafe
    {
        // 如果结构中有类型信息，这种转换将无法工作：
        // CLR 在类型转换前会检查继承层次结构，并且在其中找不到 WidthHolder，
        // 将抛出 InvalidCastException。但由于结构只是一段内存，
        // 在 unsafe 世界中没有人阻止你将它解释为任何结构
        wh = *(WidthHolder*)&hh;
    }
    Console.WriteLine("Width: " + wh.Width);
    Console.WriteLine("Secret: " + wh.Secret);
}

struct WidthHolder
{
    public int Width;
    public int Secret;
}

struct HeightHolder
{
    public int Height;
}
```

在这个例子中，我们执行了一个从严格类型化的角度来看是不允许的操作：我们将一个类型转换为了一个不兼容的另一个类型，后者包含一个额外的字段。在 `Main` 方法中，我们引入了一个额外的变量，其值在理论上是秘密的，不应该被读取。但事实并非如此。示例代码在屏幕上自信地显示了 `Main()` 方法中的变量值，尽管它不在任何结构中。你的脸上应该露出微笑，脑海中闪过一句话："天哪，这是一个安全漏洞！"... 但实际上并非如此明显。保护你的代码免受调用非托管代码的影响几乎是不可能的。这主要是因为线程栈的结构，我们稍后会讨论，通过它可以轻松进入被调用的代码，并与局部变量搞鬼。这种类型的攻击通过其他方式进行防御。例如，通过随机化栈帧大小或清除 EBP 寄存器信息来复杂化栈帧的恢复。但我们不会深入讨论这个话题：这是另一个讨论的主题。唯一值得在这个例子中提及的是，为什么尽管变量 `secret` 在变量 `hh` **之前**定义，而在结构 `WidthHolder` 中它位于 **之后**（即在视觉上完全不同的位置），它的值仍然可以被完美地读取。这是因为栈不是从左到右增长，而是相反，从右到左。即，首先声明的变量位于更高的地址上，而后声明的变量位于更早的地址上。

## 调用实例方法的行为

两种数据类型还有另一个有趣的特性，这个特性不是显而易见的，它可以为我们理解两种类型的结构提供更多的光明。

```csharp

// 引用类型的示例
class FooClass 
{
    private int x;

    public void ChangeTo(int val)
    {
        x = val;
    }
}

// 值类型的示例
struct FooStruct
{
    private int x;

    public void ChangeTo(int val)
    {
        x = val;
    }
}

FooClass klass = new FooClass();
FooStruct strukt = new FooStruct();

klass.ChangeTo(10);
strukt.ChangeTo(10);
```

如果逻辑地思考，可以很容易地得出结论，方法的主体只编译一次。即，不是每个类型的实例都编译自己的方法集，这些方法集完全相同。然而，被调用的方法完全知道它是为哪个实例被调用的。这是通过将类型实例的引用作为第一个参数传递来实现的。我们可以轻松地重写我们的示例，这将与上面写的完全相同（我故意不展示虚拟方法的示例。它们的工作方式不同）：

```csharp

// 引用类型的示例
class FooClass 
{
    public int x;
}

// 值类型的示例
struct FooStruct
{
    public int x;
}

public void ChangeTo(FooClass klass, int val)
{
    klass.x = val;
}

public void ChangeTo(ref FooStruct strukt, int val)
{
    strukt.x = val;
}

FooClass klass = new FooClass();
FooStruct strukt = new FooStruct();

ChangeTo(klass, 10);
ChangeTo(ref strukt, 10);
```

值得解释为什么我使用了 `ref` 关键字。如果我不使用它，就会出现一种情况，在这种情况下，我通过方法参数得到了结构的**副本**而不是原
---
layout: post
title: 委托, lambda表达式和事件-C#学习笔记
subtitle: A awesome static site generator.
author: Mikami
categories: csharp
tags: csharp code
top: 1
sidebar: []
---

# 第八章 委托, lambda表达式和事件

​	委托是 .NET 对方法的寻址方式. 它相当于 C/C++ 的函数指针, 但比函数指针更加安全, 用法也略有不同. Lambda 表达式与委托直接相关, 当参数为委托类型时, 可以使用 lambda 表达式实现委托引用的方法.

​	事件一般的思路是通知代码发生了什么. GUI 编程主要处理事件. 在引发事件时, 运行库需要知道应当执行哪个方法, 这就需要把处理事件的方法作为一个参数(实参, argument)传递给委托参数(形参, parameter).

## 委托

​	委托是一种特殊类型的对象, 其包含一个或多个方法的地址. 在 .NET 中, 如果要传递方法, 就必须把方法的细节封装在委托中.

### 声明委托

​	使用类时, 一般情况下有两个步骤:

1. 定义这个类, 即告诉编译器其成员组成
2. 实例化该类的一个对象(除非只使用静态方法)

使用委托时, 也必须经过这两个步骤. 定义委托时要告诉编译器这种类型的委托表示哪种类型的方法, 然后实例化该委托的一个或多个对象来使用委托. 委托的类型安全性非常高, 定义时必须给出方法签名和返回类型.

```C#
public delegate void IntMethodInvoker(int x);
```

​	上述示例定义了一个委托`IntMethodInvoker`, 并指定该委托的每个实例都可以包含一个方法的引用, 其返回值类型为`void`, 接受一个`int`参数.

​	定义一个委托实际上定义了一个新类, 所以**可以在任何可以定义类的地方定义委托, 以任何使用类的方式使用委托**(如声明数组或使用泛型等). 根据可见性和作用域, 可以在委托的定义前使用任意可用的访问修饰符(`public`, `private`, `protected`等).

### 使用委托

​	委托的构造函数总是接受一个参数, 这个参数就是委托引用的方法.

​	下面的代码展示了如何使用委托:

```C#
private delegate string GetAString();
public static void Main()
{
    int x = 40;
    GetAString stringMethod = new GetAString(x.ToString);
    Console.WriteLine($"String is {stringMethod()}");
    //output: 40
}
```

注意, `int.ToString()`是一个实例方法(非静态方法), 需要指定实例`x`和方法名来正确地初始化委托. `stringMethod`在调用时**包含了调用者对象`x`的信息**.
	`stringMethod()`与`stringMethod.Invoke()`完全相同. 实际上, 编译器会把前者替换为后者. 当委托**需要验空**时使用`stringMethod?.Invoke()`在语法上更加简洁.

​	为减少输入量, 在实例化委托对象时可以直接使用方法名称而不使用`new`关键字, 这称为委托推断. 例如下面两个初始化语句是等价的:

```C#
GetAString stringMethod = new GetAString(x.ToString);
GetAString stringMethod = x.ToString;
```

委托推断可以在需要委托实例的任意地方使用, 也可用于事件, 因为事件是基于委托的.

​	**给定委托的实例可以引用任何类型的任何对象上的实例方法或静态方法, 只要方法的返回值类型与参数列表与委托匹配即可.** 委托只关心方法的**返回值类型**与**参数列表**, 只要方法的返回值和参数与生命的委托完全匹配即可用来初始化委托对象. 这意味着委托不关心方法由哪个对象调用, 也不关心方法是实例方法还是静态方法. 

### `Action<T>`和`Func<T>`委托

​	当委托的名称不重要时, 就可以使用`Action<in T>`和`Func<in T, out TResult>`委托.

​	`Action`委托表示引用一个返回类型`void`的方法. 这个委托存在不同变体, 可以传递至多 16 中不同的参数类型. 
​	例如, 没有泛型参数的`Action`委托相当于`delegate void Fn()`, `Action<T>`相当于`delegate void Fn(T)`, `Action<T1, T2>`相当于`delegate void Fn(T1, T2)`, `Action<T1, T2, T3, T4, T5, T6, T7, T8>`相当于`delegate void Fn(T1, T2, T3, T4, T5, T6, T7, T8)`.
​	`Func`委托与`Action`类似, 并且允许指定调用方法的返回类型. `Func`至多可以传递 16 个参数类型和一个返回类型. 
​	`Func<TResult>`相当于`delegate TResult Fn()`, `Func<T, TResult>`相当于`delegate TResult Fn(T)`,  `Func<T1, T2, TResult>`相当于`delegate TResult Fn(T1, T2, TResult)`, 以此类推.

### 多播委托

​	前面使用的每个办法都只包含一个方法调用. 调用委托的次数与调用方法的次数相同. 如果要用这个委托调用多个方法, 可以多次赋值后多次调用. 但是, **委托可以包含多个方法**, 这种委托成为**多播委托**. 一般情况下, 如果调用多播委托, 就可以**按添加方法的顺序**连续调用多个方法, 这是因为目前绝大多数实现都会按照方法添加顺序执行各个方法, 且会保证在第一个方法调用完成之后在调用第二个方法(即使出现延迟或复杂操作).

```C#
using System;
using System.Threading;

class Program
{
    static void MethodA()
    {
        Console.WriteLine("Method A started.");
        Thread.Sleep(1000);  // 模拟延迟
        Console.WriteLine("Method A completed.");
    }

    static void MethodB()
    {
        Console.WriteLine("Method B started.");
        Thread.Sleep(500);   // 模拟延迟
        Console.WriteLine("Method B completed.");
    }

    static void MethodC()
    {
        Console.WriteLine("Method C started.");
        Thread.Sleep(300);   // 模拟延迟
        Console.WriteLine("Method C completed.");
    }

    static void Main()
    {
        Action multicastDelegate = MethodA;
        multicastDelegate += MethodB;
        multicastDelegate += MethodC;

        multicastDelegate.Invoke();  //方法会依次执行, 每个方法完成后才执行下一个
    }
}
```

 输出:

```
Method A started.
Method A completed.
Method B started.
Method B completed.
Method C started.
Method C completed.
```

如上所示, `MethodA` 完全执行完(包括延迟)之后, `MethodB` 才会被调用.
	虽然在 C# 中, 多播委托通常按照添加方法的顺序来调用方法, 但这种行为并不被正式定义为规范的一部分, 因此有可能存在不同的实现或环境差异. 具体来说, 这个行为在不同的版本或实现中可能会有所不同, 特别是在涉及并发或异步的情况下, 可能会改变调用顺序. 虽然多播委托通常保持顺序执行, 但这种行为并没有在在 .NET 文档中明确规范, 因此文档中建议不应依赖于这种顺序. 

​	如果多播委托返回值不为`void`, 那么**多播委托的返回值是由最后被调用的委托方法返回的**, 即调用链中的最后一个方法的返回值将作为整个多播委托的返回值. 

​	多播委托可以使用运算符`+`, `+=`用来添加方法, 也可以使用运算符`-`, `-=`, 用于从委托中删除方法.

​	如果多播委托中的某个方法抛出了异常, 默认的行为是**异常会被传播并终止后续方法的调用**. 这意味着如果委托链中的某个方法抛出异常, 后续的委托方法将不会被执行, 异常会被传递到调用委托的地方.

> 从后台执行的操作来看, 多播委托实际上是一个派生自`System.MulticastDelegate`的类, `System.Multicast.Delegate`又派生自基类`System.Delegate`. `System.MulticastDelegate`的其他成员允许把多个方法调用链接为一个列表.

### `BubbleSorter`示例

​	下面用可定义排序规则的冒泡排序说明委托的用途. `BubbleSorter`类实现一个静态方法`Sort<T>()`方法, 用于排序.

```C#
class BubbleSorter
{
    public static void Sort<T>(IList<T> sortArray, Func<T, T, bool> comparison)
    {
        bool swapped = true;
        do
        {
            swapped = false;
            for (int i = 0; i < sortArray.Count - 1; ++i)
                if (!comparison(sortArray[i], sortArray[i + 1]))
                {
                    T temp = sortArray[i];
                    sortArray[i] = sortArray[i + 1];
                    sortArray[i + 1] = temp;
                    swapped = true;
                }
        } while (swapped);
    }
}
```

## 匿名函数, lambda 表达式

​	可以把匿名函数或 lambda 表达式赋值给委托. 

### 匿名方法

​	C# 2 中引入了匿名方法的语法, 使用`delegate`关键字创建匿名函数:

```C#
//var在这里实际是 Func<string, int>?
var Fn1 = delegate(string param)
{
    int result = 0;
    //do something
    return result;
};
```

使用`delegate`关键字创建的匿名函数会根据`return`语句推断函数返回值, 可以不用显式声明.

​	从 C# 3.0 开始, 可以使用 lambda 表达式实现匿名函数. 

### lambda 表达式

​	使用`=>`运算符定义 lambda 表达式. `=>`左边列出需要的参数, 右边定义了方法的实现代码. 

* 参数只有一个时, 只写出参数名即可(若显式指定类型则需要放在圆括号中), 如果有两个或以上参数, 则必须将参数放在圆括号中. 
* 如果在定义的同时赋值给确定的委托, 则无需写出参数类型, 因为可以根据委托的签名推断出参数类型; 否则需要显式指定参数类型. 
* 如果函数体只有一条语句, 可以省略花括号.
  * 如果这一条语句是`return`语句, 则省略花括号与`return`关键字直接在`=>`右侧写出要返回的值;
  * 如果这一条语句不是`return`语句, 则直接在`=>`右侧写出该语句.

```C#
Func<string, string> expr1 = param =>
{
    param += "end of string.";
    return param;
}
var expr2 = (int val) => val * 2;
Func<int, int, int> expr3 = (val1, val2) => val1 + val2;
var expr4 = (string val1, string val2) => Console.WriteLine(val1 + val2);
// 上面的 var 是 Action<string, string>?
```

C# 中不允许在 lambda 表达式本身像 C++ 中那样使用`->`语法显式指定返回类型, 只能通过`return`的值推断返回类型.

### 闭包

​	在编程中, **闭包(Closure)**是一个函数或 lambda 表达式与其相关联的**环境**(即该函数的外部变量)的组合. 换句话说, 闭包不仅仅是函数本身, 还包括了它可以访问的变量的状态. 简单来说, 闭包是一个能够“记住”并访问其词法作用域(定义时的作用域)内的变量的函数. 

​	lambda 表达式可以捕获并访问 lambda 方法体外的变量, 这就是 C# 中的闭包. 

```C#
int someVal = 5;
Action<int> f = x => someVal += x;
f(2);	//after this, someVal is 7
f(3);	//after this, someVal is 10
Console.WriteLine(someVal);	//output: 10
```

> 当 C# 编译器处理 lambda 表达式时, 它会执行以下关键步骤: 
>
> 1. 转换为匿名方法: 编译器将 lambda 表达式转换为匿名方法或内联函数. 
> 2. 创建委托类型: lambda 表达式被绑定到一个特定的委托类型. 
> 3. 捕获外部变量(闭包): 如果 lambda 表达式引用了外部变量, 编译器会创建一个隐藏类来捕获这些变量(可能是将这些变量作为隐藏类的成员). 
> 4. 生成委托实例: 将 lambda 表达式转换为委托类型的实例, 以便在需要时调用. 
> 5. 优化(可能): 对于简单的 lambda 表达式, 编译器可能进行内联优化, 减少运行时开销. 

## 事件

​	事件基于委托, 为委托提供了一种发布/订阅机制. 它用于**通知对象某个操作已经发生或即将发生**. 事件可以**被其他对象订阅, 以便在事件发生时被通知**. 当一个事件被触发时, 它会**调用所有已经订阅它的委托**. 

​	C# 中的事件由以下三个部分组成:

- 事件发布者: 该对象定义了事件, 当事件发生时, 它会通知所有已经订阅该事件的对象. 
- 事件参数: 事件发生时需要传递的信息, 可以是任何类型的对象. 如果事件不需要传递参数, 则可以使用`EventArgs`(实参使用`EventArgs.Empty`字段). 
- 事件订阅者: 该对象订阅了事件, 并在事件发生时执行相应的操作. 

​	下面先给出一段示例代码, 再逐一解释其组成成分. 在该示例中, 我们定义了一个简单的 Calculator 类, 它可以对两个数进行加, 减, 乘, 除操作, 并在操作完成时触发一个事件. 

```C#
using System;

public class Calculator
{
    // 定义一个事件
    public event EventHandler<CalculationEventArgs> CalculationPerformed;
	// 调用事件委托
    protected virtual void OnCalculationPerformed(CalculationEventArgs e)
    {
        // 若事件未被任何对象订阅则 CalculationPerformed 为 null
        // 所以需要先验空
        CalculationPerformed?.Invoke(this, e);
    }
    
    public int Add(int x, int y)
    {
        int result = x + y;
        // 触发该事件
        OnCalculationPerformed(new CalculationEventArgs("add", x, y, result));
        return result;
    }

    public int Subtract(int x, int y)
    {
        int result = x - y;
        // 触发该事件
        OnCalculationPerformed(new CalculationEventArgs("subtract", x, y, result));
        return result;
    }

    public int Multiply(int x, int y)
    {
        int result = x * y;
        // 触发该事件
        OnCalculationPerformed(new CalculationEventArgs("multiply", x, y, result));
        return result;
    }

    public int Divide(int x, int y)
    {
        int result = x / y;
        // 触发该事件
        OnCalculationPerformed(new CalculationEventArgs("divide", x, y, result));
        return result;
    }

}
// 事件参数, 包含事件发生需要传递的信息
public class CalculationEventArgs : EventArgs
{
    public string Operation { get; set; }
    public int X { get; set; }
    public int Y { get; set; }
    public int Result { get; set; }

    public CalculationEventArgs(string operation, int x, int y, int result)
    {
        Operation = operation;
        X = x;
        Y = y;
        Result = result;
    }
}

public class Program
{
    static void Main(string[] args)
    {
        Calculator calculator = new Calculator();

        // 订阅事件
        // 此处 OnCalculationPerformed 为 Program.OnCalculationPerformed
        // 而不是 Calculator.OnCalculationPerformed
        calculator.CalculationPerformed += OnCalculationPerformed;

        int x = 10;
        int y = 5;

        int result = calculator.Add(x, y);
        Console.WriteLine("{0} + {1} = {2}", x, y, result);

        result = calculator.Subtract(x, y);
        Console.WriteLine("{0} - {1} = {2}", x, y, result);

        result = calculator.Multiply(x, y);
        Console.WriteLine("{0} * {1} = {2}", x, y, result);

        result = calculator.Divide(x, y);
        Console.WriteLine("{0} / {1} = {2}", x, y, result);

        // 取消订阅事件
        calculator.CalculationPerformed -= OnCalculationPerformed;
    }

    static void OnCalculationPerformed(object sender, CalculationEventArgs e)
    {
        Console.WriteLine("{0} {1} {2} = {3}", e.X, e.Operation, e.Y, e.Result);
    }
}
```

### `EventHandler`与`EventHandler<TEventArgs>`委托

​	`EventHandler`与`EventHandler<TEventArgs>`是用于事件处理的标准委托类型. 前者用于处理无需事件数据的委托, 后者用于处理需要事件数据, `TEventArgs`是事件数据类. 二者声明如下:

```C#
public delegate void EventHandler(object sender, EventArgs e);
public delegate void EventHandler<TEventArgs>(object sender, TEventArgs e) where TEventArgs : EventArgs;
```

其中`EventArgs`是所有事件数据类的基类, 用于在事件没有附加数据时提供事件数据的空值; 泛型参数`TEventArgs`是事件数据类, 必须派生自`EventArgs`.

​	上述委托的第一个参数`sender`是触发事件的对象, 通常是`this`. 第二个参数`e`是事件数据的一个实例, 其中包含了与本次事件相关的附加信息.

### 定义一个事件

​	因为事件基于委托, 所以需要被声明为事件的是一个委托. 事件可以使用访问限制修饰符, 一般情况下将事件声明为`public`.

```C#
public event EventHandler<CalculationEventArgs> CalculationPerformed;
```

​	根据事件的复杂程度, 在定义事件时可以选择使用`EventHandler`或`EventHandler<TEventArgs>`其一作为事件的基础委托; 如果事件需要传递的参数较为简单或不适合封装到`EventArgs`或派生类中, 也可以使用自定义委托来定义事件. 使用自定义委托定义事件需要注意以下要求:

* 委托**返回类型必须是`void`**
* 委托的参数可以自定义, 但通常包含与事件相关的信息, 如本次事件的发布者等

一般情况下应当使用`EventHandler`或`EventHandler<TEventArgs>`委托来声明事件, 二者使得事件具有统一性与较好的可拓展性.

​	如果用上述一行代码来声明事件, C# 会自动为事件提供默认的`add`和`remove`访问器, 当有订阅者订阅事件时, C# 会自动为`CalculationPerformed`事件注册和取消注册`EventHandler<CalculationEventArgs>`委托. 
​	如果需要对事件更灵活的操作, 则应该像使用属性一样先定义一个委托变量, 再用事件的`add`和`remove`方法添加和删除委托:

```C#
private EventHandler<CalculationEventArgs> _calculationPerformed;
public event EventHandler<CalculationEventArgs> CalculationPerformed
{
    add => _calculationPerformed += value;
    remove => _calculationPerformed -= value;
}
```

这非常类似于自动属性和完整属性间的关系.
	在一些场景下, 较长的写法是必须的, 因为第一种写法只能简单地向事件中注册和取消注册委托. 下面是一些例子:

* 验证订阅者或进行限制(如限制订阅次数)
* 控制事件的订阅/取消订阅行为(如线程安全, 排序)
* 修改事件的访问级别或增加额外的逻辑

​	下面演示了如何防止订阅者的重复订阅:

```C#
public event EventHandler<CalculationEventArgs> CalculationPerformed
{
    add
    {
        if (!_calculationPerformed.GetInvocationList().Contains(value))
        {
            _calculationPerformed += value;
        }
    }
    remove => _calculationPerformed -= value;
}
```

### 事件数据

​	事件数据封装了与事件相关的信息, 并在事件触发时的以参数的形式传递给事件订阅者.

​	如果使用`EventHandler`委托来定义事件, 则表示无需额外的事件数据, 事件数据参数一般使用`EventArgs.Empty`. 例如, 当初发一个简单的按钮点击事件时, 只需事件本身传递"进行了一次点击"这一信息, 而无需额外的事件数据, 这时可以使用`EventArgs.Empty`作为事件数据.

​	如果使用`EventHandler<TEventArgs>`委托定义事件, 则通常需要一个派生自`EventArgs`的自定义类以包含事件相关信息. 例如示例代码中的`CalculationEventArgs`类即为事件数据类, 其中包含了一次计算事件的运算数, 运算类型和预算结果.

### 在事件发布者类中触发事件

​	现在我们已经声明了一个事件(记住**其本质是一个委托**), 下面我们需要使事件发布者能够在进行相应操作时能够触发该事件. 触发该事件的本质就是**调用委托来使得所有订阅该事件的类的*处理该事件*的方法被执行**.

​	示例代码中`Calculator.OnCalculationPerformed`方法负责在发布者中触发该事件. 在负责运算的方法中, 每次计算执行后都会调用该方法来触发本次事件. 示例中也可以直接使用

```C#
CalculationPerformed?.Invoke(this, new CalculationEventArgs("multiply", x, y, result));
```

来代替方法中触发事件的

```C#
OnCalculationPerformed(new CalculationEventArgs("multiply", x, y, result));
```

语句, 但若触发事件时有更复杂的操作(如事件数据需要进行进一步运算或过滤), 则将触发事件的语句封装到一个方法中可以显著提高代码可读性.
	触发事件时应当考虑**检验事件是否为空**, 通常的方法是使用`.?`运算符, 像示例代码那样调用`Invoke`方法. 如果不进行验空, 当没有任何类订阅该事件而发布者却触发了该事件时, 就会导致试图调用`null.Invoke()`从而抛出`NullReferenceException`异常.

### 编写事件处理方法并订阅事件
​	有了前面的步骤, 事件发布者的部分已经编写完成, 现在需要在订阅者类中编写处理事件的方法并使用`+=`运算符将该方法注册到发布者的事件中(即订阅事件).

​	首先在订阅者类中编写事件处理方法, 该方法的签名必须与事件的委托类型匹配, 如示例代码中的`Program.OnCalculationPerformed`方法:

```C#
static void OnCalculationPerformed(object sender, CalculationEventArgs e)
{
	Console.WriteLine("{0} {1} {2} = {3}", e.X, e.Operation, e.Y, e.Result);
}
```

​	有了该事件处理方法就可以用该方法来注册到事件. 在`Main`方法或其他地方实例化发布者和订阅者(如果有需要), 并使用`+=`运算符订阅事件:

```C#
calculator.CalculationPerformed += OnCalculationPerformed;
```

注意这里实质上是把订阅者的方法添加到发布者的事件委托中, 所以是`发布者的事件 += 订阅者的方法`. 完成订阅后, **在发布者触发事件后, 发布者事件的委托会依次执行每个订阅事件的方法**, 这也是 C# 事件工作的本质.

​	如果需要取消订阅, 可以使用`-=`运算符来将某个方法从事件中取消订阅:

```C#
calculator.CalculationPerformed -= OnCalculationPerformed;
```

​	`-=`运算符在删除一个未订阅的委托时是安全的, 也就是说如果试图使用 `-=` 运算符删除一个未订阅的委托不会抛出异常, 而是简单地忽略这个操作, 事件委托列表不会发生任何变化.

## assert函数

C语言和C++都有一个专为条调试而准备的工具函数，就是assert函数

这个函数定义在C与汉的assert.h库文件里，包含到C++程序里为我们用以下语句

`#include <cassert>`

assert（）函数需要有一个参数，它将测试这个输入参数的真or假状态

如果为真，不做任何事情

如果为假，报错

我们可以利用它在某个程序里的关键假设不成立时立刻停止该程序执行并报错，从而避免发生更严重的问题

另外除了结合assert（）函数，我们还可以使用cout语句来报告程序里正在发生的事情

## 捕获异常

异常处理是C++中处理运行时错误的重要机制，它允许程序在遇到错误时能够优雅地恢复或终止，而不是直接崩溃。

### 基本概念

C++异常处理基于三个关键字：

- `throw`: 用于抛出异常
    
- `try`: 用于标识可能抛出异常的代码块
    
- `catch`: 用于捕获和处理特定类型的异常
    

#### 基本语法

```C++
try {
    // 可能抛出异常的代码
    throw someException;
} catch (ExceptionType1& e) {
    // 处理ExceptionType1类型的异常
} catch (ExceptionType2& e) {
    // 处理ExceptionType2类型的异常
} catch (...) {
    // 处理所有其他类型的异常
}
```

#### 异常类型

- 标准异常: 继承自`std::exception`，如`std::runtime_error`, `std::logic_error`等
    
- 自定义异常: 用户自定义的异常类，通常继承自`std::exception`
    
- 基本类型: 可以抛出任何类型，包括基本数据类型（int, char等）
    

#### 捕获顺序

catch块是按照它们出现的顺序进行匹配的，因此应该从最具体到最一般的顺序排列：

```JavaScript
try {
    // ...
} catch (const MyCustomException& e) {
    // 先捕获最具体的异常
} catch (const std::runtime_error& e) {
    // 然后捕获较一般的异常
} catch (const std::exception& e) {
    // 最后捕获最一般的异常
} catch (...) {
    // 捕获所有其他异常
}
```

#### 异常说明符

C++11引入了`noexcept`说明符，表示函数不会抛出异常：

```C++
void safeFunction() noexcept {
    // 这个函数保证不会抛出异常
}

void mayThrow() noexcept(false) {
    // 这个函数可能抛出异常
}
```
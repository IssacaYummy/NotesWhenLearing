虚方法是 C++ 多态性的核心机制，它允许在运行时根据对象的实际类型调用正确的函数版本。

它允许子类（派生类）重写父类（基类）的方法，并且在通过基类的指针或引用来调用该方法时，程序会根据指针或引用实际指向的对象类型来决定调用那个版本的方法，而不是根据指针或引用本身的类型

虚方法让程序在运行时（而不是编译时）决定要执行哪个函数，这个特性被称为动态绑定或晚期绑定

## 定义

在基类成员函数声明前加上`virtual`关键字

```C++
class Base
{
public:
    virtual void show() // 虚方法
    {
        cout << "Base Show\n";    
    }
};
```

派生类可以提供自己的实现版本

```C++
class Derived : public Base{
public:
    void show() override { //覆盖（override）关键字（C++11）不是必须的，但强烈推荐使用
    cout << "Derived show\n";
    }
};
```

## 典型示例

假设我们有一个图形处理程序，有一个基类 `Shape` 和两个派生类 `Circle` 和 `Rectangle`。

没有虚方法时的问题：

```C++
#include <iostream>
using namespace std;

class Shape {
public:
    void draw() {
        cout << "Drawing a generic shape." << endl;
    }
};

class Circle : public Shape {
public:
    void draw() { // 注意：这里没有 ‘virtual‘ 关键字
        cout << "Drawing a circle." << endl;
    }
};

class Rectangle : public Shape {
public:
    void draw() { // 注意：这里没有 ‘virtual‘ 关键字
        cout << "Drawing a rectangle." << endl;
    }
};

int main() {
    Circle circle;
    Rectangle rect;

    // 情况1：直接通过对象调用，没问题
    circle.draw(); // 输出: Drawing a circle.
    rect.draw();   // 输出: Drawing a rectangle.

    // 情况2：通过基类指针调用 —— 问题出现了！
    Shape* shapePtr1 = &circle;
    Shape* shapePtr2 = &rect;

    shapePtr1->draw(); // 输出: Drawing a generic shape. (但我们期望的是 Circle)
    shapePtr2->draw(); // 输出: Drawing a generic shape. (但我们期望的是 Rectangle)

    return 0;
}
```

在情况2中，虽然 `shapePtr1` 实际指向一个 `Circle` 对象，但编译器在编译时看到 `shapePtr1` 的类型是 `Shape*`，于是就调用了 `Shape::draw()`。这就是静态绑定——调用在编译时就已经确定了。

这显然不是我们想要的行为。我们希望的是：“如果指向的是圆，就画圆；如果指向的是矩形，就画矩形”。

  

## 解决方法

在基类的成员函数前加上 `virtual` 关键字。

```C++
#include <iostream>
using namespace std;

class Shape {
public:
    virtual void draw() { // 关键：声明为 virtual
        cout << "Drawing a generic shape." << endl;
    }
};

class Circle : public Shape {
public:
    void draw() override { // override 关键字（C++11）不是必须的，但强烈推荐使用
        cout << "Drawing a circle." << endl;
    }
};

class Rectangle : public Shape {
public:
    void draw() override {
        cout << "Drawing a rectangle." << endl;
    }
};

int main() {
    Circle circle;
    Rectangle rect;

    // 情况1：直接调用，行为不变
    circle.draw(); // 输出: Drawing a circle.
    rect.draw();   // 输出: Drawing a rectangle.

    // 情况2：通过基类指针调用 —— 多态生效了！
    Shape* shapePtr1 = &circle;
    Shape* shapePtr2 = &rect;

    shapePtr1->draw(); // 输出: Drawing a circle. (正确!)
    shapePtr2->draw(); // 输出: Drawing a rectangle. (正确!)

    // 情况3：通过基类引用调用，同样有效
    Shape& shapeRef = circle;
    shapeRef.draw(); // 输出: Drawing a circle.

    return 0;
}
```

由于 `Shape::draw()` 被声明为 `virtual`，当通过基类指针或引用调用 `draw()` 时，程序会在运行时检查指针/引用实际指向的对象的类型（是 `Circle` 还是 `Rectangle`），并调用该类型对应的 `draw` 方法。这就是动态绑定。

  

## 底层原理

通过两个核心机制：

1. 虚函数表（Virtual Table, vtable）：
    
    1. 编译器会为每一个包含虚函数的类（或者从包含虚函数的类派生而来的类）创建一个虚函数表。
        
    2. 这个表就像一个数组，里面存储了该类所有虚函数的地址。
        
    3. 例如，`Shape` 类的 vtable 包含 `&Shape::draw`。`Circle` 类的 vtable 包含 `&Circle::draw`。
        
2. 虚函数指针（Virtual Pointer, vptr）：
    
    1. 编译器会向每一个包含虚函数的类的对象中，悄悄地添加一个隐藏的成员指针，就是 vptr。
        
    2. 这个 vptr 在对象构造时被设置，指向该对象所属类的虚函数表。
        
    3. 一个 `Circle` 对象的 vptr 指向 `Circle` 的 vtable，一个 `Rectangle` 对象的 vptr 指向 `Rectangle` 的 vtable。
        

调用过程： 当执行 `shapePtr->draw()` 时，编译器会生成类似下面的代码：

1. 通过 `shapePtr` 找到对象的 vptr。
    
2. 通过 vptr 找到对应的 vtable。
    
3. 在 vtable 中找到 `draw` 函数的地址。
    
4. 使用找到的地址来调用函数。
    

这个过程在运行时完成，因此能够根据对象的实际类型调用正确的函数。这就是多态的开销所在（一次额外的指针寻址）

  

## 一些细节与特性

1. 析构函数必须是虚函数
    

如果一个类可能被继承（即作为基类），并且会通过基类指针来删除派生类对象，那么它的析构函数必须声明为析构函数

```C++
class Shape {
public:
    virtual ~Shape() { // 虚析构函数
        cout << "Shape destructor." << endl;
    }
};

class Circle : public Shape {
public:
    ~Circle() override {
        cout << "Circle destructor." << endl;
    }
};

int main() {
    Shape* shape = new Circle();
    delete shape; // 正确！会先调用 ~Circle()，再调用 ~Shape()
    return 0;
}
```

如果析构函数不是虚的，那么 `delete shape;` 就只会调用 `Shape` 的析构函数，导致 `Circle` 的析构函数不会被调用，从而发生资源泄漏。

2. `override` 关键字（C++11）
    
    1. 它不是必须的，但强烈推荐使用。
        
    2. 它明确地告诉编译器和阅读代码的人：“我意图重写一个虚函数”。
        
    3. 如果加上 `override` 但基类中没有相同签名的虚函数，编译器会报错。这可以防止因拼写错误或参数列表不匹配而意外创建新函数的情况，大大提高了代码的安全性。
        
3. `final`关键字（C++11）
    
    1. 可以用于类表示该类不能被继承
        

`class MyFinalClass final { ... };`

- 可以用于虚函数，表示该虚函数在派生类中不能被重写
    

`virtual void draw() const final;`

4. 纯虚函数和抽象类
    
    1. 如果一个虚函数后面加上`= 0`它就变成了纯虚函数。他只有声明，没有定义（但C++11后可以有默认实现）
        
    
    ```C++
    class Shape {
    public:
        virtual void draw() = 0; // 纯虚函数
    };
    ```
    

- 包含至少一个纯虚函数的类称为抽象基类。
    
- 不能创建抽象基类的对象。`Shape s;` 会导致编译错误。
    
- **它的作用是为所有派生类定义一个统一的接口。任何派生类必须重写所有纯虚函数，否则它自己也会成为抽象类。**
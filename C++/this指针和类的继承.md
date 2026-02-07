# this指针

用法一：当类中的属性和构造函数中传入的参数相同时，可以通过this指针来区分

```C++
class Human
{
    char Yummy;
    Human(char Yummy);
}
Human::Human(char Yummy)
{
    Yummy = Yummy;
}
```

在 Yummy = Yummy 之前所有的语法都没有任何问题：

- Human()构造函数中有一个名为Yummy的参数
    
- 虽然他与Human类里面的属性同名，但却是不相关的两样东西
    

通过使用this指针指向当前类生成的对象的属性来区分,即

this -> Yummy = Yummy;

这样，赋值操作符左边将被解释为当前对象的属性，右边被解释为构造函数传入的参数

使用this指针的基本原则：代码存在二义性隐患

# 类的继承

继承机制使得程序员可以创建一个类的堆叠层次结构，每个子类均将继承在它的基类里定义的方法和属性

通过继承机制，程序员可以对现有代码进行进一步拓展，并应用在新的程序中

## 基类和子类

基类：基类是可以派生出其他的类，也称为父类或超类

子类：子类是从基类里派生出来的类

继承关系的C++描述如下：

class SubClass: public SuperClass { ... };

例如：class Pig: public Animal;

示例代码：

```C++
#include <iostream>
#include <string>

class Animal
{
private:
    /* data */
public:
    std::string mouth;

    void eat();
    void sleep();
    void droll();

    Animal(/* args */);
    ~Animal();
};

Animal::Animal(/* args */)
{
}

Animal::~Animal()
{
}

class Pig : public Animal
{
private:
    /* data */
public:
    void climb();

    Pig(/* args */);
    ~Pig();
};

Pig::Pig(/* args */)
{
}

Pig::~Pig()
{
}

class Turtle : public Animal
{
private:
    /* data */
public:
    void swim();

    Turtle(/* args */);
    ~Turtle();
};

Turtle::Turtle(/* args */)
{
}

Turtle::~Turtle()
{
}

void Animal::eat()
{
    std::cout << "Eating..." << "\n" << std::endl;
}

void Animal::sleep()
{
    std::cout << "Sleeping..." << "\n" << std::endl;
}

void Animal::droll()
{
    std::cout << "Drooling.." << "\n" << std::endl;
}

void Pig::climb()
{
    std::cout << "Climbing..." << "\n" << std::endl;
}

void Turtle::swim()
{
    std::cout << "Swimming..." << "\n" << std::endl;   
}

int main()
{
    Pig pig;
    Turtle turtle;

    pig.climb();
    turtle.swim();
    
    return 0;
}
```

## 继承机制中的构造器和析构器

原则：基类必须在子类之前初始化

在继承机制中应当这样定义：

```C++
Animal::Animal( std::string theName )
{
    name = theName;
}
Pig::Pig( std::string theName ):Animal(theName)
{
    
}
```

注意在子类的构造器定义里的“:Animal(theName)"的语法含义是： 当调用Pig()构造器时（以theName为输入参数），Animal( )构造器也将被调用（theName输入参数将传递给它）

即，theName将同时传递给Pig( )和Animal( )，赋值动作实际发生在Animal( )方法里

在销毁某个对象时，基类的析构器也将自动调用，但基类的析构器将在子类的最后一条语句执行完毕后才被调用

以下例子很好的说明了各构造器和析构器的调用顺序

```C++
#include <iostream>
#include <string>

class BaseClass
{
    public:
        BaseClass();
        ~BaseClass();

        void DoSomething();

};

class SubClass : public BaseClass
{
    public:
        SubClass();
        ~SubClass();
};

BaseClass::BaseClass()
{
    std::cout << "Into BaseClass Construct Function..." << "\n" << std::endl;
    std::cout << "Don Something in BaseClass Construct Function... " << "\n\n" << std::endl;
}

BaseClass::~BaseClass()
{
    std::cout << "Into BaseClass DeConstruct Function..." << "\n" << std::endl;
    std::cout << "Don Something in BaseClass DeConstruct Function... " << "\n\n" << std::endl;
}

void BaseClass::DoSomething()
{
    std::cout << "Do Sonmething ...\n\n" << std::endl;
}

SubClass::SubClass()
{
    std::cout << "Into SubClass Construct Function...\n\n" << std::endl;   
}

SubClass::~SubClass()
{
    std::cout << "Into SubClass Deconstruct Functino...\n\n" << std::endl;
}

int main()
{
    SubClass subclass;
    subclass.DoSomething();

    std::cout << "OVER!\n\n" << std::endl;

    return 0;
}
```

使用类的原则：

1. 基类和子类之间的关系应该自然和清晰
    
2. 构造器的设计越简明越好，应只用他来初始化各种有关的属性
    
3. 设计、定义和使用一个类的时候应该让他的每个组成部分简单到不能再简单
    
4. 析构器的基本用途时对前面所做的事情进行清理，尤其是在使用了动态内存的程序里，析构器将至关重要
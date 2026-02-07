`const` 关键字用于定义常量、保护数据不被修改，并在编译时提供类型安全检查

## 1.基本概念
    

const关键字用于指定一个对象或变量不能被修改。一旦声明为const，任何试图修改该变量的操作都会导致编译错误

## 2.const与变量

### 2.1 基本数据类型与常量

```C++
    const int days_in_week = 7;
    const float pi = 3.14159f;
    const char newline = '\n';
```
### 2.2 常量必须初始化
const变量必须在声明时初始化，否则会导致编译错误

## 3.const与指针
### 3.1 指向常量的指针
      指针指向的内容是常量，不能通过指针修改内容
    
```C++
    int value = 10;
    const int* ptr = &value;  // ptr 指向一个常量整数
    // *ptr = 20;             // 错误：不能通过 ptr 修改值
    value = 20;               // 正确：可以直接修改值
```
###   3.2 常量指针

      指针本身是常量，不能指向其他地址
```C++
    int value1 = 10, value2 = 20;
    int* const ptr = &value1;  // ptr 是一个常量指针
    *ptr = 30;                 // 正确：可以修改指向的值
    // ptr = &value2;          // 错误：不能修改指针的指向
```
 
###   3.3 指向常量的常量指针
      指针本身和指向的内容都是常量
    
```C++
    int value = 10;
    const int* const ptr = &value;  // ptr 是指向常量的常量指针
    // *ptr = 20;                   // 错误：不能修改指向的值
    // ptr = &some_other_value;     // 错误：不能修改指针的指向
```
    

## 4 const与函数
    
###   1.const函数参数
        
    
      使用从const保护传入函数的参数不被修改
    
    ```C++
    void printString(const string& str) {
        // str[0] = 'A';  // 错误：不能修改 const 引用
        cout << str << endl;
    }
    
    void processArray(const int arr[], int size) {
        // arr[0] = 10;   // 错误：不能修改 const 数组
        for (int i = 0; i < size; i++) {
            cout << arr[i] << " ";
        }
    }
    ```
    
    2. ###   const函数返回值
        
    
      返回const值可以防止返回值被修改
    
    ```C++
    const int getMaxValue() {
        return 100;
    }
    
    // getMaxValue() = 200;  // 错误：不能修改 const 返回值
    ```
    
    3. ###   const成员函数
        
    
      const成员函数承诺不会修改类的成员变量
    
    ```C++
    class MyClass {
    private:
        int value;
        mutable int counter;  // 即使在 const 函数中也可以修改
    
    public:
        MyClass(int v) : value(v), counter(0) {}
        
        // const 成员函数
        int getValue() const {
            // value = 10;      // 错误：不能修改成员变量
            counter++;          // 正确：counter 是 mutable 的
            return value;
        }
        
        // 非 const 成员函数
        void setValue(int v) {
            value = v;
        }
    };
    
    int main() {
        const MyClass obj(5);
        cout << obj.getValue() << endl;  // 正确：可以调用 const 成员函数
        // obj.setValue(10);             // 错误：不能调用非 const 成员函数
    }
    ```
    

## 5.const与类
    
###   1.const对象
    
      const对象只能调用const成员函数
    
```C++
    class Rectangle {
    private:
        double width, height;
    
    public:
        Rectangle(double w, double h) : width(w), height(h) {}
        
        double area() const {
            return width * height;
        }
        
        void resize(double w, double h) {
            width = w;
            height = h;
        }
    };
    
    int main() {
        const Rectangle rect(5.0, 3.0);
        cout << rect.area() << endl;  // 正确：area() 是 const 成员函数
        // rect.resize(10.0, 5.0);    // 错误：resize() 不是 const 成员函数
    }
```
    
###   2.const数据成员
    
      const数据成员必须在构造函数初始列表中初始化
    
```C++
    class Circle {
    private:
        const double pi;
        double radius;
    
    public:
        Circle(double r) : pi(3.14159), radius(r) {}  // pi 在初始化列表中初始化
        
        double area() const {
            return pi * radius * radius;
        }
    };
```
    
###   3.mutable关键字
    
      mutable允许在const成员中修改特定的成员变量：
    
```C++
    class Logger {
    private:
        mutable int logCount;  // 即使在 const 函数中也可以修改
    
    public:
        Logger() : logCount(0) {}
        
        void log(const string& message) const {
            cout << message << endl;
            logCount++;  // 正确：logCount 是 mutable 的
        }
        
        int getLogCount() const {
            return logCount;
        }
    };
```

## 6.const与引用
    
###   1.const引用
    
      const引用可以绑定到临时对象和右值，并且可以延长临时对象的生命周期
    
```C++
    void print(const string& str) {
        cout << str << endl;
    }
    
    int main() {
        string s1 = "Hello";
        print(s1);               // 正确：绑定到左值
        print("World");          // 正确：绑定到右值
        
        const int& ref = 42;     // 正确：const 引用可以绑定到右值
        // int& ref2 = 42;       // 错误：非 const 引用不能绑定到右值
    }
```

    
###   2.const引用作为函数参数
        
    
      使用const引用作为函数参数可以避免不必要的拷贝
    
```TypeScript
    // 高效：避免字符串拷贝
    void processLargeObject(const string& largeStr) {
        // 处理大对象，但不修改它
    }
    
    // 低效：会产生字符串拷贝
    void processLargeObject(string largeStr) {
        // 处理大对象的副本
    }
```
    

## 7.constexpr（C++11起）
    

`constexpr` 是 C++11 引入的关键字，全称为constant expression，用于指示一个值或函数可以在编译时就被计算出来。

它的核心目的是将计算从运行时提前到编译时，从而提升程序的运行效率。

声明一个变量为 `constexpr`，意味着该变量是一个常量，并且其值必须由编译器在编译时就能确定。

声明一个函数为 `constexpr`，意味着如果其参数是编译时常量，那么函数的结果也可以在编译时计算出来。它也可以用于运行时计算。

```C++
constexpr int square(int x) {
    return x * x;
}

constexpr int max_size = 100;
constexpr int squared = square(5);  // 在编译时计算

int arr[squared];  // 正确：squared 是编译时常量
```

## 8.const 与类型推导
    
###   1.auto 与 const
        
    
```C++
    const int x = 10;
    auto y = x;           // y 的类型是 int（去掉了 const）
    const auto z = x;     // z 的类型是 const int
    
    const int& ref = x;
    auto a = ref;         // a 的类型是 int
    const auto& b = ref;  // b 的类型是 const int&
```
    
###   2.decltype 与 const
        
    
```C++
    const int x = 10;
    decltype(x) y = 20;   // y 的类型是 const int
    
    int a = 5;
    const int& ref = a;
    decltype(ref) b = a;  // b 的类型是 const int&
```
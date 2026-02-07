下面的内容截取于deepseek

`new` 和 `delete` 是 C++ 中用于动态内存管理的核心运算符，它们提供了比 C 语言的 `malloc()` 和 `free()` 更安全、更面向对象的内存管理方式。

## new运算符

### 功能

- 在堆内存上动态分配内存
    
- 返回指向分配内存的指针
    
- 自动调用对象的构造函数（对于类类型）
    

### 使用方式

1. #### 分配单个对象
    

```Go
int* p = new int;      // 分配未初始化的 int
int* p2 = new int(42); // 分配并初始化为 42
MyClass* obj = new MyClass(); // 调用默认构造函数
```

2. 分配数组
    

```C++
int* arr = new int[10];        // 分配 10 个 int 的数组
MyClass* objs = new MyClass[5]; // 调用 5 次默认构造函数
```

3. 初始化列表
    

```C++
int* p = new int{42};          // 单个对象初始化
int* arr = new int[4]{1,2,3,4}; // 数组初始化
```

4. 不抛出异常的版本
    

```C++
int* p = new(std::nothrow) int[1000000];
if (p == nullptr) {
    // 处理内存分配失败
}
```

## delete运算符

### 功能

- 释放由 `new` 分配的内存
    
- 自动调用对象的析构函数（对于类类型）
    
- 将内存返还给系统
    

### 使用方式

1. 释放单个对象
    

```C++
delete p;   // 释放单个对象
delete obj; // 调用析构函数
```

2. 释放数组
    

```C++
delete[] arr;  // 释放数组
delete[] objs; // 调用每个元素的析构函数
```

## 底层操作

### `new` 的底层操作

1. 调用 `operator new` 分配内存
    
2. 在分配的内存上调用构造函数
    
3. 返回指向新对象的指针
    

### `delete` 的底层操作

1. 调用对象的析构函数
    
2. 调用 `operator delete` 释放内存
    

## 与C语言内存函数的对比

|   |   |   |
|---|---|---|
|特性|C++ (new/delete)|C (malloc/free)|
|内存分配|运算符|函数|
|构造/析构调用|自动调用|不调用|
|类型安全|是（返回正确类型指针）|否（返回 void*）|
|内存不足处理|抛出 bad_alloc 异常|返回 NULL|
|计算对象大小|自动计算|需手动 sizeof|
|初始化|支持初始化|需额外 memset|
|重载能力|支持类特定重载|不可重载|

## 注意事项

1. ### 匹配使用
    

必须严格匹配分配和释放方式

```C++
int* p = new int;      // 正确释放方式: delete p;
int* arr = new int[5]; // 正确释放方式: delete[] arr;

// 错误示例:
// delete[] p;   // 未定义行为
// delete arr;   // 只释放第一个元素，内存泄漏
```

2. ### 空指针安全
    

删除空指针是安全的

```C++
int* p = nullptr;
delete p; // 安全，无操作
```

3. ### 析构函数调用
    

```C++
class ResourceHolder {
    int* resource;
public:
    ResourceHolder() : resource(new int[100]) {}
    ~ResourceHolder() { delete[] resource; } // 必须释放资源
};

ResourceHolder* rh = new ResourceHolder();
delete rh; // 自动调用析构函数释放内部资源
```

4. ### 忘记调用delete会造成内存泄漏风险
    

5. ### 自定义内存管理
    

可以重载特定的new和delete

```C++
class MyClass {
public:
    // 自定义 new
    void* operator new(size_t size) {
        std::cout << "分配 " << size << " 字节\n";
        return ::operator new(size);
    }
    
    // 自定义 delete
    void operator delete(void* ptr) {
        std::cout << "释放内存\n";
        ::operator delete(ptr);
    }
};
```

  

## 一些高级用法

1. 布局 new (Placement new)
    

在预分配的内存上构造对象：

```C++
#include <new>

char buffer[sizeof(MyClass)]; // 预分配内存
MyClass* p = new (buffer) MyClass(); // 在 buffer 上构造对象

// 显式调用析构函数
p->~MyClass();
```

2. 自定义分配器
    

```C++
struct Arena {
    static constexpr size_t size = 1024*1024;
    char data[size];
    char* current = data;
    
    void* allocate(size_t bytes) {
        if (current + bytes > data + size) 
            throw std::bad_alloc();
        void* ptr = current;
        current += bytes;
        return ptr;
    }
};

Arena arena;
MyClass* p = new (arena.allocate(sizeof(MyClass))) MyClass();
```

3. 对齐控制 (C++17)
    

```C++
// 分配对齐到 64 字节边界的内存
alignas(64) int* p = new int;
```

## 最佳实践

1. **优先使用智能指针**：使用 `std::unique_ptr` 和 `std::shared_ptr` 自动管理内存
    
    ```C++
    #include <memory>
    auto p = std::make_unique<int>(42); // 自动管理内存
    ```
    
2. **RAII 原则**：资源获取即初始化
    
    ```C++
    class FileHandler {
        FILE* file;
    public:
        FileHandler(const char* name) : file(fopen(name, "r")) {}
        ~FileHandler() { if(file) fclose(file); }
    };
    ```
    
3. **避免裸 new/delete**：在业务代码中尽量使用容器和智能指针
    
    ```C++
    std::vector<int> vec(100); // 替代 new int[100]
    ```
    
4. **异常安全**：确保在异常发生时资源被正确释放
    
    ```C++
    void safeFunction() {
        auto p = std::make_unique<Resource>();
        // 即使抛出异常，资源也会被释放
    }
    ```
    
5. **内存泄漏检测**：使用工具如 Valgrind、AddressSanitizer
    

## 常见错误

```C++
// 1. 内存泄漏
int* p = new int[100];
// 忘记 delete[]

// 2. 重复释放
delete p;
delete p; // 未定义行为

// 3. 悬垂指针
int* p = new int(10);
delete p;
*p = 20; // 访问已释放内存

// 4. 不匹配的 new/delete
int* arr = new int[10];
delete arr; // 应该用 delete[] arr;

// 5. 构造函数抛出异常
class Problematic {
public:
    Problematic() { throw std::runtime_error("Oops"); }
};

try {
    Problematic* p = new Problematic;
} catch(...) {
    // 内存泄漏！operator new 已分配内存但构造函数失败
}
```

注意：

C++允许我们定义一个基类的指针指向子类的数据空间

但当子类中存在覆盖基类的方法时，我们想用指针调用子类的方法，就会出现问题，结果是调用了基类的方法

在编译的时候，编译器会为认为这里是一个BaseClass型的指针，因此也就认为指针中的test方法是基类的方法。

为了让编译器知道他应该根据这两个指针在运行时的类型而有选择的调用正确的方法，我们必须把子类覆盖基类的方法声明为虚方法
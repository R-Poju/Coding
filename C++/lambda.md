在 C++ 中，**Lambda 表达式**（匿名函数）是一种轻量级的“就地定义”函数对象，允许你在代码中直接嵌入简短的函数逻辑。它结合了函数指针的灵活性和函数对象的强大表达能力，是现代 C++ 中最常用的特性之一（尤其在 STL 算法、回调、异步编程中）。

### 一、核心概念：什么是 Lambda？

Lambda 表达式本质是**一个匿名的“闭包”（Closure）**——它封装了：

- 

  **函数体**（要执行的逻辑）；

- 

  **捕获列表**（从外部作用域“捕获”的变量，用于函数体内部访问）；

- 

  **参数列表**（调用时需要传入的参数，可选）；

- 

  **返回类型**（可由编译器自动推导，可选）。

闭包类型由编译器自动生成，每个 Lambda 都有唯一的类型（无法直接写出，通常用 `auto`存储）。

### 二、基本语法

```cpp
[capture-list](parameters) mutable(optional) noexcept(optional) -> return-type(optional) {
    function-body
}
```

#### 各组成部分详解：

| 部分                   | 说明                                                         |
| ---------------------- | ------------------------------------------------------------ |
| **`[capture-list]`**   | **捕获列表**（核心）：指定从外部作用域捕获哪些变量（值/引用/初始化捕获）。 |
| **`(parameters)`**     | **参数列表**：与普通函数参数类似，支持 `auto`（C++14 起）、模板参数（C++20 起）。 |
| **`mutable`**          | 可选关键字：允许修改**值捕获**的变量（默认情况下，值捕获的变量在 Lambda 内是 `const`的）。 |
| **`noexcept`**         | 可选：指定 Lambda 是否抛出异常（C++11 起）。                 |
| **`-> return-type`**   | 可选：显式指定返回类型（若省略，编译器根据函数体自动推导）。 |
| **`{function-body}``** | **函数体**：Lambda 的执行逻辑。                              |

### 三、捕获列表（Capture List）：核心机制

捕获列表决定了 Lambda 能否访问外部作用域的变量，以及如何访问（值/引用）。这是 Lambda 与普通函数的最大区别。

#### 1. 基础捕获方式

| 捕获方式           | 语法示例            | 说明                                                         |
| ------------------ | ------------------- | ------------------------------------------------------------ |
| **值捕获**         | `[x, y]`            | 复制外部变量 `x`和 `y`的值到 Lambda 内部（修改不影响外部）。 |
| **引用捕获**       | `[&x, &y]`          | 捕获外部变量 `x`和 `y`的引用（修改会影响外部）。             |
| **隐式值捕获**     | `[=]`               | 自动捕获所有外部变量（按值）。                               |
| **隐式引用捕获**   | `[&]`               | 自动捕获所有外部变量（按引用）。                             |
| **混合捕获**       | `[=, &x]`/ `[&, x]` | `[=, &x]`：除 `x`外按值捕获，其余按引用捕获；`[&, x]`：除 `x`外按引用捕获，其余按值捕获。 |
| **不捕获任何变量** | `[]`                | 空捕获列表，Lambda 不能访问外部变量（仅依赖参数）。          |

#### 2. 进阶捕获方式（C++11 及以后）

##### （1）初始化捕获（Init Capture，C++14 起）

允许用**任意表达式**初始化捕获的变量（解决“移动捕获”“重命名捕获”等问题）。

语法：`[var_name = expression]`

**示例：移动捕获 `unique_ptr`**

```cpp
#include <memory>
int main() {
    auto ptr = std::make_unique<int>(42);  // unique_ptr 不可复制，只能移动
    
    // 用初始化捕获移动 ptr 的所有权到 Lambda 内部
    auto lambda = [p = std::move(ptr)]() {  // p 是 Lambda 内部的变量（值为移动后的 ptr）
        // 使用 p...
    };
}
```

**示例：重命名捕获变量**

```cpp
int x = 10;
auto lambda = [y = x * 2]() {  // 捕获表达式 x*2，命名为 y
    return y;  // 返回 20
};
```

##### （2）捕获 `this`指针（类成员函数中）

在类的成员函数中，Lambda 可通过 `[this]`捕获当前对象的指针，从而访问成员变量/函数：

```cpp
class MyClass {
    int value = 42;
public:
    void func() {
        // 捕获 this，访问成员变量 value
        auto lambda = [this]() {
            return value;  // 等价于 this->value
        };
        lambda();  // 返回 42
    }
};
```

⚠️ **注意**：若对象提前销毁（如异步 Lambda 中），捕获 `this`可能导致悬垂指针！C++17 起可用 `[*this]`捕获对象的副本（值捕获整个对象）。

### 四、参数列表与返回类型

#### 1. 参数列表

与普通函数参数类似，支持任意类型（包括 `auto`，C++14 起）：

```cpp
// C++11：显式指定参数类型
auto add = [](int a, int b) { return a + b; };

// C++14：用 auto 实现泛型 Lambda（等价于模板函数）
auto add_auto = [](auto a, auto b) { return a + b; };  // 支持 int/double/string 等
add_auto(1, 2);      // 3（int）
add_auto(1.5, 2.5);  // 4.0（double）
```

#### 2. 返回类型推导

- 

  **单条 return 语句**：编译器自动推导返回类型（无需显式指定）。

- 

  **多条 return 语句/无 return**：需显式指定返回类型（用 `-> type`）。

```cpp
// 自动推导返回类型（int）
auto square = [](int x) { return x * x; };

// 显式指定返回类型（double）
auto divide = [](double a, double b) -> double {
    if (b == 0) return 0.0;  // 多条 return，需显式指定
    return a / b;
};
```

### 五、关键特性：`mutable`关键字

默认情况下，Lambda 的**调用运算符是 `const`的**，因此**值捕获的变量在 Lambda 内部不可修改**（视为 `const`）。若需修改值捕获的变量，需添加 `mutable`关键字：

```cpp
int x = 10;
auto lambda = [x](int delta) mutable {  // mutable 允许修改 x
    x += delta;  // 修改的是 Lambda 内部复制的 x（不影响外部）
    return x;
};

lambda(5);  // 返回 15（内部 x 变为 15）
lambda(5);  // 返回 20（内部 x 继续累加）
x;          // 仍为 10（外部 x 未被修改）
```

### 六、使用场景

Lambda 几乎可用于所有需要“临时函数”的场景，尤其适合**简洁逻辑**和**就地定义**。

#### 1. 作为 STL 算法的谓词（Predicate）

STL 算法（如 `sort`、`find_if`、`for_each`）常需传入自定义比较/过滤逻辑，Lambda 是最便捷的选择：

```cpp
#include <vector>
#include <algorithm>
#include <iostream>

int main() {
    std::vector<int> nums = {3, 1, 4, 1, 5, 9};
    
    // 1. 排序（降序）：用 Lambda 自定义比较
    std::sort(nums.begin(), nums.end(), [](int a, int b) {
        return a > b;  // 降序排列
    });
    
    // 2. 过滤偶数（find_if + Lambda）
    auto it = std::find_if(nums.begin(), nums.end(), [](int x) {
        return x % 2 == 0;  // 找第一个偶数
    });
    
    // 3. 遍历打印（for_each + Lambda）
    std::for_each(nums.begin(), nums.end(), [](int x) {
        std::cout << x << " ";  // 输出：9 5 4 3 1 1
    });
}
```

#### 2. 作为回调函数（Callback）

在异步编程（如线程、事件驱动）中，Lambda 可作为简洁的回调逻辑：

```cpp
#include <thread>
#include <iostream>

int main() {
    int result = 0;
    
    // 启动线程，用 Lambda 作为线程函数（捕获 result 的引用）
    std::thread t([&result]() {
        result = 42;  // 修改外部变量
        std::cout << "Thread: result = " << result << "\n";
    });
    
    t.join();
    std::cout << "Main: result = " << result << "\n";  // 输出 42
}
```

#### 3. 封装复杂逻辑（替代函数对象）

对于简短逻辑，Lambda 比手写函数对象（Functor）更简洁：

```cpp
// 传统函数对象（繁琐）
struct Adder {
    int operator()(int a, int b) const { return a + b; }
};
Adder add_obj;
add_obj(1, 2);  // 3

// Lambda（简洁）
auto add_lambda = [](int a, int b) { return a + b; };
add_lambda(1, 2);  // 3
```

### 七、与 `std::bind`的对比（回顾）

Lambda 是现代 C++ 中替代 `std::bind`的首选方案，优势如下：

| 特性         | `std::bind`                            | Lambda 表达式                        |
| ------------ | -------------------------------------- | ------------------------------------ |
| **可读性**   | 复杂绑定时代码晦涩（如占位符 `_1/_2`） | 逻辑直观，接近自然语言               |
| **灵活性**   | 仅支持参数绑定/重排                    | 支持任意逻辑（循环、分支、局部变量） |
| **性能**     | 间接调用开销（函数指针跳转）           | 编译器可直接内联优化（效率更高）     |
| **类型安全** | 依赖隐式转换                           | 强类型检查                           |
| **C++ 版本** | C++98 起支持                           | C++11 起支持（现代项目首选）         |

**示例：等效功能对比**

```cpp
// 目标：绑定 add(10, ?)，实现 10+x
int add(int a, int b) { return a + b; }

// 用 std::bind
#include <functional>
auto add10_bind = std::bind(add, 10, std::placeholders::_1);

// 用 Lambda（更直观）
auto add10_lambda = [](int x) { return add(10, x); };
```

### 八、注意事项（避坑指南）

1. 

   **避免悬垂引用**

   引用捕获（`[&]`）的变量需确保其生命周期长于 Lambda。例如，异步 Lambda 中捕获局部变量的引用会导致未定义行为：

   ```cpp
   auto create_lambda() {
       int x = 10;
       return [&x]() { return x; };  // 危险！x 在函数返回后销毁
   }
   ```

2. 

   **谨慎使用隐式捕获（`[=]`/`[&]`）**

   过度捕获会增加闭包大小，甚至意外捕获无关变量。建议显式列出需要捕获的变量（如 `[x, &y]`）。

3. 

   **`mutable`的正确使用**

   仅当需要修改值捕获的变量时才用 `mutable`，避免滥用导致逻辑混乱。

4. 

   **性能考量**

   Lambda 通常可内联优化，但若用 `std::function`包装（跨接口传递），会有额外开销（类型擦除）。优先用 `auto`存储 Lambda。

### 九、C++14/17/20 新特性扩展

- 

  **C++14**：泛型 Lambda（`auto`参数）、初始化捕获、返回类型推导增强。

- 

  **C++17**：`constexpr`Lambda（可在编译期执行）、捕获 `*this`（值捕获整个对象）。

- 

  **C++20**：模板 Lambda（`template<typename T> [](T x) { ... }`）、`noexcept`作为尾置返回类型的一部分、结构化绑定捕获。

### 十、总结

Lambda 表达式是 C++ 中**“就地定义函数逻辑”**的神器，核心优势是：

- 

  **简洁性**：无需额外定义函数/函数对象；

- 

  **灵活性**：通过捕获列表自由访问外部变量；

- 

  **可读性**：逻辑与上下文紧密结合，一目了然。

**现代 C++ 开发准则**：能用 Lambda 就不用 `std::bind`或手写函数对象，让代码更紧凑、更易维护。

**一句话概括**：Lambda 是“写在代码里的临时函数”，让 C++ 的逻辑表达更接近自然语言。
在 C++ 中，`try-catch`是处理异常的核心机制，它允许程序在运行时检测和处理错误条件。结合之前讨论的 **RAII**，可以构建健壮的异常处理系统。以下是详细解析：

------

### 一、基本语法

```cpp
try {
    // 可能抛出异常的代码
    throw some_error;  // 主动抛出异常
}
catch (ExceptionType1& e) {  // 捕获特定类型异常
    // 处理 ExceptionType1
}
catch (ExceptionType2& e) {  // 捕获另一种异常
    // 处理 ExceptionType2
}
catch (...) {  // 捕获所有未处理的异常
    // 兜底处理
}
```

------

### 二、关键概念

#### 1. **异常抛出 (`throw`)**

- 

  使用 `throw`关键字抛出异常对象

- 

  可以是基本类型、字符串或自定义类型

```cpp
throw std::runtime_error("File not found");
```

#### 2. **异常捕获 (`catch`)**

- 

  按声明顺序匹配异常类型（最具体的最先匹配）

- 

  使用引用捕获避免切片问题：`catch (const std::exception& e)`

- 

  `catch(...)`捕获所有异常（用于日志记录/清理）

#### 3. **栈展开 (Stack Unwinding)**

- 

  当异常抛出时，C++ 会逐层退出函数调用栈

- 

  **RAII 在此阶段发挥作用**：每个局部对象的析构函数会被自动调用

```cpp
void risky() {
    Resource res;  // RAII 对象
    throw std::logic_error("Oops");  // 触发栈展开
    // res 的析构函数会自动调用！
}
```

------

### 三、标准异常体系 (`<stdexcept>`)

| 异常类                | 描述               | 头文件        |
| --------------------- | ------------------ | ------------- |
| `std::exception`      | 所有标准异常的基类 | `<exception>` |
| `std::logic_error`    | 程序逻辑错误       | `<stdexcept>` |
| ├─ `invalid_argument` | 无效参数           |               |
| ├─ `out_of_range`     | 越界访问           |               |
| `std::runtime_error`  | 运行时错误         | `<stdexcept>` |
| ├─ `overflow_error`   | 算术上溢           |               |
| └─ `underflow_error`  | 算术下溢           |               |

**使用示例：**

```cpp
#include <stdexcept>
try {
    if (value < 0) 
        throw std::invalid_argument("Negative value");
}
catch (const std::invalid_argument& e) {
    std::cerr << "Error: " << e.what();  // 输出错误信息
}
```


### 四、实用与陷阱

#### ✅ 正确做法

1. 

   **按引用捕获异常**

   `catch (const std::exception& e)`避免对象切片

2. 

   **从最具体到最通用排序**

   ```cpp
   catch (const FileError& e) {...}
   catch (const IOError& e) {...}  // 基类在后
   catch (...) {...}
   ```

3. 

   **结合 RAII 管理资源**

   确保异常发生时资源自动释放

4. 

   **提供有意义的错误信息**

   重写 `what()`返回具体错误原因

#### ⚠️ 常见陷阱

1. 

   **吞没异常**

   ```cpp
   catch (...) { /* 忽略所有错误 */ }  // 危险！
   ```

2. 

   **抛出局部对象的指针**

   ```cpp
   throw new MyError;  // 内存泄漏！应用值传递
   ```

3. 

   **过度使用异常**

   简单错误处理（如输入校验）优先用返回值

4. 

   **跨语言边界抛异常**

   在 C/C#/Java 交互时慎用


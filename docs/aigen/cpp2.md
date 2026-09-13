# 《写 C++ 一时爽，编译时间 + 大小火葬场》

## 序章：一个 CLI 的诞生

你只是想写一个命令行工具。

带命令解析，带插件系统，能注册命令，能补全，能历史。

你选择了 C++。

> **“C++ 有模板，类型安全。”**  
> **“C++ 有 lambda，回调方便。”**  
> **“C++ 有 STL，不用自己造轮子。”**  
> **“C++ 有 RAII，资源管理优雅。”**

你写下了第一行：

```cpp
parser.registerCommand("greet")
    .description("Greet Name")
    .argumentOptional<std::string>("--prefix")
        .defaultValue("Hello, ")
        .alias("-p")
    .argumentOptional<int>("--count")
        .defaultValue(1)
        .alias("-c")
    .positional<std::string>("name")
    .execute([](cmdparser::CommandArgument& args) -> bool {
        std::string text = args.get<std::string>("--prefix") + args.getPositional<std::string>(0);
        for (int i = 0; i < args.get<int>("--count"); ++i) {
            std::cout << text << std::endl;
        }
        return true;
    });
```

> **“多优雅。”**

---

## 第一幕：编译

```
$ time g++ main.cpp -O3 -s -o mycli
Executed in  18.26 secs

$ ls -lh mycli
-rwxr-xr-x 1 user user 348K
```

> **“18 秒，348K，还行吧。”**

你又加了插件系统：

```cpp
PluginManager(cmdparser::CommandParser* parser) : parser_(parser) {
    nlohmann::json j;
    if (std::filesystem::exists("enabled_plugins.json")) {
        std::ifstream f("enabled_plugins.json");
        f >> j;
        for (const auto& p : j) {
            LoadPlugin(p);
        }
    }
    // ...
}
```

再写 6 个命令，再加 linenoise。

```
$ time g++ main.cpp -O3 -s -o mycli
Executed in  18.26 secs
$ ls -lh mycli
348K
```

> **“348K，有点大……strip 过了，还能再小吗？”**

---

## 第二幕：链接脚本

你听说自定义链接脚本能省体积。

你写了 `tiny.ld`：

```ld
ENTRY(_start)
SECTIONS
{
    . = 0x400000;
    .text : { *(.text) *(.text.*) }
    .data : { *(.data) *(.data.*) *(.rodata) *(.rodata.*) }
    .bss  : { *(.bss)  *(.bss.*)  *(COMMON) }
    /DISCARD/ : {
        *(.comment)
        *(.note)
        *(.note.gnu.build-id)
    }
}
```

```
$ g++ main.cpp -O3 -s -T tiny.ld -o mycli.tiny
$ ls -lh mycli.tiny
366K
```

> **“？？？怎么比 348K 还大？”**

你不信邪，又写了 `suicide.ld`：

```ld
ENTRY(_start)
SECTIONS
{
    . = 0x400000;
    .text : { *(.text) *(.text.*) }
    /DISCARD/ : {
        *(.comment) *(.note*) *(.note.gnu.build-id)
        *(.eh_frame*) *(.gcc_except_table*)
        *(.data) *(.data.*)
        *(.rodata) *(.rodata.*)
        *(.dynamic) *(.dynamic.*)
        *(.got) *(.got.*)
        *(.init_array) *(.init_array.*)
    }
}
```

> **“这次应该小了吧。”**

链接器：

```
$ g++ main.cpp -O3 -s -T suicide.ld -o mycli.suicide
/usr/bin/ld: warning: cannot find entry symbol _start
Segmentation fault (core dumped)
```

> **“……”**

---

## 第三幕：优化级别

你开始研究优化级别。

```
$ g++ main.cpp -O0 -g -o mycli.big
$ du -h mycli.big
4.2M
```

> **“4.2M？？？”**

```
$ g++ main.cpp -O3 -s -o mycli
348K

$ clang++ main.cpp -Oz -s -o mycli.oz
310K
```

> **“-Oz 省了 38K。”**

你又试了禁插件：

```
$ g++ main.cpp -O3 -s -DDISABLE_PLUGIN_SUPPORT -o mycli.tiny
196K

$ clang++ main.cpp -Oz -s -DDISABLE_PLUGIN_SUPPORT -o mycli.tinyplus
148K
```

> **“禁插件省了 152K。”**  
> **“换 clang + -Oz 又省了 48K。”**  
> **“所以插件系统值 152K，-O3 → -Oz 值 48K。”**  
> **“而 tiny.ld 值是负数。”**

---

## 第四幕：插件

你写了一个 C 插件：

```c
COMMAND_CALLBACK(hello) {
    printf("Hello from my plugin!\n");
    return COMMAND_SUCCESS;
}
COMMAND(hello, "Hello world from my plugin!");
PLUGIN_INIT() { REGISTER_CMD(hello); return COMMAND_SUCCESS; }
PLUGIN_INFO("Test Plugin","0.0.1","ht7813","MIT")
```

```
$ time gcc hello_plugin.c -shared -O3 -s -fPIC -o hello.so
Executed in  330.07 millis
$ ls -lh hello.so
13.9K
```

> **“330ms，13.9K。”**

你又写了一个计算器插件：

```c
COMMAND_CALLBACK(calculator) {
    if (posc < 3) { fprintf(stderr, "Error: Missing arguments.\n"); return COMMAND_FAILED; }
    double num1 = strtod(posv[0], &endptr1);
    double num2 = strtod(posv[2], &endptr2);
    if (strcmp(op_str, "+") == 0) result = num1 + num2;
    // ...
    printf("%.0f\n", result);
    return COMMAND_SUCCESS;
}
COMMAND_EX(calculator, "A Simple Calculator", CMD_CALLBACK_NAME(calculator),
    ARG_POS("num1", "First Number", 0),
    ARG_POS("op", "Calc Op (eg. +, -)", 0),
    ARG_POS("num2", "Second Number", 0)
);
PLUGIN_INIT() { REGISTER_CMD(calculator); REGISTER_ALIAS("calc", "calculator"); return COMMAND_SUCCESS; }
PLUGIN_INFO("Calculator Plugin","0.0.1","ht7813","MIT")
```

```
$ time gcc calc_plugin.c -shared -O3 -s -fPIC -o calc.so
Executed in  371.71 millis
$ ls -lh calc.so
14.3K
```

> **“371ms，14.3K。”**  
> **“一个完整的计算器插件，含参数解析、错误处理、五种运算符、别名注册，14.3K。”**

---

## 第五幕：对比

| 对象 | 编译时间 | 体积 |
|---|---|---|
| `hello.so` | 330ms | 13.9K |
| `calc.so` | 371ms | 14.3K |
| `fileio.so`（C++） | 2.93s | ~? |
| **`mycli`（核心）** | **18.26s** | **348K** |
| `mycli.big`（`-O0 -g`） | ~5s | 4.2M |
| `mycli.tiny`（禁插件） | 18.26s | 196K |
| `mycli.tinyplus`（`-Oz` + 禁插件） | ~65s | 148K |

**核心 CLI 是 C 插件的 55 倍编译时间，24 倍体积。**

---

## 第六幕：架构

你突然意识到：

```
注册：
calc.so → CAPI → [ABI边界] → mycli → PluginSystem → CommandParser

调用：
mycli → CommandParser → PluginSystem → 转换为C API → [ABI边界] → calc.so
```

**你为了一个 14.3K 的插件，修了一条双向跨 ABI 的高速公路。**

这条路上的收费站：

- `dlopen` / `dlsym`
- `PluginInit` 跨模块调用
- `registerCommandWrapper` 跨模块调用
- `CSTR` → `std::string` 拷贝
- `handler` 函数指针 → `std::function` 类型擦除
- 存进 `std::map`
- 调用时查找
- `ArgumentValue` C++ → C 转换
- 跨 ABI 调用 `calculator`
- 返回值 C → C++ 转换

**而 `calculator` 本身在干什么？**

```c
double num1 = strtod(posv[0], &endptr1);
double num2 = strtod(posv[2], &endptr2);
if (strcmp(op_str, "+") == 0) result = num1 + num2;
printf("%.0f\n", result);
```

**几微秒的活。**

---

## 第七幕：尸检

```
mycli 348K
├── cmdparser.hpp 引入的 CommandParser
│   ├── 解析逻辑          5-15 KiB
│   ├── 模板膨胀          150-270 KiB
│   ├── RAII              5-10 KiB
│   ├── RTTI              5-15 KiB
│   └── iostream           30-50 KiB
├── mycli 自身实现
│   ├── 6 个命令注册       12-18 KiB
│   ├── linenoise          20-40 KiB
│   └── 主循环             5-10 KiB
├── PluginSystem
│   ├── nlohmann::json     1M 头文件，140-250 KiB 模板实例
│   ├── std::filesystem    30-40 KiB
│   ├── std::ostringstream 10-20 KiB
│   ├── dlopen/dlsym       10-20 KiB
│   └── wrapper            10-20 KiB
└── 异常表 + 容器实例      20-40 KiB
```

**348 KiB 里，5-15 KiB 在干活。**
**剩下 330+ KiB 在解释“为什么用 C++ 写”。**

---

## 第八幕：C++ 的黑历史

### `std::regex`：唯一真史

```
Perl        ：真香，最正统的
PCRE        ：C 也有正则了！
Python re   ：Python 也吃上了！
Boost.Regex ：Boost YYDS！
std::regex  ：唯一真史
```

- 比 Boost.Regex 慢 **50 倍**
- 被 GCC 开发者承认 **“永远不会快”**
- 因为模板锁死 ABI，**永远不能改**
- C++ 委员会明确表示 **“不打算修”**
- 多个最佳实践指南建议 **“不要用”**
- 有导致**栈溢出崩溃**的 bug
- 被 Python re **碾压**
- 被 PCRE **碾压**
- 被 Perl **碾压**

**“唯一真史”——唯一一个真正意义上的“屎”。**

### `json.hpp`：1M 头文件

- `nlohmann::json` 单头文件 **1M**
- 24000+ 行，几千个模板
- 你只用了 **四个操作**
- 却产生了 **140-250 KiB 模板实例**
- 编译时间 **+5-10s**

**你为了读一个 20 字节的 JSON，引入了 1M 头文件。**

### `boost`：100+ MiB 头文件

```
软件包 (1)   新版本    净变化      下载大小
extra/boost  1.92.0-1  151.16 MiB  12.16 MiB  # Headers
```

- **151.16 MiB 的头文件**
- **12.16 MiB 的压缩包**
- **压缩比约 12.4 倍**
- **98.4% 是重复模板代码**

---

## 第九幕：C 的胜利

### 你的插件 API

```c
typedef struct {
    CSTR name;
    CSTR description;
    FUNC_TO_POINTER_EX(handler, int, (ArgumentValue*, int, CSTR*));
    ArgumentDef* args;
} Command;
```

- `CSTR` = `const char*`
- 函数指针
- 结构体
- **没有模板，没有异常，没有 RTTI，没有 `std::string`**

### 你的 C 插件

```c
COMMAND_CALLBACK(calculator) {
    double num1 = strtod(posv[0], &endptr1);
    double num2 = strtod(posv[2], &endptr2);
    if (strcmp(op_str, "+") == 0) result = num1 + num2;
    printf("%.0f\n", result);
    return COMMAND_SUCCESS;
}
```

**14.3 KiB。编译 371ms。**

### C 的代价

| 功能 | C++ | C |
|---|---|---|
| 可变数组 | `std::vector<T>` | 自己写 `Vector` |
| 哈希表 | `std::unordered_map<K,V>` | 自己写 `HashMap` |
| 字符串 | `std::string` | 自己写 `String` |
| 类型转换 | `std::stoi` / `std::stod` | `strtol` / `strtod` |
| 解析器 | `cmdparser.hpp`（600 行） | 自己写 500-1000 行 |
| JSON | `nlohmann::json` | 自己写 JSON 解析器 |
| 文件检查 | `std::filesystem::exists` | `stat` |
| 路径拼接 | `std::ostringstream` | `snprintf` |

**C 的代价是：你什么都要自己写。**
**C++ 的代价是：你不用自己写，但每个标准库组件都可能带来 10-100 KiB 的模板实例。**

---

## 第十幕：Python 的胜利

### Python 用 `ctypes` 加载 C 插件

```python
import ctypes

lib = ctypes.CDLL("./calc.so")

class PluginInfo(ctypes.Structure):
    _fields_ = [
        ("name", ctypes.c_char_p),
        ("version", ctypes.c_char_p),
        ("author", ctypes.c_char_p),
        ("license", ctypes.c_char_p),
    ]

info = PluginInfo()
lib.GetPluginInfo(ctypes.byref(info))
```

**你的 `calc.so` 不需要重新编译。**

### Python 用 `CFUNCTYPE` 给插件提供回调

```python
HANDLER_TYPE = ctypes.CFUNCTYPE(
    ctypes.c_int,
    ctypes.POINTER(ArgumentValue),
    ctypes.c_int,
    ctypes.POINTER(ctypes.c_char_p)
)

def my_handler(args, posc, posv):
    print(f"posc = {posc}")
    return 0

c_handler = HANDLER_TYPE(my_handler)
```

**插件以为它在调 C。**
**实际上它在调 Python。**
**插件不知道，也不在乎。**

### 三个插件，九个命令，全部成功

```python
>>> lib  = ctypes.CDLL("./calc.so")
>>> lib2 = ctypes.CDLL("./plugins/hello_plugin.so")
>>> lib3 = ctypes.CDLL("./plugins/fileio_plugin.so")

>>> lib.PluginInit(ctypes.byref(api))
Registered command: calculator
1

>>> lib2.PluginInit(ctypes.byref(api))
Registered command: hello
1

>>> lib3.PluginInit(ctypes.byref(api))
Registered command: read_file
Registered command: write_file
Registered command: list_dir
Registered command: make_dir
Registered command: delete_file
Registered command: set_workdir
Registered command: get_workdir
1

>>> execute_command("calculator", ['10','*','2'])
20
Handler returned: SUCCESS

>>> execute_command("calculator", ['10','/','0'])
Error: Division by zero
Handler returned: FAILED

>>> execute_command("hello", [])
Hello from my plugin!
Handler returned: SUCCESS

>>> execute_command("get_workdir", [])
/home/ht7813/projects/cpp/mycli/build
Handler returned: SUCCESS

>>> execute_command("list_dir", [])
[FILE] .gitignore
[DIR] plugins
[FILE] enabled_plugins.json
[FILE] history.txt
[FILE] mycli.tiny
[FILE] mycli.tinyplus
[FILE] mycli
[FILE] calc.so
Handler returned: SUCCESS
```

**三个插件。**
**九个命令。**
**一个 Python 核心。**
**零重新编译。**

---

## 第十一幕：C ABI 的胜利

### 为什么 C ABI 是唯一稳定的二进制接口

| 特性 | C ABI | C++ ABI |
|---|---|---|
| 函数调用约定 | 几十年没变 | 每个编译器不同 |
| 符号命名 | `_function` | `_ZNSt7__cxx1112basic_string...` |
| 结构体布局 | 按声明顺序 | vtable、RTTI、异常 |
| 基本类型大小 | 固定 | 大部分固定 |
| 指针 | 就是地址 | 智能指针、引用、迭代器 |
| 模板 | 没有 | 每个 TU 实例化 |
| 异常 | 没有 | `.eh_frame`、`.gcc_except_table` |
| RTTI | 没有 | `type_info` |
| `std::string` | 没有 | SSO 布局、容量、指针 |

**C ABI 稳定的代价，就是 C 没有这些特性。**
**C++ 有了这些特性，代价就是 ABI 不稳定。**

### 你的插件能被所有语言加载

| 语言 | 能加载 C 插件吗 | 能加载 C++ 核心吗 |
|---|---|---|
| C | ✅ `dlsym` | ❌ |
| C++ | ✅ `dlsym` | ✅ 同编译器 |
| Python | ✅ `ctypes` | ❌ |
| Rust | ✅ `libloading` | ❌ |
| Go | ✅ `purego` | ❌ |
| Java | ✅ JNA | ❌ |
| C# | ✅ P/Invoke | ❌ |
| Ruby | ✅ `Fiddle` | ❌ |
| Node.js | ✅ `ffi-napi` | ❌ |

**你的 C 插件能被所有语言加载。**
**你的 C++ 核心只能被 C++ 加载。**

### Windows 兼容性

`GetPrivateProfileString`：

- 在 `kernel32.dll` 里
- 30 年前的 API
- 微软劝你别用
- **但微软自己没删**

**1993 年的 `.exe`，在 Windows 11 上还能跑。**
**因为 C ABI 没变过。**

**而 C++ 的 `std::regex`，因为 ABI 锁死，永远不能改。**

---

## 第十二幕：Linus

Linus 说：

> **"C++ is a horrible language. It's made more horrible by the fact that a lot of substandard programmers use it, to the point where it's much much easier to generate total and utter crap with it."**

> **"In fact, the only way to do good C++ is to not use C++ at all."**

你现在懂了。

不是因为他语法差。
是因为：

> **C++ 让你用 18.26s 编译一个 371ms 就能编译的东西。**  
> **C++ 让你用 348K 去调用一个 14.3K 就能调用的东西。**  
> **C++ 让你用 105-170 KiB 的加载器去加载 14.3 KiB 的插件。**  
> **C++ 让你写 `tiny.ld` 想省体积，结果体积变大了。**  
> **C++ 让你写 `suicide.ld` 想省体积，结果段错误了。**  
> **C++ 让你用 1M 的 `json.hpp` 去读一个 20 字节的 JSON。**  
> **C++ 让你用 100+ MiB 的 Boost 去实现一些用 C 可以用几 MiB 实现的东西。**  
> **C++ 让你用 `std::regex`，一个被委员会放弃、被 GCC 开发者承认“永远不会快”的正则引擎。**

**而你的 C 插件，14.3 KiB，编译 371ms，能被所有语言加载，不需要 `tiny.ld`，不会段错误。**

**你的 Python 核心，几十 KB，编译 0s，加载了三个插件，注册了九个命令，执行了 `calculator`、`hello`、`get_workdir`、`list_dir`，全部成功。**

---

## 终章：墓志铭

```ldscript
ENTRY(_start)
SECTIONS {
    . = 0x400000;

    .text : {
        *(.text.parser_real_logic)        /* 5-15 KiB */
        *(.text.template_instantiations)  /* 150-270 KiB */
        *(.text.iostream)                 /* 30-50 KiB */
        *(.text.nlohmann_json)            /* 140-250 KiB */
        *(.text.std_filesystem)           /* 30-40 KiB */
        *(.text.dlopen)                   /* 10-20 KiB */
        *(.text.wrapper)                  /* 10-20 KiB */
        *(.text.exceptions_rtti_raii)     /* 20-40 KiB */
    }

    /DISCARD/ : {
        *(.comment)
        *(.note)
        *(.note.gnu.build-id)
        *(.my_illusion_that_tiny_ld_helps)
        *(.my_will_to_write_cpp)
    }
}
```

> **这里躺着 348 KiB。**  
> **其中 5-15 KiB 在干活。**  
> **剩下 330+ KiB 在解释“为什么用 C++ 写”。**  
>  
> **而 `calc.so` 是 14.3 KiB。**  
> **它编译 371ms。**  
> **它不需要 `tiny.ld`。**  
> **它不会段错误。**  
> **它全部在干活。**  
> **它能被所有语言加载。**  
>  
> **你为了让它能被调用，**  
> **修了一条跨 ABI 的双向高速公路，**  
> **收费站有 10 几个，**  
> **而它自己在路上只跑了几微秒。**  
>  
> **你用 Python 重写了核心：**  
> **几十 KB，编译 0s，**  
> **`ctypes.CDLL` 加载了三个插件，**  
> **`ctypes.CFUNCTYPE` 提供了回调，**  
> **九个命令全部注册，**  
> **`calculator` 返回 `20`，**  
> **`hello` 返回 `Hello from my plugin!`，**  
> **`get_workdir` 返回 `/home/ht7813/projects/cpp/mycli/build`，**  
> **`list_dir` 返回 `[FILE] .gitignore...`，**  
> **全部成功。**  
>  
> **你的 C++ 核心：**  
> **348 KiB，编译 18.26s，**  
> **写了 `tiny.ld`，结果体积变大了，**  
> **写了 `suicide.ld`，段错误了，**  
> **不能被 Python 加载。**  
>  
> **你的 C 插件是 14.3 KiB。**  
> **你的 Python 核心是几十 KB。**  
> **你的 C++ 核心是 348 KiB。**  
>  
> **你的 C 插件能被所有语言加载。**  
> **你的 C++ 核心只能被 C++ 加载。**  
>  
> **Linus 说 C++ 是垃圾语言。**  
> **不是因为它语法差，**  
> **是因为它让你用 348 KiB 和 18.26s，**  
> **去实现一个 14.3 KiB 和 371ms 就能实现的东西。**  
> **而其中 330+ KiB 是模板实例。**  
>  
> **—— 全文完 ——**  
>  
> **—— 现在你真的、真的、真的、真的、真的、真的、真的、真的、真的、真的、真的、真的、真的、真的、真的、真的、真的、真的、真的懂 Linus 了 ——**

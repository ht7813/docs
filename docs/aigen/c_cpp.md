# **同一个库的 C++ 版和 C 版**

## 第一幕：C++ 版用户视角

```cpp
CommandParser parser("mycli");
parser.registerCommand("commit")
        .description("Commit changes")
        .argument<std::string>("-m")
        .argumentFlag<bool>("-a")
        .execute([](CommandArgument &args) -> bool
        {
            std::cout << "Commit: \nMessage: " << args.get<std::string>("-m") << "\nAll: " << (args.has("-a") ? "yes" : "no") << "\n";
            return true;  // 一个字：爽！
        });
```

---

## 第二幕：C 版用户视角

```c
static int commit_command(CommandArgument* args) {
    printf("Commit\nMessage: %s\nAll: %s\n", 
           cmdparser_get_arg_string(args, "-m"), 
           (cmdparser_has_arg(args, "-a") ? "yes" : "no"));
    return 1;
}

CommandParser* parser = cmdparser_parser_new("mycli");        // 开局怎么是指针？？？
Command* cmd = cmdparser_command_new("commit");               // 梅开二度
cmdparser_command_set_description(cmd, "Commit changes");     // 设个描述还要写又臭又长的函数名
cmdparser_command_add_argument(cmd, CMDPARSER_ARGUMENT_STRING("-m"));  // 类型就预设的那几个
cmdparser_command_add_argument(cmd, CMDPARSER_ARGUMENT_FLAG("-a"));    // 没有C++模板的日子可真难熬
cmdparser_command_set_handler(cmd, CMDPARSER_COMMAND_HANDLER(commit_command)); // 不想评价
cmdparser_parser_add_command(parser, cmd);                   // 配置完Command还要手动添加
```

---

## 第三幕：开发人员互评（新增）

**该库的 C++ 版开发人员（翘着二郎腿）：**

> "我自己也用这个库，写起来是真的爽。需求随便提，**单头（header-only）都行！** 用户只要`#include "cmdparser.hpp"`，没有`.dll`，没有`.so`，没有`.dylib`，**零依赖地狱**，解压即用，爽就完事了。"

**该库的 C 版开发人员（抱着头蹲在角落）：**

> "我也不想搞这么复杂啊！**C 的局限就摆在那**——没有模板，没有lambda，没有运算符重载，没有RAII，我拿什么给它做流式接口？！你嫌函数名又臭又长，我比你还嫌！"  
>  
> "你知道吗，我自己写项目用这个库的时候，**也经常忘记最后那步`cmdparser_parser_add_command`**！然后编译通过，一运行发现命令没注册，查了半天才发现漏了这行。😅"  
>  
> "我也想要单头啊！但C的单头是什么？是一整个`.h`里塞满宏和`static inline`函数，然后每个编译单元展开一遍，编译时间直接翻三倍。你管那叫'单头'？我管那叫'编译慢到想砸电脑'。"

---

## 终极补刀（围观群众）

> **C++ 用户：** "那你为什么不直接用 C++ 封装一层给 C 调用？"  
>  
> **C 开发人员：** "那样的话……我不就变成写 C++ 的了？"  
>  
> **C++ 开发人员：** "所以你现在理解为什么我们写 C++ 了吧？"  
>  
> **C 开发人员：** "……给我一个单头，我也写 C++。"  
>  
> **C++ 开发人员（递过 `cmdparser.hpp`）：** "欢迎入教。"

---

## 结尾金句

> **C 写起来像是给机器看的，C++ 写起来像是给人看的。**  
> **但C++编译出来的报错，是给神看的。**

（全场沉默，C++开发人员默默收回了递出的`.hpp`）
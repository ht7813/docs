# 《内核级“辅助”》

---

## 第一幕：诱饵

**视频标题**：《7.1.6内核亲测！MC最强公益辅助，反杀黑心服主！》

画面里，UP主用“Minecraft服务器被服主不公平对待”的剧本（进服被针对→放狠话→加载神秘插件→获得创造模式→反击服主），配上激昂的BGM，**特别强调**：

> **“适配最新Linux内核7.1.6！全发行版通用！完全免费，加群获取！”**

观众小A（化名）被视频里的“复仇”情节打动，加了群。

---

## 第二幕：群聊

群公告：

> **“公益名额已满，找管理员XXX发三连截图获取优惠卡密，限量100份！”**

小A私聊XXX，XXX回复：

> **“辅助本体35元，卡密找群主激活。先下载安装包，装好找我拿卡密。”**

XXX发来三个文件：`mcfuzhu.deb`、`mcfuzhu.rpm`、`mcfuzhu.pkg.tar.zst`。

小A用Arch Linux，下载了`.pkg.tar.zst`，安装后打开，弹窗：

> **“请输入卡密激活：”**

小A找XXX要卡密，XXX说：

> **“激活码50元，支付后找群主获取设备码授权。”**

---

## 第三幕：技术门槛

小A付了50元，XXX发来教程：

> **“1. 安装linux-headers：`sudo pacman -S linux-headers`**  
> **2. 下载后端模块：`wget [链接] -O mcfuzhu_backend.ko`**  
> **3. 加载模块：`sudo insmod mcfuzhu_backend.ko`**  
> **4. 查看dmesg获取设备码，发给群主兑换卡密。”**

小A照做：

```bash
$ sudo insmod mcfuzhu_backend.ko
insmod: ERROR: could not insert module mcfuzhu_backend.ko: Invalid module format
$ dmesg | tail -1
[  123.456789] mcfuzhu: version magic '7.1.5-arch1-2 SMP preempt mod_unload ' should be '7.1.6-custom-build SMP preempt mod_unload '
```

小A在群里问：

> **“我内核是7.1.6-custom-build，模块是7.1.5-arch1-2的，加载不了。”**

XXX迅速回复：

> **“Arch Linux官方还没推送7.1.6，我们现在只支持Arch官方仓库里的7.1.5内核。你用自己编译的custom-build内核，不在我们的支持范围内。请换回Arch官方内核再试。”**

小A查了一下，**Arch确实还没更新7.1.6**——Greg KH在8月3日就发布了，但Arch打包者还没推送。

小A有点犹豫，但觉得自己确实用了“非标准”内核，可能是自己的问题。

---

## 第四幕：旁观者

群里另一个用户B看到了对话。B是Linux内核爱好者，本地维护着完整的Linux源码树（`linux-mainline+stable.git`），自己也编译过`floppinux`。他看到XXX的回复后，在群里问了一句：

> **“Arch官方确实还没更新7.1.6，但你们的模块vermagic是`7.1.5-arch1-2`——说明它是在Arch 7.1.5上编译的。那如果用户换成Arch官方7.1.5内核，你们的模块就能正常工作了吗？”**

XXX：

> **“当然，我们只支持官方环境。”**

B没再说话。他打开虚拟机，安装了纯净的Arch Linux，**确保内核是官方7.1.5**，然后下载了`mcfuzhu.ko`，执行：

```bash
$ sudo insmod mcfuzhu_backend.ko
```

**结果**：

```text
Kernel panic - not syncing: Detected VM!
CPU: 0 UID: 0 PID: 123 Comm: insmod Tainted: P        O       7.1.5-arch1-2 #1
Tainted: [P]=PROPRIETARY_MODULE, [O]=OOT_MODULE
Hardware name: QEMU Standard PC (Q35 + ICH9, 2009), BIOS unknown 02/02/2022
Call Trace:
    dump_stack_lvl+0x4d/0x70
    vpanic+0x200/0x3f0
    panic+0x66/0x70
    HTtb1iC8HEs4+0x? [mcfuzhu]
    bj6AgwujIMy6+0x? [mcfuzhu]
    T5TpBWqINhh+0x? [mcfuzhu]
```

B截图，附带一段分析，发到群里：

> **“我用纯净的Arch官方7.1.5内核，完全符合你们说的‘官方环境’。结果你们的模块直接触发Kernel Panic，报`Detected VM!`——它在检测虚拟机，然后主动炸了系统。**  
>  
> **这不是‘版本不匹配’的问题，这是模块本身的行为问题。你们为什么要检测虚拟机？一个MC辅助，在虚拟机里加载有什么问题？”**

---

## 第五幕：技术拆解

群里的吃瓜群众开始议论。B趁热打铁，发了第二组截图：

**截图1**：`readelf -r mcfuzhu.ko` 输出：

```text
0000000000000028  0000000400000002 R_X86_64_PC32     0000000000000000 kallsyms_lookup_name - 4
0000000000000030  0000000500000002 R_X86_64_PC32     0000000000000000 sys_call_table - 4
0000000000000038  0000000600000002 R_X86_64_PC32     0000000000000000 call_usermodehelper - 4
```

**截图2**：`strings mcfuzhu.ko | grep systemd` 输出：

```text
systemd-detect-virt
```

**截图3**：加载后的污点值：

```text
$ cat /proc/sys/kernel/tainted
4097
```

B附了一段话：

> **“你们的模块依赖三个内核符号：`kallsyms_lookup_name`（查符号地址）、`sys_call_table`（系统调用表）、`call_usermodehelper`（执行用户态命令）。**  
>  
> **前面两个是用来**挂钩系统调用**的——这是内核级后门的标准操作。第三个是用来执行`systemd-detect-virt`检测虚拟机的。**  
>  
> **一个MC辅助，为什么要挂钩系统调用表？为什么要检测虚拟机？**  
>  
> **而且加载后内核污点值变成4097（P+O），比普通外部模块（只有O，4096）多了一个P——因为你们用了`Proprietary`许可证。NVIDIA都开源了，你们还有什么商业机密？”**

---

## 第六幕：群主下场

群主（之前一直不说话）突然出现：

> **“你这人怎么回事？我们模块是商业机密，不便公开源码。你发的那些截图都是伪造的，我们模块没有那些功能。而且你用虚拟机测试本身就不对，我们模块有防虚拟机保护，防止被破解。”**

B回复：

> **“防虚拟机保护？你们防的是破解，还是防的是**安全分析**？一个游戏辅助，有什么好‘防破解’的？你们卖的是卡密，不是代码。如果代码真的没问题，让用户拿去用就是了，为什么要阻止别人在虚拟机里跑？”**

群主：

> **“我们有权保护自己的知识产权。”**

B直接发了Linus Torvalds在2022年NVIDIA开源时的名言：

> **“NVIDIA has been the single worst company we've ever dealt with.”**  
>  
> **—— Linus Torvalds, 2012**  
>  
> **但NVIDIA在2022年开源了他们的Linux内核驱动，采用MIT+GPLv2双许可证。一个GPU厂商都敢开源，你们一个MC辅助却用`Proprietary`许可证——你们的技术比NVIDIA更机密？”**

---

## 第七幕：反噬

群里的风向开始变了：

- 用户C：“所以这个模块在虚拟机里会Panic？那我在物理机上用会不会有风险？”
- 用户D：“我加载的时候也报过`Tainted: P O`，当时没在意……”
- 用户E：“B说的那些符号是什么意思？有没有人能解释一下？”

B在群里发了一个公开笔记链接：

> **《mcfuzhu.ko 技术分析报告：一个自称“MC辅助”的内核后门》**

页面上详细列出了：

1. **时间差陷阱分析**：
   - 模块vermagic是`7.1.5-arch1-2`，在Arch 7.1.5上编译
   - 用户用自编译`7.1.6-custom-build`时报vermagic不匹配
   - 骗子利用“Arch还没更新7.1.6”这个事实，把问题归咎于“用户环境不标准”
   - 但换回Arch官方7.1.5后，模块依然Panic——**说明问题不是环境，是模块本身**

2. **高危符号依赖**：`kallsyms_lookup_name`、`sys_call_table`、`call_usermodehelper`

3. **虚拟机检测**：`systemd-detect-virt` 字符串 + 虚拟机内触发Panic

4. **污点值变化**：从4096到4097（P+O）

5. **NVIDIA开源对比**：MIT+GPLv2 vs Proprietary

---

## 第八幕：溃败

群主试图踢B出群，但已经有十几个用户截图保存了B的分析。群主解散了群聊，XXX的QQ号被举报封禁。

三天后，B在 `lore.kernel.org` 上看到一个帖子：

> **Subject: [BUG] Proprietary module mcfuzhu.ko triggers kernel panic with "Detected VM!"**  
> **From: bugzilla-daemon@kernel.org**  
> **Date: 2026-08-07 10:23:45**  
>  
> **Description:**  
> A proprietary kernel module named `mcfuzhu.ko` (author: "Veryhard") has been reported to cause kernel panic when loaded in a virtual machine. The module attempts to access `sys_call_table` and `kallsyms_lookup_name`, and uses `call_usermodehelper` to execute `systemd-detect-virt`. Full dmesg log attached.

帖子下方有一条回复，来自一个熟悉的ID：

> **“So someone is trying to sell a 'Minecraft cheat' as a proprietary kernel module that panics in a VM. And people actually paid for it.**
>
> **This is the stupidest thing I've seen this week.**
>
> **—— Linus Torvalds”**

---

## 第九幕：尾声（彩蛋）

一周后，小A在B站上刷到一个新视频：

> **《最新内核辅助7.2发布！全发行版兼容！拒绝恶意抹黑！》**

视频简介里写着：

> **“这次完全免费，三连即可获取！”**

小A点开评论区，看到置顶评论：

> **“此UP主之前被扒皮用内核模块骗钱，新号已举报。建议各位看看这个分析笔记：[链接]”**

小A笑了笑，点了举报。

---

**（终）**

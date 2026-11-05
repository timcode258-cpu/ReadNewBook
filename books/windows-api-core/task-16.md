# 任务16：锁顺序与锁外通知——避免互相等待

日期：2026-11-05。接任务15，建议阅读20分钟、两问5分钟；本课采用阅读与推演。先保存任务15答案，再读本份；15/16可连续完成后一起提交。

对应本书1.5关键段的两个注意事项：多锁顺序和减少持锁时间。本课不增加新的同步API，重点练“谁持有什么、谁正在等谁”。

**本课前提：** 代码按正常调用路径执行。同步窗口通知在同一进程内完成，线程B在当前窗口过程中读完传入的数据，然后返回。后面的局部副本示例按这条明确流程讲解。

## 一、先把任务15的前置接上

EnterCriticalSection正常返回，调用线程才占有该关键段。它在等待另一把锁时，不会自动放开已经占有的锁；程序必须安排对应的Leave。

本课场景：同一进程内使用两份已初始化的关键段g_cs1/g_cs2，按代码给出的Enter/Leave顺序执行。窗口通知部分由线程A发送、线程B处理。

## 二、原书的相反顺序

来源：王端明《深入浅出Windows API程序设计：核心编程篇》1.5，EPUB/xhtml/Chapter01_0006.xhtml，关键段注意事项（1）。下面摘出原书两个线程函数，省略函数间引导句，仅统一空白和缩进：

```cpp
DWORD WINAPI ThreadProc1(LPVOID lpParameter)
{
EnterCriticalSection(&g_cs1);
EnterCriticalSection(&g_cs2);
// 从资源1读取数据
// 向资源2写入数据
LeaveCriticalSection(&g_cs2);
LeaveCriticalSection(&g_cs1);
return 0;
}
DWORD WINAPI ThreadProc2(LPVOID lpParameter)
{
EnterCriticalSection(&g_cs2);
EnterCriticalSection(&g_cs1);
// 从资源1读取数据
// 向资源2写入数据
LeaveCriticalSection(&g_cs1);
LeaveCriticalSection(&g_cs2);
return 0;
}
```

原文解释：

> 线程2函数所做的改动是调换EnterCriticalSection和LeaveCriticalSection函数使用两个关键段对象的顺序，假设线程1开始运行并得到g_cs1关键段的所有权，然后执行线程2并得到g_cs2关键段的所有权，程序将发生死锁，当线程1和线程2中的任何一个试图继续执行时，都无法得到它需要的另一个关键段的所有权。

> 为了解决这个问题，我们必须在代码中以完全相同的顺序来获得关键段的所有权。调用LeaveCriticalSection函数时顺序则无关紧要，这是因为调用该函数从来不会使线程进入等待状态。

下面按四步顺序推演。执行到第四步，双方形成等待环。

## 三、用四步形成一条等待环

把g_cs1称为锁1，g_cs2称为锁2：

| 步骤 | 线程A | 线程B | 结果 |
|---:|---|---|---|
| 1 | 取得锁1 | — | A持有锁1 |
| 2 | — | 取得锁2 | B持有锁2 |
| 3 | 想取得锁2 | 持有锁2 | A等B释放锁2，但A仍拿着锁1 |
| 4 | 持有锁1 | 想取得锁1 | B等A释放锁1，但B仍拿着锁2 |

这四步执行完成后，A等B，B等A，双方都走不到后面的Leave。这就是死锁。

助手绘制的等待关系：

```mermaid
flowchart LR
    A[线程A<br/>持有锁1，等待锁2] --> B[线程B<br/>持有锁2，等待锁1]
    B --> A
```

Sleep只能改变时序，不能消除这个相反获取顺序造成的风险；等待线程结束也不会代替线程释放它持有的关键段。

## 四、统一获取顺序，怎样拆掉这个环？

同一组锁约定一致顺序，例如所有路径都先锁1、再锁2：

```cpp
EnterCriticalSection(&g_cs1);
EnterCriticalSection(&g_cs2);

// 需要同时保护的资源操作

LeaveCriticalSection(&g_cs2);
LeaveCriticalSection(&g_cs1);
```

如果A已拿到锁1，B就在获取锁1处等；B尚未拿走锁2，所以不会出现“A拿1等2，B拿2等1”的这条环。所有可能同时获取这两把锁的代码路径都要遵守约定，不能只改某一个函数。

本课统一采用：按锁1→锁2获取，按锁2→锁1释放，便于配对检查。

接着看第二种环：一把锁，加上一次同步窗口调用。

## 五、持有锁时SendMessage，可能等到谁？

同一节注意事项（2）的原文：

> （2）一个线程不要长时间独占共享资源。如果一个关键段被长时间独占，那么其他需要获得关键段所有权的线程只能进入等待状态，这会影响到应用程序的性能。下面的代码将在WM_SOMEMSG消息被发送到另一个窗口并得到处理前阻止其他线程修改g_struct结构的值：

原书展示在锁内SendMessage、再改为锁内复制数据、锁外SendMessage。以下用助手的具体场景解释：

- 工作线程A持有g_cs，保护一份共享状态。
- A用SendMessage向窗口W报告状态，等待窗口过程处理并返回。
- 窗口由线程B创建，窗口过程在线程B执行；B不必是主线程。
- B处理这条消息时也尝试进入同一个g_cs。

A在等B处理消息；B在等A交还关键段。A还没等到消息处理返回，就走不到Leave；B拿不到关键段，就完成不了消息处理。即使只用一把锁，也能形成等待环。

```mermaid
flowchart LR
    A[线程A<br/>持有g_cs<br/>等SendMessage处理返回] --> B[窗口线程B<br/>处理消息时等g_cs]
    B --> A
```

按这条调用链，A要等B处理完消息，B要先取得g_cs，等待环由此形成。

## 六、锁内复制快照，锁外交给窗口

“快照”就是把当时的数据复制一份，后面处理这份副本，不一直占着共享数据的锁。下面的Status只有两个普通整数，不含指向其他共享内容的指针：

```cpp
struct Status
{
    int done;
    int total;
};

Status g_status;
CRITICAL_SECTION g_cs;
#define WM_STATUS (WM_APP + 3)

void ReportStatus(HWND hwnd)
{
    Status snapshot;

    EnterCriticalSection(&g_cs);
    snapshot = g_status;  // 在保护下，复制一份一致的数据
    LeaveCriticalSection(&g_cs);

    // 不再持有g_cs；接收方必须在这次同步调用内读完
    SendMessage(hwnd, WM_STATUS, 0, (LPARAM)&snapshot);
}
```

这是核心通知函数。线程B的窗口过程按下面顺序处理WM_STATUS：读取参数中的Status，把它复制到B自己的变量，用B的副本更新显示，然后返回。snapshot在ReportStatus的作用域内存活；同步调用返回前，B已完成本次读取。

A等待窗口处理时已释放g_cs，B可以取得这把锁并继续。窗口显示的是复制时的状态；共享数据随后改变，snapshot中的两个整数保持不变。

## 七、为什么不能把上面的Send随手改成Post？

snapshot是ReportStatus里的局部对象。PostMessage只投递消息，不保证接收方在ReportStatus返回前处理：

```text
发送线程：把&snapshot投递出去
发送线程：ReportStatus返回，snapshot寿命结束
窗口线程：稍后才读消息里的地址 —— 地址所指对象已失效
```

强制类型转换只是把地址放入参数，不会替你复制整份Status数据。想改成异步投递，就必须同时安排数据持续有效和由谁清理；如果只是进度等小整数，可像任务13那样直接按值放进消息参数。

锁外通知按两条安排执行：A先释放g_cs再发送；B持续处理窗口消息，处理期间不等待A结束。这样拆开了本例的等待环。

本节选段没有新的原书图，上方关系图由助手绘制。先理解两种环，SRW锁与条件变量后续再展开。

## 八、两问（遮住正文后回答）

1. A先取得L1再申请L2，B先取得L2再申请L1。给出能形成死锁的一个交错，说明两者分别持有什么、等待什么。如果所有路径都改为先L1再L2，为什么可以避免这一条环？在A取L1后加Sleep，能否替代这个顺序约定？

我的回答与理由：

信心（高/中/低）：

2. 工作线程持有g_cs后SendMessage给线程B创建的窗口，B处理时也要进入g_cs。说明等待环，并提出怎样复制数据、释放锁再通知。若副本是函数内局部Status，能否只把SendMessage改成PostMessage、仍传这个局部副本的地址就完成修复？为什么？

我的回答与理由：

信心（高/中/低）：

实际用时（阅读/答题，未计时写未记录）：

是否回看材料/用了什么提示（是否先保存任务15答案再读本份）：

桌面保存后告诉我“任务16完成”，或手机聊天按题号作答；15/16可以一起交。静态推理、延迟保持和实际运行仍分别记录。

## 官方核对

2026-11-05核对：[多关键段的一致锁顺序](https://learn.microsoft.com/en-us/windows/win32/sync/using-critical-section-objects)、[SendMessage的跨线程处理与等待](https://learn.microsoft.com/en-us/windows/win32/api/winuser/nf-winuser-sendmessagew)、[PostMessage异步投递](https://learn.microsoft.com/en-us/windows/win32/api/winuser/nf-winuser-postmessagew)。原文与原文代码来自用户提供的EPUB；等待环、快照代码及寿命推演为助手解释。

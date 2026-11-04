# 任务15：关键段——保护一整段共享操作

日期：2026-11-04。接任务14，建议阅读20分钟、两问5分钟；继续静态学习，不要求运行。先保存任务14答案，再读本份；14/15可连续完成后一起提交。

对应本书1.5“2．关键段”。目标：理解所有参与线程进入同一个关键段，为什么能保护“检查条件并更新数据”这一整段；分清初始化、进入、离开和最终销毁。

## 一、先看完整问题：检查后扣减余额

这是一个程序模型，不是实际账户系统：共享余额为100，A/B都想扣80，程序规则是“余额足够才扣减”。只看单线程，先判断、再扣减很自然；但若两个线程各自检查后就继续，可能都根据旧的100认定可以扣80。

关键问题是：A检查时成立的条件，在A真正修改之前可能已被B改变。要让“检查余额 → 确认够用 → 扣减”作为一个受保护的区间执行。不能只把最后的单次数值更新换成原子操作，就认定前面的条件检查也一起受保护。

以上是助手的逻辑模型。无同步的真实C++共享读写还可能有数据竞争、未定义行为，不能按这段描述枚举其所有实际运行结果。下面直接学习正确的共同保护安排。

## 二、对照原书，先厘清保护对象

来源：王端明《深入浅出Windows API程序设计：核心编程篇》1.5，EPUB/xhtml/Chapter01_0006.xhtml，“2．关键段”首段。以下是完整原文，保留措辞：

> 关键段（Critical Section）对象也称为临界区对象，即把操作共享资源的一段代码保护起来，当一个线程正在执行操作共享资源的这段代码时，其他试图访问共享资源的线程都将被挂起，一直等待到前一个线程执行完，其他线程才可以执行操作共享资源的代码。当然，系统也可以暂停当前线程去调度其他线程，但是在当前线程离开关键段前，系统是不会去调度任何想要访问同一资源的其他线程的。

助手结合当前官方说明补充：这段要表达的是“互斥访问”，其中关于系统不会调度其他线程的说法不能按字面推广。**持有关键段的线程仍可能被切走，其他线程仍可执行无关代码；其他线程在尝试进入同一个已被占用的关键段时，才需要等待。**

系统不会自动识别“这个变量已经有人保护”，也不会拦住绕过关键段的普通读写。关键段的作用依赖所有相关参与者使用同一份保护规则。

本节没有单独的新原书配图，下方时间线由助手绘制。

## 三、先认清四个动作

原书列出的四个接口，以下省略SAL注解并统一空白，保留函数名与参数类型：

```cpp
VOID WINAPI InitializeCriticalSection(LPCRITICAL_SECTION lpCriticalSection);
VOID WINAPI EnterCriticalSection(LPCRITICAL_SECTION lpCriticalSection);
VOID WINAPI LeaveCriticalSection(LPCRITICAL_SECTION lpCriticalSection);
VOID WINAPI DeleteCriticalSection(LPCRITICAL_SECTION lpCriticalSection);
```

这里省略原型中的SAL注解，仅保留参数类型和函数名；不是改动接口行为。LPCRITICAL_SECTION就是指向CRITICAL_SECTION结构的指针，传入&g_cs就是把同一个对象的地址交给API。

| 动作 | 谁在何时做 | 具体作用 |
|---|---|---|
| Initialize | 主线程，在任何使用者开始前做一次 | 初始化这份关键段对象 |
| Enter | 每个要访问受保护数据的线程 | 请求占有同一个关键段；若其他线程占有，就等待 |
| Leave | 成功进入的持有线程，在本段操作结束时做 | 交还占有权，允许其他参与者进入 |
| Delete | 确认所有使用者不再使用后做一次 | 最终释放关键段相关资源 |

这四个接口没有BOOL返回值，不套用“非0成功”的写法。Enter正常返回，才表示已取得占有权；它不是“发出申请后立即返回，稍后再占有”。关键段用于同一进程的线程，本课不把它当跨进程共享工具。

## 四、把检查和扣减放在同一个区间

助手的教学函数，假设调用金额为正、初始化已完成、所有对余额的并发访问都遵循同一个g_cs，且本例整数计算没有异常路径：

```cpp
CRITICAL_SECTION g_cs;
int g_balance = 100;

bool TryWithdraw(int amount)
{
    bool success = false;

    EnterCriticalSection(&g_cs);

    if (g_balance >= amount)   // 检查也必须在保护区间内
    {
        g_balance -= amount;
        success = true;
    }

    LeaveCriticalSection(&g_cs);
    return success;
}
```

A/B都调用同一个函数、传入80。假定这次A先进入：

```text
A：Enter返回，已经占有g_cs
A：检查100足够，扣80，余额变20
B：尝试Enter同一个g_cs；若A还没Leave，B在这里等待
A：Leave，交还占有权
B：Enter返回，此时才检查余额，看到20不足80，不扣减
B：Leave
```

先进入的也可能是B；在这两个80元请求的例子中，最终都是一人成功、余额20，不保证哪条线程成功。

助手绘制的一种可能顺序：

```mermaid
sequenceDiagram
    participant A as 线程A
    participant C as 同一个g_cs
    participant B as 线程B
    A->>C: Enter，取得占有权
    B->>C: Enter，等待占有权
    Note over A: 检查并扣减：100变20
    A->>C: Leave
    C-->>B: B取得占有权，Enter返回
    Note over B: 检查20不足80，不扣减
    B->>C: Leave
```

两条线程必须传入同一个&g_cs。若A用csA、B用csB，各自的Enter都可能成功，但这两把不同的锁并不能互相挡住，无法保护同一个余额。

## 五、为什么不能先检查，再进入？

```text
错误范围：先检查余额足够 → Enter → 扣减 → Leave
正确范围：Enter → 检查余额是否足够 → 决定是否扣减 → Leave
```

前一种安排中，两个线程仍可能在占有关键段之前都通过检查；只保护扣减不保护决策，无法保证“余额足够才扣”。正确范围要覆盖这条规则所依赖的整个读、判断、更新过程。

任何需要与更新一致的并发读取，也应遵循对应的保护方案。只让写线程进入、读线程直接读，不会自动获得同样保证。

## 六、谁初始化，谁最后销毁？

下面用控制台情境串起完整生命周期，假设各线程正常结束、所列调用成功：

```text
主线程：InitializeCriticalSection(&g_cs)
主线程：创建A/B，让它们分别调用TryWithdraw
A/B：各次成功Enter后，在操作结束时对应Leave
主线程：分别等待A/B，检查结果，确认它们已经结束
主线程：关闭不再使用的线程句柄
主线程：确认不再有线程使用、等待或占有g_cs
主线程：DeleteCriticalSection(&g_cs)
```

原书说明离开与销毁时的两段原文：

> 执行完操作共享资源的代码后，需要调用LeaveCriticalSection函数释放对关键段对象的所有权，以便其他正在等待的线程获得关键段对象的所有权并执行操作共享资源的代码。

> 不再需要关键段对象时需要调用DeleteCriticalSection函数释放关键段对象，该函数会释放关键段对象使用的所有系统资源。

Leave是每次操作结束时交还占有权，Delete是对象生命周期结束时最终清理。一个线程Leave，不代表其他线程已不再需要这个对象，不能立刻据此Delete。

CRITICAL_SECTION不是CreateEvent返回的那类句柄，不能用CloseHandle替代DeleteCriticalSection。初始化后的结构也不要随意复制、移动或重复初始化。

本例让函数在Leave后才return，避免某条正常分支漏掉Leave。更复杂代码还要考虑异常或提前退出时交还占有权，后面用作用域管理进一步说明。持有关键段时保持操作短小，避免在里面等另一个可能也需要它的线程；窗口调用/回调也可能形成等待环，应安排在合适的锁外位置。

关键段允许同一持有线程再次进入；每次进入都要对应一次离开。它不保证等待线程按先来先到的顺序取得占有权。本课先掌握一次进入/一次离开的基本模式，不要求展开自旋与重入细节。

## 七、两道短题（遮住正文后回答）

1. 余额初始100。A想扣70，B想扣60；它们都通过同一份已初始化的g_cs，把“检查够不够及扣减”全部放在Enter/Leave之间。没有其他访问，主线程最后成功等两者结束。能否两人都扣成功？最终余额可能是多少？是否保证A先成功？说明理由。

我的回答与理由：

信心（高/中/低）：

2. A用csA、B用csB保护同一个余额，各自初始化并进入自己的关键段，这能保证双方互斥吗？应该怎样改？另外，主线程仅知道自己已经Leave，就立刻Delete那份仍会被工作线程使用的关键段，是否可行？给出正确清理时机。

我的回答与理由：

信心（高/中/低）：

实际用时（阅读/答题，未计时写未记录）：

是否回看材料/用了什么提示（是否先保存任务14答案再读本份）：

桌面保存后告诉我“任务15完成”，或在手机聊天按题号作答；14/15可以一起交。原子递增、关键段区间、延迟保持和独立运行分开记录，预先拿到材料不表示已通过。

## 官方核对

2026-11-04核对：[Critical Section Objects的单进程范围、互斥及线程仍可运行](https://learn.microsoft.com/en-us/windows/win32/sync/critical-section-objects)、[EnterCriticalSection占有权与正常返回](https://learn.microsoft.com/en-us/windows/win32/api/synchapi/nf-synchapi-entercriticalsection)、[InitializeCriticalSection初始化与对象管理](https://learn.microsoft.com/en-us/windows/win32/api/synchapi/nf-synchapi-initializecriticalsection)、[DeleteCriticalSection清理前提](https://learn.microsoft.com/en-us/windows/win32/api/synchapi/nf-synchapi-deletecriticalsection)。原书文字与助手示例/图/官方纠正分别标注，不把未运行片段当作用户实践。

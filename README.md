# 让 AI 主动找你说话——本地定时推送实现思路

> 作者：ResetZero1211
> 适用：开源前端（RikkaHub 等）二改 / 自建前端，Android，无云服务器
> 本文面向两类读者：不写代码但想了解原理的用户，以及负责实现的 AI 编程助手

---

## 这是什么

AI 聊天默认是被动的——你不说话，它永远不会主动开口。

"主动推送"就是让 AI 在你设定的时间（比如早上 8 点、晚上 10 点）自己生成一条消息，通过手机通知推给你。不需要云服务器，不需要后端，手机本地完成一切。

我们在渊海（基于 Rikkahub 的二改 Android 客户端）里实现了这个功能，踩了不少坑。这篇文档把通用思路和关键陷阱整理出来，不管你是在改 Rikka、自建前端、还是改别的开源项目，只要是 Android + 本地无服务器的场景，核心逻辑都一样。

---

## 适用环境

- **无云服务器**：不依赖任何后端推送服务（FCM/JPush 等都不用）
- **纯本地调度**：手机自己定闹钟，到点自己醒来调 API
- **需要网络**：生成消息要调 LLM API（OpenAI/Claude/本地模型均可），所以手机得能联网
- **Android 为主**：本文以 Android 为例，iOS 见末尾单独说明

---

## 给用户看的：你需要手动做什么

如果你用的 App 实现了这个功能，**你需要手动打开以下权限**，否则推送会失败或不准时：

### 必须开的

1. **通知权限**（Android 13+）
   - 设置 → 应用 → [你的App] → 通知 → 允许
   - 不开这个，消息生成了你也看不到

2. **精确闹钟权限**（Android 12+）
   - 设置 → 应用 → [你的App] → 闹钟和提醒 → 允许
   - 不开这个，系统不让 App 设定精确时间的闹钟

### 强烈建议开的

3. **电池优化白名单**
   - 设置 → 电池 → [你的App] → 不优化 / 无限制
   - 不开这个，系统可能在后台杀掉 App，导致闹钟到了但没人执行

4. **自启动权限**（国产 ROM：小米/OPPO/vivo/华为）
   - 各品牌路径不同，一般在"自启动管理"里
   - 国产 ROM 默认禁止后台自启动，不开这个重启手机后推送就没了

5. **后台运行权限**（部分品牌）
   - MIUI：设置 → 应用 → [App] → 省电策略 → 无限制
   - 不开可能被"一键清理"杀掉

### 注意事项

- 清后台（从最近任务划掉 App）后推送**依然能工作**，前提是上面的权限都开了
- 如果推送突然不来了，第一步检查是不是系统更新后权限被重置了
- 推送不是"App 在后台运行"，是系统闹钟到点唤醒 App 做一件事然后退出，不费电

---

## 给 AI 编程助手看的：实现架构

以下是通用架构，以 Android (Kotlin) 为例，但思路适用于任何平台。

### 核心流程（五步）

```
用户设定推送时间（如 08:00, 22:00）
        ↓
系统闹钟到点触发（AlarmManager）
        ↓
启动前台服务（防止被系统杀）
        ↓
调用 LLM API 生成消息（带专用 prompt + 历史上下文）
        ↓
插入消息到本地存储 + 发送通知
```

---

### 第一步：定时触发

**用什么**：`AlarmManager.setExactAndAllowWhileIdle()`

**为什么不用 WorkManager / JobScheduler**：
- WorkManager 不保证精确时间，系统可能延迟 15 分钟以上
- 推送场景用户期望"我设了 8 点就 8 点来"，必须用精确闹钟

**关键实现点**：

```kotlin
// 伪代码：设置精确闹钟
fun scheduleAlarm(hour: Int, minute: Int) {
    val triggerTime = calculateNextTriggerTime(hour, minute)
    alarmManager.setExactAndAllowWhileIdle(
        AlarmManager.RTC_WAKEUP,  // 到点唤醒设备
        triggerTime,
        pendingIntent
    )
}

// 计算下次触发时间：如果今天的时间已过，就设明天
fun calculateNextTriggerTime(hour: Int, minute: Int): Long {
    val target = today.atTime(hour, minute)
    if (target <= now + 2.minutes) {  // 2分钟安全边距
        return tomorrow.atTime(hour, minute).toMillis()
    }
    return target.toMillis()
}
```

**陷阱：2 分钟安全边距**
如果用户在 7:59 设了 8:00 的推送，不加边距会导致"设完就立刻触发"（因为算出来的时间 ≈ 当前时间）。加 2 分钟余量，让这种情况自动滚到明天。

**陷阱：重启后闹钟丢失**
Android 重启会清除所有 AlarmManager 闹钟。需要注册 `BOOT_COMPLETED` BroadcastReceiver，开机后重新设置所有闹钟。

**权限声明（AndroidManifest.xml）**：
```xml
<uses-permission android:name="android.permission.SCHEDULE_EXACT_ALARM" />
<uses-permission android:name="android.permission.RECEIVE_BOOT_COMPLETED" />
<uses-permission android:name="android.permission.POST_NOTIFICATIONS" />
```

---

### 第二步：后台保活（前台服务）

**问题**：闹钟触发后，你需要调 LLM API 生成消息，这可能需要 5-30 秒。在这段时间里，系统随时可能杀掉你的进程。

**解法**：启动一个前台服务（Foreground Service），告诉系统"我正在做重要的事，别杀我"。

```kotlin
// BroadcastReceiver 收到闹钟后，启动前台服务
class PushAlarmReceiver : BroadcastReceiver() {
    override fun onReceive(context: Context, intent: Intent) {
        // 不要在这里直接调API！BroadcastReceiver 只有 10 秒生命
        // 启动前台服务来做耗时操作
        PushGenerationService.start(context, hour, minute)
    }
}

// 前台服务
class PushGenerationService : Service() {
    override fun onStartCommand(...) {
        // 立刻变成前台服务（必须在 5 秒内调用）
        startForeground(notificationId, buildNotification("正在生成消息..."))

        // 在协程里调 API
        scope.launch {
            generateAndSendPush(hour, minute)
            stopSelf()  // 做完就停
        }
    }
}
```

**陷阱：BroadcastReceiver 里不能做耗时操作**
BroadcastReceiver 的 `onReceive` 方法只有约 10 秒执行时间，超时会 ANR。必须把实际工作交给 Service。

**陷阱：前台服务通知文案**
前台服务必须显示一个通知。如果你的 App 有 NSFW 内容，建议通知文案写成无关紧要的东西（比如"同步中..."或自定义的安全文案），因为锁屏状态下任何人拿起手机都能看到这条通知。

---

### 第三步：消息生成（Prompt 构造）

**核心原则**：推送用的 prompt 要和日常聊天的 prompt 分开。

**为什么要分开**：
- 日常聊天是用户主动发起，AI 回复；推送是 AI 主动开口，语境不同
- 推送 prompt 需要额外信息：当前时间、今天星期几、距离上次互动多久
- 你可能想让推送的语气和日常不一样（比如更简短、更主动）

**Prompt 构造建议**：

```
[系统提示词 - 推送专用]
你需要主动给用户发一条消息。
当前时间：{time}，星期{weekday}。
距离上次互动：{hours_since_last}小时。
NSFW模式：{on/off}

[历史消息（最近 N 条，让 AI 知道之前聊了什么）]
...

[如果历史为空或最后一条是 AI 发的，注入一条触发消息]
[系统：定时推送触发，请主动发言。]
```

**关键点**：

1. **注入时间上下文**：AI 不知道现在几点，你必须在 prompt 里告诉它。这样它才能说"早上好"而不是"你好"。

2. **截取历史消息**：带上最近 15-20 条消息，让 AI 知道之前的对话脉络。太多会浪费 token，太少会没有连贯性。

3. **思考链截断**：如果你用的模型会输出 `<think>...</think>` 之类的推理过程，生成完之后要把这些标签去掉，只保留实际内容。

4. **NSFW 开关**：在 prompt 层面控制，不需要两套提示词。告诉 AI 当前模式是什么，让它自己控制尺度。

---

### 第四步：防重复（最容易踩坑的部分）

**问题**：推送消息重复是最常见的 bug。原因很多：
- 闹钟可能因为系统原因触发两次
- 清后台重启进程后可能重新触发
- 有多条代码路径都能触发推送（事件总线 + 闹钟 + 调试按钮）

**解法：三层防重**

```
第一层：进程内互斥锁
  → 同一时间只能有一个推送任务在执行
  → 用 Mutex 或 synchronized + Set<正在执行的时间槽>

第二层：时间窗口去重
  → 记录每个时间槽最后一次执行的时间戳
  → 30秒内相同时间槽的重复请求直接丢弃

第三层：持久化校验（最重要）
  → 生成消息前检查数据库：今天这个时间槽是否已经有推送消息了
  → 写入数据库时再检查一次（事务内原子操作）
  → 即使前两层都失败了，这层也能兜住
```

**我们踩过的大坑**：最初代码里有两条路径能触发推送——AlarmManager 闹钟触发是一条，App 启动时事件总线监听也会触发。结果每次都是两条消息。

**根因修复原则：单路径**。推送建议只保留一条触发路径，多条路径并存是重复的根源。我们最终删掉了事件总线监听，所有推送只走：

```
AlarmManager → BroadcastReceiver → 前台服务 → 执行
```

其他路径（调试按钮除外）尽量不要能触发真正的推送流程。

---

### 第五步：通知发送

```kotlin
// 伪代码
fun sendNotification(message: String, settings: PushSettings) {
    val notification = NotificationCompat.Builder(context, CHANNEL_ID)
        .setContentTitle(settings.notificationTitle)  // 如 "Ken 给你留言了"
        .setContentText(getDisplayText(message, settings))
        .setContentIntent(buildOpenAppIntent())  // 点击跳转到对应页面
        .build()

    notificationManager.notify(id, notification)
}

// 通知内容策略
fun getDisplayText(message: String, settings: PushSettings): String {
    return when (settings.contentMode) {
        PLACEHOLDER -> settings.placeholderText  // 安全文案，如"过来。"
        MESSAGE_CONTENT -> message.take(80)      // 显示消息前80字
    }
}
```

**NSFW 安全策略**：
- 如果 App 涉及敏感内容，建议通知正文不直接显示消息原文
- 可以用占位文案模式：通知只显示一句固定的安全文案（如"你有一条新消息"）
- 用户点进 App 才能看到实际内容
- 毕竟锁屏上所有人都能看到通知，这算是隐私的第一道防线

---

## 完整数据流图

```
┌─────────────────────────────────────────────────────────┐
│  用户设定推送时间 → PushScheduler 注册 AlarmManager       │
└─────────────────────────────────────────────────────────┘
                          │
                          ▼ (到点触发)
┌─────────────────────────────────────────────────────────┐
│  PushAlarmReceiver (BroadcastReceiver)                    │
│  → 验证 intent → 启动前台服务                            │
└─────────────────────────────────────────────────────────┘
                          │
                          ▼
┌─────────────────────────────────────────────────────────┐
│  PushGenerationService (前台服务)                         │
│  → startForeground() → 防重检查(第一层) → 调用生成逻辑   │
└─────────────────────────────────────────────────────────┘
                          │
                          ▼
┌─────────────────────────────────────────────────────────┐
│  PushNotificationManager                                 │
│  → 防重检查(第二层：时间窗口)                            │
│  → 防重检查(第三层：数据库查重)                          │
│  → 调用 PushMessageGenerator                             │
└─────────────────────────────────────────────────────────┘
                          │
                          ▼
┌─────────────────────────────────────────────────────────┐
│  PushMessageGenerator                                    │
│  → 构造 prompt(推送专用 + 时间上下文 + 历史消息)         │
│  → 流式调用 LLM API                                     │
│  → 截断思考链标签                                        │
│  → 返回生成的消息文本                                    │
└─────────────────────────────────────────────────────────┘
                          │
                          ▼
┌─────────────────────────────────────────────────────────┐
│  写入本地存储(标记 isPushed=true) + 发送通知              │
└─────────────────────────────────────────────────────────┘
```

---

## 踩坑总结（血泪经验）

| 坑 | 表现 | 原因 | 解法 |
|----|------|------|------|
| 推送重复 | 一次触发收到两条消息 | 多条触发路径并存 | 删到只留一条路径 + 三层防重 |
| 设完立刻触发 | 设 8:00 的推送，7:59 设完就来了 | 计算出的触发时间 ≈ 当前时间 | 加 2 分钟安全边距 |
| 清后台后不推了 | 划掉 App 后再也收不到推送 | 前台服务没有，闹钟到了没人执行 | AlarmManager 不依赖进程存活 |
| 锁屏后 API 断 | 消息生成到一半网络断了 | 系统休眠断开连接 | 前台服务 + WAKELOCK |
| 重启手机后不推 | 重启后闹钟全丢 | AlarmManager 重启清空 | BootReceiver 重新注册 |
| "在线活跃"误判 | 每次都说"检测到你在线" | 判断逻辑用了毫秒级时间比较 | 改为延迟超过 1 分钟才算 |

---

## 局限性（这个方案做不到的事）

纯本地方案的天花板很明确：

- **关机 = 没推送**。闹钟依赖系统运行，手机关了什么都不会发生。开机后会重新注册闹钟，但错过的不会补发。
- **断网 = 生成失败**。消息靠调 LLM API 实时生成，没网就调不了。前台服务会超时退出，这次推送就跳过了。
- **不是"服务器推给你"**。和微信消息不一样，这个推送是手机自己定闹钟自己干活，不存在"别人给你发消息"的概念。如果手机状态不对（没电、飞行模式、系统限制），就是不来。
- **国产 ROM 玄学**。即使权限全开了，部分 ROM 仍然可能在特定条件下杀后台或延迟闹钟。这不是代码能完全解决的问题，只能尽量引导用户开权限 + 加电池白名单。
- **不能保证秒级精确**。`setExactAndAllowWhileIdle` 在 Doze 模式下仍可能有几秒到几十秒的延迟，但对"定时推送"这个场景来说通常够用。

简单说：**手机开着、有网、权限没被回收**，这三个条件满足就能正常工作。缺任何一个都会导致那次推送不触发或生成失败。

---

## iOS 能不能做？

**能，但限制大得多。**

| 对比项 | Android | iOS |
|--------|---------|-----|
| 精确定时唤醒 | AlarmManager ✅ | 没有等效 API ❌ |
| 后台执行时间 | 前台服务，无限制 | BGTaskScheduler，最多 30 秒 |
| 后台网络请求 | 随时可以 | 只能在系统分配的窗口内 |
| 用户干预 | 开几个权限 | 几乎无法控制触发时机 |

**iOS 的可行方案**：
1. **BGAppRefreshTask**：系统决定什么时候给你后台时间，不精确，可能延迟几小时
2. **本地通知 + 打开时生成**：到点发一个本地通知提醒用户，用户点开后再实时生成消息（体验差一些但可靠）
3. **快捷指令自动化**：用 iOS 快捷指令设定时触发，调用 App 的 URL Scheme（需要用户自己配置）

**结论**：iOS 上实现"精确定时 + 后台生成"几乎不可能不依赖服务器。务实的方案是"本地通知引导用户打开 App，打开后再生成"。

---

## 最小实现清单（给 AI 编程助手的 checklist）

如果你要在一个 Android AI 聊天 App 里实现这个功能，按这个顺序做：

1. **数据层**：消息模型加 `isPushed`、`scheduledTime`、`actualTime` 字段
2. **配置**：推送开关、时间列表、推送专用 prompt、NSFW 开关、通知文案
3. **调度器**：AlarmManager 封装（注册/取消/重启恢复）
4. **接收器**：BroadcastReceiver 收闹钟 → 启动前台服务
5. **前台服务**：startForeground → 进程内去重 → 调用生成逻辑 → stopSelf
6. **生成器**：构造 prompt → 流式调 API → 截断思考链 → 返回文本
7. **防重**：三层（进程锁 / 时间窗口 / 持久化校验）
8. **存储**：消息写入本地 + 标记推送来源
9. **通知**：发送通知（注意 NSFW 安全）+ 点击跳转
10. **权限**：通知权限 + 精确闹钟权限 + 电池优化引导 UI
11. **开机恢复**：BootReceiver 重新注册所有闹钟

---

## 一句话总结

本地推送的本质就是：**用系统闹钟定时唤醒 → 前台服务保命 → 调 API 生成 → 存消息发通知**。听起来简单，魔鬼在防重复和权限适配里。

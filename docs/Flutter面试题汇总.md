# Flutter 面试题汇总（个人笔记整理）

> 来源：本地截图笔记 + `iOS笔试题.docx` 中与 Flutter/Dart 相关的题目与答案思路。  
> 下文在保留原意的基础上做了结构化，并补充 **Dart 语法与关键词** 便于对照复习。

---

## 目录

1. [StatefulWidget 生命周期](#1-statefulwidget-生命周期)
2. [InheritedWidget](#2-inheritedwidget)
3. [Key 的类型与使用场景](#3-key-的类型与使用场景)
4. [Isolate 与并发](#4-isolate-与并发)
5. [手写题：随机方块不重叠（Flutter）](#5-手写题随机方块不重叠flutter)
6. [Dart 语法与关键词扩展](#6-dart-语法与关键词扩展)
7. [面试问答速记（Q&A）](#7-面试问答速记qa)
8. [附录：同份笔试材料中的通用题目（非 Flutter 专向）](#8-附录同份笔试材料中的通用题目非-flutter-专向)

---

## 1. StatefulWidget 生命周期

### 1.1 各阶段回调（按调用顺序理解）

| 阶段 | 回调 | 要点 |
|------|------|------|
| 创建 State | `createState` | `StatefulWidget` 被使用时立刻执行，用于创建对应的 `State`。 |
| 初始化 | `initState` | 给 `State` 成员赋初值；可发起异步请求，拿到数据后 **`setState`** 更新 UI。 |
| 依赖变化 | `didChangeDependencies` | 当本 `State` 在 `build` 中依赖的 **`InheritedWidget`**（等）发生变化时会调用；主题、`Locale` 等变化也会走这里。 |
| 构建 | `build` | 返回要渲染的子树；**会被多次调用**，只应做「描述 UI」的逻辑，避免副作用。 |
| Debug | `reassemble` | **仅 debug**：热重载（hot reload）时会调用，可放调试辅助代码。 |
| 父级重建 | `didUpdateWidget` | 框架用 `Widget.canUpdate` 比较同一位置新旧节点；若 **Key 与 runtimeType 都相同** 则 `canUpdate` 为 `true`，随后会调 `didUpdateWidget`，**之后一定会再 `build`**。 |
| 暂时移出树 | `deactivate` | 从树上摘下时调用；若不再挂到别处，接下来会 `dispose`。 |
| 销毁 | `dispose` | 永久移除，释放控制器、`Animation`、`Stream` 订阅等资源。 |

### 1.2 一帧绘制后的回调：`addPostFrameCallback`

- 在当前帧 **绘制完成后** 执行一次，**注册后不能取消**，且 **只回调一次**。
- 定义在 **`SchedulerBinding`**；由于 `WidgetsBinding` **`mixin WidgetsBinding on SchedulerBinding`**，两种写法等价思路：

```dart
SchedulerBinding.instance.addPostFrameCallback((_) {
  // 首帧布局完成后再量尺寸、滚动到底部等
});

WidgetsBinding.instance.addPostFrameCallback((_) {
  // 同上
});
```

### 1.3 生命周期归纳（四个大块）

1. **初始化**：`createState` → `initState`
2. **首次进入树后的创建链路**：`didChangeDependencies` → `build`
3. **再次触发 `build` 的常见原因**：`setState`、`didChangeDependencies`、`didUpdateWidget`（以及父组件重建导致子 `build`）
4. **销毁**：`deactivate` → `dispose`

---

## 2. InheritedWidget

### 2.1 是什么

- `InheritedWidget` 是一种特殊的 `Widget`，在 **子树向下** 传递数据，**不必层层构造函数传参**。
- 数据变化时，可 **`updateShouldNotify`** 决定是否通知依赖者，依赖方通过 `context.dependOnInheritedWidgetOfExactType<T>()` **注册依赖**，从而在数据变化时 **自动重建**。

### 2.2 最小示例（与笔记一致）

```dart
import 'package:flutter/material.dart';

class MyData extends InheritedWidget {
  const MyData({
    super.key,
    required this.counter,
    required super.child,
  });

  final int counter;

  static MyData? of(BuildContext context) {
    return context.dependOnInheritedWidgetOfExactType<MyData>();
  }

  @override
  bool updateShouldNotify(MyData oldWidget) => counter != oldWidget.counter;
}
```

### 2.3 面试常追问

- **与 `Provider` / `Riverpod` 的关系**：状态管理库多在 `InheritedWidget`（或类似机制）之上封装 **更新粒度、依赖收集、异步与测试**。
- **`Element` 与依赖**：`dependOnInheritedWidgetOfExactType` 会在 `Element` 上登记依赖，通知时精准重建相关子树。

---

## 3. Key 的类型与使用场景

### 3.1 顶层分类

- `Key` 的常见子类：**`LocalKey`**、**`GlobalKey`**。
- `LocalKey` 再分为：`ValueKey`、`ObjectKey`、`UniqueKey`。

### 3.2 LocalKey 三种

| 类型 | 标识依据 | 典型用途 |
|------|----------|----------|
| `ValueKey` | **值相等**（`==` / `hashCode`） | 列表项用稳定业务 id：`ValueKey(todo.id)` |
| `ObjectKey` | **对象身份**（引用同一实例） | 以「对象本身」区分身份时 |
| `UniqueKey` | **每次 new 都不同** | 强制每次重建都是新实例（慎用，易抖动） |

### 3.3 GlobalKey（补充）

- **全局**定位到某个 `Element` / `State`，可跨分支取 `State`、`context`，或拿 `RenderObject` 做测量。
- 代价：打断局部复用、增加查找与维护成本；**不要滥用**。

### 3.4 列表示例

```dart
ListView(
  children: [
    ListTile(key: ValueKey(1), title: const Text('Item 1')),
    ListTile(key: ObjectKey(myObject), title: const Text('Item 2')),
    ListTile(key: UniqueKey(), title: const Text('Item 3')),
  ],
);
```

---

## 4. Isolate 与并发

### 4.1 模型要点（面试高频）

- Dart **默认单线程事件循环**；`Future` / `async` **不**等于多核并行。
- **`Isolate`** 才是 **多内存堆** 的并行单元：**不共享可变内存**，靠 **`SendPort` / `ReceivePort`** 传消息（可序列化对象）。
- `Isolate.spawn` 的入口必须是 **顶层函数** 或 **`static` 方法**（与截图笔记一致）。

### 4.2 最小 spawn 示例（主 Isolate 收消息）

```dart
import 'dart:isolate';

Future<void> main() async {
  final receivePort = ReceivePort();

  await Isolate.spawn(isolateEntry, receivePort.sendPort);

  receivePort.listen((message) {
    print('Received message from isolate: $message');
  });
}

void isolateEntry(SendPort replyToMain) {
  replyToMain.send('hello from worker');
}
```

### 4.3 双向通信思路（笔记中的「握手 + 单次请求 Future」）

- Worker 启动后先把 **自己的 `SendPort`** 发回主 Isolate（握手）。
- 主 Isolate 用 **临时 `ReceivePort`** 把「本次请求的回调 `SendPort`」随参数发给 Worker，从而把一次往返包成 **`Future`**（`receivePort.first`）。

涉及 **`await for`**：对 `ReceivePort` 监听时，可按流式处理多条请求。

---

## 5. 手写题：随机方块不重叠（Flutter）

**题意**：在 **300×600** 区域内放置 **随机数量** 的 **50×50** 正方形，**互不重叠**（可用 OC / Swift / Flutter 任一）。

**思路摘要**：

- 若不要随机坐标，可用规则网格（如 `GridView`）直接铺满。
- 随机坐标：在合法范围内随机 `(x, y)`，用 `Rect` 与已有矩形 **`overlaps`** 检测；重叠则重试，并对失败次数设上限避免死循环。

```dart
import 'dart:math';
import 'package:flutter/material.dart';

void main() => runApp(const MaterialApp(home: Scaffold(body: Center(child: NonOverlappingSquares()))));

class NonOverlappingSquares extends StatefulWidget {
  const NonOverlappingSquares({super.key});

  @override
  State<NonOverlappingSquares> createState() => _NonOverlappingSquaresState();
}

class _NonOverlappingSquaresState extends State<NonOverlappingSquares> {
  final List<Rect> squarePositions = <Rect>[];
  int overCount = 0;

  @override
  void initState() {
    super.initState();
    final random = Random();
    // 粗算最多约 12 * 6 = 72 个互不重叠的 50x50（依实现与随机性而定）
    generateSquarePositions(random.nextInt(72));
  }

  void generateSquarePositions(int count) {
    final random = Random();
    squarePositions.clear();
    overCount = 0;

    while (squarePositions.length < count) {
      final x = random.nextDouble() * (300 - 50);
      final y = random.nextDouble() * (600 - 50);
      final rect = Rect.fromLTWH(x, y, 50, 50);

      if (!checkOverlap(rect)) {
        squarePositions.add(rect);
      } else {
        overCount += 1;
        if (overCount > count) break;
      }
    }

    setState(() {});
  }

  bool checkOverlap(Rect rect) {
    for (final existing in squarePositions) {
      if (existing.overlaps(rect)) return true;
    }
    return false;
  }

  @override
  Widget build(BuildContext context) {
    return Container(
      width: 300,
      height: 600,
      color: Colors.white,
      child: Stack(
        children: [
          for (final r in squarePositions)
            Positioned(
              left: r.left,
              top: r.top,
              child: Container(width: 50, height: 50, color: Colors.yellow),
            ),
        ],
      ),
    );
  }
}
```

---

## 6. Dart 语法与关键词扩展

下面按「上面笔记里出现或强相关」整理，方便背诵与串联。

### 6.1 异步

| 语法 | 含义 |
|------|------|
| `async` | 标记函数体可用 `await`，返回值包装为 `Future`。 |
| `await` | 等待 `Future`/`Stream` 事件（在异步函数内）。 |
| `await for` | 异步 for：遍历 `Stream`（如持续从 `ReceivePort` 收消息）。 |
| `Future<T>` | 表示稍后完成的单次异步结果。 |
| `Stream<T>` | 表示随时间产生的一系列事件。 |

### 6.2 面向对象与类型系统

| 语法 | 含义 |
|------|------|
| `class`、`extends` | 类与单继承。 |
| `implements` | 实现接口（可多实现）。 |
| `mixin ... on ...` | 受限 mixin：只能混入在指定超类型子类型上（如 `WidgetsBinding on SchedulerBinding`）。 |
| `super(...)` | 调用父类构造 / 父成员。 |
| `this.field` / 初始化列表 | 构造器简写：`MyData({required this.counter})`。 |
| `final` | 运行期一次性赋值；常用于不可变字段、Widget 配置字段。 |
| `const` | 编译期常量；`const` 构造的 Widget 有助于减少重建成本。 |
| `static` | 属于类本身：`Isolate.spawn` 常用 `static` 顶层入口。 |

### 6.3 空安全（Null Safety）

| 语法 | 含义 |
|------|------|
| `T` / `T?` | 非空类型 / 可空类型。 |
| `!` | 断言非空（失败会运行时异常）。 |
| `?.`、`??`、`??=` | 可空调用、合并空值、空则赋值。 |
| `required` | 命名参数必须传入（常与 NNBD 一起用于构造器）。 |

### 6.4 函数与集合

| 语法 | 含义 |
|------|------|
| `=>` | 单行函数体 / 箭头表达式。 |
| `_` | 匿名参数占位：`(_, __) {}`。 |
| `for-in` / collection `for` | `for (final x in xs)`；UI 里常用 `[for (final r in rs) Widget(...)]`。 |
| `List.generate` | 按下标生成固定长度列表。 |

### 6.5 并发与底层库

| 语法 | 含义 |
|------|------|
| `import 'dart:isolate';` | `Isolate`、`SendPort`、`ReceivePort`。 |
| `import 'dart:convert';` | `json.encode` / `json.decode`。 |
| `import 'dart:math';` | `Random`、`Rect`（在 `dart:ui`/`vector_math` 概念上常配合 Flutter 几何）。 |

### 6.6 Flutter 侧常见「看起来像 Dart 关键字」的概念

- `@override`：**注解**，标记重写，利于编译器检查。
- `BuildContext`：元素树上的句柄，用于查找 `Theme`、`MediaQuery`、`InheritedWidget` 等。
- `setState(() {})`：`State` 内触发当前 `Element` **标记脏**并在合适时机 `build`。

---

## 7. 面试问答速记（Q&A）

这部分用于沉淀「每次面试前/面试中临时问到的点」，主打**短、准、可复习**。

### 7.1 Cubit 是怎么实现数据驱动刷新的？

- **核心链路**：`emit(newState)` → 状态流（`Stream<State>`）发出新状态 → `BlocBuilder/BlocSelector` 订阅到变化 → 触发自身重建并用最新 `state` 再次 build。
- **关键点**：你不需要业务里手写 `setState`，但 `BlocBuilder` 内部一定有“等价的标脏/重建机制”来驱动刷新。

### 7.2 用 Cubit 就必须写 `setState` 吗？能全用 `StatelessWidget` 吗？

- **不需要你写**：Cubit/Bloc 的刷新通常由 `BlocBuilder` 等组件内部完成。
- **可以全用 `StatelessWidget`**：外层页面可以是 `StatelessWidget`，把需要刷新的局部用 `BlocBuilder/BlocSelector` 包起来；重建发生在这些订阅组件自身。

### 7.3 所有状态管理触发刷新，都会走 `setState` 这条线吗？

- 更严谨的说法：都会回到 Flutter 的本质——让某个 `Element` **标记为 dirty**，下一帧重新执行 `build`（以及按需 layout/paint）。
- `setState` 是最直接的入口；很多状态管理库并不暴露 `setState`，但内部会用**等价机制**触发“标脏→重建”。

### 7.4 BuildContext 是什么？

- **本质**：`BuildContext` 是 “当前 widget 在树上的位置句柄”，底层就是对应 `Element` 的接口。
- **常用能力**：
  - 向上查找并建立依赖：`dependOnInheritedWidgetOfExactType<T>()`（例如 `Theme.of(context)` 等）。
  - 只读取不建立依赖（各库 `read` 思路）：避免不必要的重建。
  - 导航/弹窗/拿 `ScaffoldMessenger` 等：依赖树上祖先组件能力。
- **常见坑**：跨 `await` 使用 context 前先判断 `context.mounted`；`initState` 里不要做会建立依赖的查找（更适合 `didChangeDependencies` / 首帧回调）。

### 7.5 `context.dependOnInheritedWidgetOfExactType<T>()` 会有性能问题吗？

- **查找成本通常不大**（沿祖先链向上找），真正的性能风险在于：建立依赖后，`InheritedWidget` 更新会**通知依赖者重建**。
- 常见踩坑：
  - 在很上层建立对“高频变化数据”的依赖 → 造成大范围重建。
  - `updateShouldNotify` 写得太宽（总返回 `true`） → 依赖者被无差别刷新。
- 优化思路：拆粒度、局部订阅（`Selector/BlocSelector`）、精确实现 `updateShouldNotify`、需要一次性读取就用不建立依赖的方式。

### 7.6 `for` 循环里写 `setState` 会执行多次重绘吗？

- **会多次调用 `setState`**，即多次“标记需要重建”；但**不一定每次都立刻重绘**。
- 同一事件循环/同一帧内连续多次 `setState` 通常会在下一帧**合并成一次 build**（仍有额外调度开销）。
- 如果循环中夹了 `await`/定时器/多次事件回调导致跨帧，就可能产生**多次 build/多次绘制**。
- 实践建议：循环里先累积数据，循环结束后只 `setState` 一次；需要逐步更新则用动画/定时器/流并控制频率。

### 7.7 Flutter 项目模块化怎么理解？结合 `xbit-app-mobile-v2` 怎么讲

- **一句话结论**：`xbit-app-mobile-v2` 属于**单工程（single app）**形态的模块化——通过**目录分层 + barrel exports + 集中式 DI/路由/多环境入口**来实现“可维护/可复用/可并行”的模块边界；业务域按 feature 概念存在，但没有拆成多 package 的 monorepo。
- **项目里的“模块化落点”**：
  - **核心能力集中**：`lib/core/` 聚合数据访问（GraphQL/REST）、存储、服务、用例与 BLoC，并通过 `lib/core/core.dart` 统一 `export` 作为“核心模块出口”。
  - **UI 组件体系**：`lib/ui/` 类 design system（atoms/molecules/organisms/templates/pages）+ `ui/v1/` 兼容/迁移层，并通过 `lib/ui/ui.dart` 做统一出口。
  - **路由集中注册**：`lib/routes/` 用 `go_router` 统一组装（如 `lib/routes/router.dart`），路径常量集中在 `lib/routes/pages.dart`，避免字符串散落。
  - **依赖注入集中组装**：`get_it` 在 `lib/dependencies.dart` / `setupDependencies()` 集中注册 service/client/bloc，模块对外靠接口或统一出口暴露。
  - **多环境模块化**：`env/` 下 `.unstable/.staging/.production`，入口文件 `lib/main.<env>.dart` + `flutter_dotenv`（`lib/config/app_env.dart`）实现环境隔离。
- **面试加分点（说现状也说风险）**：
  - 现状已经做到“工程组装点清晰”（路由/DI/env）与“UI 组件体系化”。
  - **改进空间**：存在 `core` 反向依赖 `ui/v1` 的耦合（例如 core utils 里 toast/导航），更理想是让 **core 只依赖抽象接口**，UI 提供实现并在 DI 注入，避免 `core↔ui` 双向依赖。
- **关键词**：`模块边界` `目录分层` `barrel export` `go_router` `get_it` `core/ui 解耦` `依赖方向` `env/flavor`

### 7.8 Flutter/Dart 的事件循环与帧调度机制是什么？（参考官方描述）

- **一句话结论**：Dart 的异步由 **事件循环**调度（microtask queue 优先于 event queue）；Flutter 在此之上用 **SchedulerBinding** 把 UI 更新组织成“帧”，并在每帧内按固定阶段驱动渲染流水线（layout/paint/composite/semantics），最后执行 post-frame 回调。
- **要点**：
  - **Dart microtask 优先**：`scheduleMicrotask` 注册的回调会在其它异步事件（如 Timer）之前执行；microtask 用多了可能饿死 event queue（官方警告）。  
    参考：`scheduleMicrotask`（Dart API）`https://api.dart.dev/stable/dart-async/scheduleMicrotask.html`
  - **帧回调分类（SchedulerBinding）**：官方将帧内回调分为 transient / persistent / post-frame，并解释它们与 `onBeginFrame/onDrawFrame` 的关系。  
    参考：`SchedulerBinding`（Flutter API）`https://docs.flutter.dev/flutter/scheduler/SchedulerBinding-mixin.html`
  - **addPostFrameCallback 的语义**：回调在一帧结束后、渲染流水线 flush 之后执行；不主动请求新帧；只执行一次且不可取消。  
    参考：`addPostFrameCallback`（Flutter API）`https://api.flutter.dev/flutter/scheduler/SchedulerBinding/addPostFrameCallback.html`
  - **渲染流水线阶段（drawFrame）**：官方列出每帧包含 layout、compositing bits、paint、compositing、semantics、finalization 等阶段。  
    参考：`RendererBinding.drawFrame`（Flutter API）`https://api.flutter.dev/flutter/rendering/RendererBinding/drawFrame.html`
- **常见追问**：
  - **为什么要用 `addPostFrameCallback`**：拿尺寸/做滚动/依赖布局结果时，保证本帧渲染完成后再执行。
  - **`Future`/`async` 等于多线程吗**：不等；它们仍在同一 isolate 的事件循环上，真正并行靠 Isolate。
- **更新时间**：2026-04
- **关键词**：`event loop` `microtask` `SchedulerBinding` `drawFrame` `addPostFrameCallback`

### 7.9 `RepaintBoundary` 原理是什么？面试怎么讲

- **一句话结论**：`RepaintBoundary` 会在渲染树中插入一个 **compositing boundary（合成边界）**，把子树提升为**独立的 layer**；当子树发生重绘时，重绘会尽量**局部化**在该 layer 内，避免整片 UI 跟着 repaint。
- **它解决的问题**：
  - Flutter 的绘制是从 `RenderObject` 生成 layer tree；当某个节点 `markNeedsPaint`，重绘可能向上影响更大范围。
  - `RepaintBoundary` 把一块 UI 的“绘制影响范围”切开，**减少 repaint 的扩散**（尤其是动画/频繁变化区域叠在大静态背景之上时）。
- **原理要点（抓住 3 句话）**：
  - `RepaintBoundary` 对应 `RenderRepaintBoundary`，会让该节点成为 **layer 的边界**（独立 `Layer`，通常是 `OffsetLayer` 一类），框架在 `paint` 阶段会以它为单位生成/复用 layer。
  - 子树发生 repaint 时，理想情况下只需要更新该边界内的 layer（以及必要的合成），**不用让父节点整块重新 paint**。
  - 它优化的是 **paint/compositing** 成本，不直接优化 **build/layout**（build 还是会跑，layout dirty 也一样会向上影响）。
- **什么时候用（高频面试场景）**：
  - **高频动画/倒计时/进度条** 覆盖在复杂静态背景上：把高频区域包进 `RepaintBoundary`。
  - **列表复杂 item** 中只有局部在动：给局部动的子树加边界，避免整行/整屏 repaint。
- **副作用与误区（加分点）**：
  - **不是越多越好**：layer 变多会增加合成开销与内存占用；某些场景还可能影响 GPU 合成。
  - **它不等于“减少 rebuild”**：要减少 rebuild 用 `const`、拆 widget、`Selector/BlocSelector` 等；`RepaintBoundary` 主要管 paint。
  - 是否收益要用工具验证：看 DevTools 的 repaint 统计/性能时间线（以数据为准）。
- **更新时间**：2026-04
- **关键词**：`RepaintBoundary` `RenderRepaintBoundary` `layer tree` `paint` `compositing boundary`

### 7.10 Flutter 怎么做性能优化？（结合项目《App性能分析》查漏补缺）

- **一句话结论**：Flutter 性能优化不要“凭感觉改”，核心是围绕 **FPS/Jank 的根因（CPU 过载 vs GPU 过载）** 做定位：先用 DevTools/Timeline 找到掉帧发生在哪一帧、是哪段耗时（build/layout/paint 或业务方法），再用“减少工作量/缩小影响范围/拆帧与并行”三板斧去改。
- **先讲指标与根因（面试开场）**：
  - **Jank 两大类原因**（项目文档结论）：
    - **CPU**：UI 绘制前的任务太重（同步计算、JSON/格式化、埋点、复杂初始化等）
    - **GPU**：页面过于复杂/层级深，渲染耗时或触发不必要的 repaint
- **定位步骤（你做过什么）**：
  - **看帧耗时**：在 DevTools Performance/Timeline 里找掉帧帧，确认是 build/layout/paint 哪块超预算。
  - **看页面初始化**：例如文档以某页面为例，`initState` 曾占用单帧约 36%（\(6.1/16.67ms\)），优化后关注第二帧是否仍丢帧，并继续找“方法耗时”。
- **CPU 侧优化（减少主线程工作量）**：
  - **延迟非必要工作**：埋点/日志等可延迟到首帧后几帧执行（`addPostFrameCallback`/延时），或放到后台任务队列顺序执行（避免首帧拥塞）。
  - **Isolate/compute**：CPU 密集型（如 JSON → Model、大量计算）放到 isolate/`compute`。
  - **格式化/解析选择**：金额格式化用 `intl/NumberFormat`（避免自写低效逻辑）。
- **GPU/渲染侧优化（减少 paint/compositing 压力）**：
  - **降低页面复杂度与嵌套**：减少重型 widget、避免无意义层级。
  - **必要时用 `RepaintBoundary`**：把高频变化区域与大静态背景隔离，减少 repaint 扩散（见 7.9）。
- **减少 build 次数 / 缩小刷新范围（项目文档重点）**：
  - **`build()` 保持 Clean**：build 可能被多次调用，不能有不可预期副作用。
  - **避免大范围 `setState`**：把需要刷新的小块抽成更小的 `StatefulWidget`，缩小重建范围。
  - **可变组件下沉到叶子节点**：让上层尽量稳定，只让最小子树变化。
  - **优先 `const`**：不变的 widget 用 `const`，减少重建成本。
- **动画与重建（隔离可变/不可变）**：
  - 动画场景要把“不会变的外壳”（如 `GestureDetector` 等）与“会变的内容”拆开。
  - 内容类型一致但数据变动时，合理使用 `ValueKey`；也可考虑 `AnimatedSwitcher`/`PageView`/`ListView.builder` + 定时器实现轮播，并缓存可复用子 widget。
- **函数 Widget vs StatelessWidget（项目文档结论）**：
  - 用函数返回 widget 容易**错过框架复用优化**，也不利于隔离可变/不可变区域。
  - 抽成 `StatelessWidget`（或更小 widget）提升复用、可读性，并可与状态管理配合更清晰。
- **图片内存专项（项目文档给了量化结果）**：
  - 使用 `cacheWidth/cacheHeight` 按目标显示尺寸解码缩放，能显著降低内存占用（文档示例：图片内存从约 36MB 降到约 6MB，降幅 ~80%；App 总内存也有下降）。
  - 关注磁盘缓存策略：只缓存原图仍需 decode；可评估缓存“目标尺寸图”的策略（按业务取舍）。
- **状态管理（UI = f(state)）**：
  - 页面要有清晰的数据流（单向数据流），共享状态上提；复杂页面刷新控制在最小范围内（Selector/拆 widget）。
- **其它稳定性/性能习惯（查漏补缺清单）**：
  - 生命周期里及时释放：Stream/Controller/监听在 `dispose` 里关闭。
  - `mounted` 判断：异步回调更新 UI 前先判断 `mounted`。
  - 大对象及时清理：页面关闭时可手动清空大 List/Map。
  - 资源压缩：安装包内图片资源压缩后再用。
  - 开启静态检查：启用 `flutter_lints` 规范化。
- **更新时间**：2026-04
- **关键词**：`DevTools` `FPS` `Jank` `build clean` `setState scope` `const` `RepaintBoundary` `cacheWidth` `cacheHeight` `Isolate` `compute`

---

## 8. 附录：同份笔试材料中的通用题目（非 Flutter 专向）

以下内容来自 `iOS笔试题.docx`，与 Flutter 无强绑定，但工程面试常一起出现，可作附录速览：

1. **组件封装**：工具类（文件、权限、网络签名、加解密、JSON、地图蓝牙等）与二次封装价值（可控、简洁、可复用）。
2. **陌生模块排错**：复现 → 断点/日志 → 从堆栈与类型/空值入手 → 注释二分 → 记录防回归。
3. **播放器 + 投屏排期**：先播放核心链路，再投屏搜索/连接/控制，再联调权限与跳转，最后测试与增值功能分期。
4. **IM 多端一致**：消息状态字段（撤回/删除/已读等）+ 服务端通知/透传；本地缓存用 **服务端 maxId vs 本地 maxId** 做增量对齐。
5. **图片预览组件**：下载缓存、转场动画（起止 rect）、可演进为通用文件预览。
6. **任务列表实时状态**：相对轮询，更倾向 **长连接按需推送**，降低无效请求压力并可按任务 id 局部刷新。

---

## 维护说明

- 若你后续继续补充截图或新题，建议按 **「概念题 / 手写题 / 工程题」** 三类往对应章节追加，Dart 关键词统一补到第 6 节做索引。

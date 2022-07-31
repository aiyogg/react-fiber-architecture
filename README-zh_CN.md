# React Fiber 架构

## 介绍

React Fiber 是对 React 核心算法的不断重新实现。这是 React 团队两年多的研究成果。

React Fiber 的目标是提高其在动画、布局和手势等领域的适用性。它的主要功能是**增量渲染**：能够将渲染工作分成块并将其分散到多个帧上。

其他关键功能包括在新更新到来时暂停、中止或重用工作的能力；能够为不同类型的更新分配优先级；和新的并发原语。

### 关于本文档

Fiber 引入了几个新概念，仅通过查看代码很难理解。这份文档一开始是我在 React 项目中跟踪 Fiber 的实现时所做的笔记集合。随着它的发展，我意识到它也可能是对其他人有用的资源。

我将尝试使用最简单的语言，并通过明确定义关键术语来避免行话。如果可能，我还将大量链接到外部资源。

请注意，我不是 React 团队的成员，也没有任何权威发言。**这不是官方文档**。我已经要求 React 团队的成员检查它的准确性。

这也是一项正在进行的工作。**Fiber 是一个正在进行的项目，在完成之前可能会经历重大的重构。**我也在尝试在这里记录它的设计。非常欢迎改进和建议。

我的目标是，在阅读完这份文档后，你将充分理解 Fiber，以便[跟踪其实现](https://github.com/facebook/react/commits/master/src/renderers/shared/fiber), 甚至最终能够为 React 做出贡献。

### 先决条件

我强烈建议您在继续之前熟悉以下资源：

- [React Components, Elements, and Instances](https://facebook.github.io/react/blog/2015/12/18/react-components-elements-and-instances.html) - “组件”通常是一个重载的术语。牢牢掌握这些术语至关重要。
- [Reconciliation](https://facebook.github.io/react/docs/reconciliation.html) - React 协调算法的高级描述。
- [React Basic Theoretical Concepts](https://github.com/reactjs/react-basic) - 没有实现负担的 React 概念模型的描述。其中一些在初读时可能没有意义。没关系，随着时间的推移会更有意义。
- [React Design Principles](https://facebook.github.io/react/contributing/design-principles.html) - 请特别注意调度部分。它很好地解释了*为什么*要有 React Fiber。

## 回顾

如果您还没有，请查看先决条件部分。

在我们深入研究新东西之前，让我们回顾一些概念。

### 什么是协调 (reconciliation)?

<dl>
  <dt>reconciliation</dt>
  <dd>React 使用的算法将一棵树与另一棵树进行比较，以确定需要更改哪些部分。</dd>

  <dt>update</dt>
  <dd>用于呈现 React 应用程序的数据的更改。通常是 `setState` 的结果。最终导致重新渲染。</dd>
</dl>

React API 的中心思想是将更新视为导致整个应用程序重新渲染。这允许开发人员进行声明式推理，而不必担心如何有效地将应用程序从任何特定状态转换到另一个状态（A 到 B、B 到 C、C 到 A 等等）。

实际上，在每次更改时重新渲染整个应用程序仅适用于最琐碎的应用程序；在现实世界的应用程序中，就性能而言，它的成本高得令人望而却步。 React 具有优化，可以创建整个应用重新渲染的外观，同时保持出色的性能。这些优化的大部分是称为**协调**的过程的一部分。

Reconciliation 是被普遍理解为“虚拟 DOM”的算法。一个高级描述是这样的：当你渲染一个 React 应用程序时，一个描述应用程序的节点树被生成并保存在内存中。然后将该树刷新到渲染环境——例如，在浏览器应用程序的情况下，它被转换为一组 DOM 操作。当应用程序更新时（通常通过 `setState`），会生成一个新树。新树与之前的树进行比较，以计算更新渲染应用程序所需的操作。

尽管 Fiber 是 reconciler 的全新重写，但 [React 文档中描述的](https://facebook.github.io/react/docs/reconciliation.html)高级算法将大体相同。关键点是：

- 假设不同的组件类型会生成截然不同的树。 React 不会尝试区分它们，而是完全替换旧树。
- 列表的区分是使用 key 执行的。keys 应该是“稳定的、可预测的和唯一的”。

### Reconciliation VS rendering

DOM 只是 React 可以渲染的渲染环境之一，其他主要目标是通过 React Native 实现的原生 iOS 和 Android 视图。 （这就是为什么“虚拟 DOM”有点用词不当。）

它可以支持这么多目标的原因是因为 React 的设计使得协调（reconciliation）和渲染（rendering）是独立的阶段。协调器负责计算树的哪些部分发生了变化；渲染器然后使用该信息来实际更新渲染的应用程序。

这种分离意味着 React DOM 和 React Native 可以使用自己的渲染器，同时共享由 React 核心提供的同一个协调器。

Fiber 重新实现了协调器。它主要不关心渲染，尽管渲染器需要更改以支持（并利用）新架构。

### Scheduling

<dl>
  <dt>scheduling</dt>
  <dd>确定何时应执行工作的过程。</dd>

  <dt>work</dt>
  <dd>必须执行的任何计算。工作通常是更新的结果（例如 `setState`）。
</dl>

React 的 [设计原则](https://facebook.github.io/react/contributing/design-principles.html#scheduling) 文档在这个主题上非常好，我将在这里引用它：

> 在其当前实现中，React 递归地遍历树并在一个时刻调用整个更新树的渲染函数。但是在未来它可能会开始延迟一些更新以避免丢帧。
>
> 这是 React 设计中的一个常见主题。一些流行的库实现了“推”方法，在新数据可用时执行计算。然而，React 坚持“拉”的方法，在这种方法中，计算可以被推迟到必要的时候。
>
> React 不是一个通用的数据处理库。它是一个用于构建用户界面的库。我们认为它在应用程序中具有独特的定位，可以知道哪些计算现在是相关的，哪些不相关。
>
> 如果有东西不在屏幕上，我们可以延迟任何与之相关的逻辑。如果数据到达的速度快于帧速率，我们可以合并和批量更新。我们可以优先考虑来自用户交互的工作（例如由按钮单击引起的动画）而不是不太重要的后台工作（例如渲染刚从网络加载的新内容），以避免丢帧。

关键点是：

- 在 UI 中，不必立即应用每个更新；事实上，这样做可能会造成浪费，导致丢帧并降低用户体验。
- 不同类型的更新有不同的优先级——动画更新需要比数据存储更新更快地完成。
- 基于推的方法需要应用程序(你，程序员)来决定如何安排工作。基于拉的方法允许框架(React)变得聪明，并为您做出这些决策。

React 目前没有在很大程度上利用调度。更新会导致立即重新渲染整个子树。彻底改造 React 的核心算法以利用调度是 Fiber 背后的驱动理念。

---

现在我们准备好深入研究 Fiber 的实现。下一节比我们目前讨论的更具技术性。在继续之前，请确保您对以前的材料感到满意。

## 什么是 Fiber？

我们即将讨论 React Fiber 架构的核心。与应用程序开发人员通常认为的相比，Fiber 是一个更底层的抽象。如果您发现自己在尝试理解它时感到沮丧，请不要灰心。继续尝试，它最终会变得有意义。 （当你最终得到它时，请建议如何改进这部分。）

黑喂狗！

---

我们已经确定 Fiber 的主要目标是使 React 能够利用调度。具体来说，我们需要能够

- 暂停工作，稍后再回来。
- 为不同类型的工作分配优先级。
- 重用之前完成的工作。
- 如果不再需要，则中止工作。

为了做到这一点，我们首先需要一种将工作分解为单元的方法。从某种意义上说，这就是纤维。一根纤维代表一个**工作单元**。

更进一步，让我们回到 [React 组件作为数据函数](https://github.com/reactjs/react-basic#transformation)的概念，通常表示为

```
v = f(d)
```

因此，渲染一个 React 应用程序类似于调用一个函数，该函数的主体包含对其他函数的调用，依此类推。这个类比在考虑 fibers 时很有用。

计算机通常跟踪程序执行的方式是使用[调用堆栈](https://en.wikipedia.org/wiki/Call_stack)。当一个函数被执行时，一个新的**栈帧**被添加到栈中。该堆栈帧表示该函数执行的工作。

在处理 UI 时，问题是如果一次执行过多的工作，可能会导致动画掉帧并且看起来不连贯。更重要的是，如果这些工作被更新的更新所取代，其中一些工作可能是不必要的。这是 UI 组件和函数之间的比较失败的地方，因为组件比一般的函数具有更具体的关注点。

较新的浏览器（和 React Native）实现了有助于解决这个确切问题的 API：`requestIdleCallback` 安排在空闲期间调用低优先级函数，而 `requestAnimationFrame` 安排在下一个动画帧上调用高优先级函数。问题是，为了使用这些 API，您需要一种将渲染工作分解为增量单元的方法。如果你只依赖调用堆栈，它会一直工作直到堆栈为空。

如果我们可以自定义调用堆栈的行为以优化渲染 UI，那不是很好吗？如果我们可以随意中断调用堆栈并手动操作堆栈帧，那不是很好吗？

这就是 React Fiber 的目的。 Fiber 是堆栈的重新实现，专门用于 React 组件。您可以将单个 Fiber 视为**虚拟堆栈帧**。

重新实现堆栈的优点是您可以[将堆栈帧保存在内存中](https://www.facebook.com/groups/2003630259862046/permalink/2054053404819731/)，并根据需要（以及*何时*）执行它们。这对于实现我们的调度目标至关重要。

除了调度之外，手动处理堆栈帧释放了并发和错误边界等功能的潜力。我们将在以后的章节中介绍这些主题。

在下一节中，我们将更多地了解 fiber 的结构。

### fiber 的结构

*注意：随着我们对实现细节的了解更加具体，某些事情可能发生变化的可能性也会增加。如果您发现任何错误或过时的信息，请提交 PR。*

具体而言，fiber 是一个 JavaScript 对象，其中包含有关组件、其输入和输出的信息。

一个 fiber 对应于一个堆栈帧，但它也对应于一个组件的实例。

下面是属于 fiber 的一些重要字段 （此列表并不详尽。）

#### `type` 和 `key`

fiber 的 type 和 key 与 React 元素的作用相同。 （实际上，当从元素创建纤维时，这两个字段会直接复制过来。）

fiber 的 type 描述了它对应的组件。对于复合组件，类型是函数或类组件本身。对于宿主组件（`div`、`span` 等），类型是字符串。

从概念上讲，type 是堆栈帧正在跟踪其执行的函数（如 `v = f(d)`）。

与 type 一起，在协调期间使用 key 来确定 fiber 是否可以重复使用。

#### `child` 和 `sibling`

这些字段指向其他 fiber，描述 fiber 的递归树结构。

子 Fiber 对应于组件的 `render` 方法返回的值。所以在下面的例子中

```js
function Parent() {
  return <Child />
}
```

`Parent` 的子 Fiber 对应于 `Child`。

sibling 字段解释了渲染返回多个 child 的情况（Fiber 中的一个新功能！）：

```js
function Parent() {
  return [<Child1 />, <Child2 />]
}
```

子 fiber 形成一个单链表，其头部是第一个 child。所以在这个例子中，`Parent` 的孩子是 `Child1`，而 `Child1` 的兄弟是 `Child2`。

回到我们的函数类比，您可以将子 fiber 视为[尾部调用函数](https://en.wikipedia.org/wiki/Tail_call)。 

#### `return`

返回 fiber 是程序在处理完当前 fiber 后应该返回的 fiber。它在概念上与堆栈帧的返回地址相同。它也可以被认为是父级 fiber。

如果一个 fiber 有多个子 fiber，则每个子fiber的返回 fiber 都是父 fiber。所以在我们上一节的例子中，`Child1` 和 `Child2` 的返回 Fiber 是 `Parent`。

#### `pendingProps` 和 `memoizedProps`

从概念上讲，props 是函数的参数。 Fiber 的 `pendingProps` 在执行开始时设置，`memoizedProps` 在执行结束时设置。

当传入的 `pendingProps` 等于 memoizedProps 时，它表示可以重用 Fiber 之前的输出，防止不必要的工作。

#### `pendingWorkPriority`

一个数字，表示由 fiber 表示的工作的优先级。 [ReactPriorityLevel](https://github.com/facebook/react/blob/master/src/renderers/shared/fiber/ReactPriorityLevel.js) 模块列出了不同的优先级以及它们代表的内容。

除了 `NoWork` 为 0，数字越大表示优先级越低。例如，您可以使用以下函数来检查 fiber 的优先级是否至少与给定级别一样高：
```js
function matchesPriority(fiber, priority) {
  return fiber.pendingWorkPriority !== 0 &&
         fiber.pendingWorkPriority <= priority
}
```

*此功能仅用于说明；它实际上并不是 React Fiber 代码库的一部分。*

调度程序使用优先级字段来搜索下一个要执行的工作单元。该算法将在以后的章节中讨论。

#### `alternate`

<dl>
  <dt>flush</dt>
  <dd>刷新 fiber 就是将其输出渲染到屏幕上。</dd>

  <dt>work-in-progress</dt>
  <dd>尚未完成的 fiber；从概念上讲，一个尚未返回的堆栈帧。</dd>
</dl>

在任何时候，一个组件实例最多有两个与之对应的纤程：当前的、已刷新的纤程和正在进行的纤程。

当前 Fiber 的 Alternate 为 work-in-progress，work-in-progress 的 Alternate 为当前 Fiber。

使用名为 `cloneFiber` 的函数懒惰地创建 Fiber 的备用。与其总是创建一个新对象，`cloneFiber` 将尝试重用 Fiber 的备用（如果存在），从而最大限度地减少分配。

您应该将 alternate 字段视为实现细节，但它在代码库中经常出现，因此在这里讨论它很有价值。

#### `output`

<dl>
  <dt>host component</dt>
  <dd>React 应用程序的叶节点。它们特定于渲染环境（例如，在浏览器应用程序中，它们是 `div`、`span` 等）。在 JSX 中，它们使用小写的标记名称表示。</dd>
</dl>

从概念上讲，fiber 的输出是函数的返回值。

每个 fiber 最终都有输出，但输出仅由宿主组件（host component）在叶节点处创建。然后将输出传输到树上。

输出是最终提供给渲染器的内容，以便它可以将更改刷新到渲染环境。渲染器负责定义如何创建和更新输出。

## 未来章节

目前就这些了，但这份文件还远远不够完整。以后的章节将描述在更新的整个生命周期中使用的算法。主题包括:

- 调度器（scheduler）如何找到下一个要执行的工作单元。
- 如何通过 fiber 树跟踪和传播优先级。
- 调度器（scheduler）如何知道何时暂停和恢复工作。
- 如何刷新工作并将其标记为完成。
- 副作用（例如生命周期方法）如何工作。
- 什么是协同程序，以及如何使用它来实现诸如 context 和 layout 之类的特性。

## 相关视频
- [What's Next for React (ReactNext 2016)](https://youtu.be/aV1271hd9ew)

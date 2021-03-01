---
title: React函数组件和类组件的差异
date: 2021-02-27 20:10:22
tags:
---

最近使用函数组件的场景越来越多了 ಠ_ಠ，一直也在不断的摸索函数组件和类组件的区别，尤其是最近做的一个项目，简直完全就是函数组件的堆砌了，那么和 React 类组件相比，React 函数式组件究竟有何不同，难道真的就是简洁而已嘛？

在过去一段时间里，典型的回答是类组件提供了更多的特性（比如 state）。不过当有了 [Hooks](https://reactjs.org/docs/hooks-intro.html) 后，答案就不再是这样了。

<!-- more -->

或许你曾听过它们中的某一个在性能上的表现优于另一个。那是哪一个？很多此类的判断标准都存在这样那样的缺陷，所以我们不能片面的看待这个问题。性能主要取决于代码的作用，而不是选择函数式还是类组件。在持续的观察中，尽管优化策略各有略微不同，但性能差异可以忽略不计。

理论上，其实并不推荐重构你现有的组件。Hooks 还很年轻（如同 2014 年的 React），并且有些“最佳实践”还没有找到它们的切入方式。

那么现在是个什么情况？React 的函数式组件和类组件之间是否有任何根本上的区别？当然有 —— 在心智模型上。

> 自 2015 年我们推出函数式组件以来，它一直存在，但是经常被忽略：
> 函数式组件捕获了渲染所用的值。（Function components capture the rendered values.）

让我们来看看这意味着什么。

### 🐮 一

**首先我们来看一下这个组件：**

```js
function ProfilePage(props) {
  const showMessage = () => {
    alert("Followed " + props.user);
  };

  const handleClick = () => {
    setTimeout(showMessage, 3000);
  };

  return <button onClick={handleClick}>Follow</button>;
}
```

它渲染了一个利用 `setTimeout` 来模拟网络请求，然后显示一个确认警告的按钮。例如，如果 `props.user` 是 Dan，它会在三秒后显示 Followed Dan。非常简单。

> 在上面的示例中我是否用了箭头函数或者函数声明并不重要。`function handleClick()`也将完全以同样方式有效。

### 🐮 二

如果是类组件我们怎么写？一个简单的重构可能就象这样：

```js
class ProfilePage extends React.Component {
  showMessage = () => {
    alert("Followed " + this.props.user);
  };

  handleClick = () => {
    setTimeout(this.showMessage, 3000);
  };

  render() {
    return <button onClick={this.handleClick}>Follow</button>;
  }
}
```

通常情况下，我们认为这两个代码片段是等效的。大家经常在这两种模式中自由的重构代码，但是很少注意到它们的含义：
![img](1.gif)
但是，这两个代码片段还是有略微的不同，不然我这篇文章就没必要哒哒哒讲这么多。仔细的看看它们。现在看出他们的不同了吗？反正我是花了好一会儿才看明白这一点。

接下来的文章是『剧透』，如果你想自己搞明白，你可以试着自己写一份 demo。 本文的剩余部分解释了这里面的差异以及阐述了为什么这很重要。

接下来的需要着重强调一点，下文所描述的差异与 React Hooks 完全无关。毕竟上面的例子中都还没有使用到 Hooks...

它全部是关于 React 中函数式组件与类组件的区别的。如果你打算在你的 React 应用中更频繁地使用函数式组件，就可能需要理解它。

接下来将通过 React 应用程序中的一个常见错误来说明其中的不同。

按照我这个例子，你将看到一个当前账号选择框以及两个上面 ProfilePage 的实现 —— 每个都渲染了一个 Follow 按钮。

### 🐮 三

尝试按照以下顺序来分别使用这两个按钮：

1. 点击 其中某一个 Follow 按钮。
2. 在 3 秒内 切换 选中的账号。
3. 查看 弹出的文本。
   你将看到一个奇特的区别:

- 当使用 函数式组件 实现的 ProfilePage, 当前账号是 Dan 时点击 Follow 按钮，然后立马切换当前账号到 Sophie，弹出的文本将依旧是 'Followed Dan'。
- 当使用 类组件 实现的 ProfilePage, 弹出的文本将是 'Followed Sophie'：

也就是下面这样：
![img](2.gif)

在这个例子中，第一个行为是正确的。如果我关注一个人，然后导航到了另一个人的账号，我的组件不应该混淆我关注了谁。 在这里，类组件的实现很明显是错误的。

所以为什么我们的例子中类组件会有这样的表现？

### 🐮 四

让我们来仔细看看我们类组件中的 `showMessage` 方法：

```js
class ProfilePage extends React.Component {
  showMessage = () => {
    alert("Followed " + this.props.user);
  };
}
```

这个类方法从 `this.props.user` 中读取数据。在 React 中 Props 是不可变(immutable)的，所以他们永远不会改变。然而，this 是，而且永远是，可变(mutable)的。

事实上，这就是类组件 this 存在的意义。React 本身会随着时间的推移而改变，以便你可以在渲染方法以及生命周期方法中得到最新的实例。

所以如果在请求已经发出的情况下我们的组件进行了重新渲染，`this.props` 将会改变。`showMessage` 方法从一个“过于新”的 `props` 中得到了 `user`。

这暴露了一个关于用户界面性质的一个有趣观察。如果我们说 UI 在概念上是当前应用状态的一个函数，那么事件处理程序则是渲染结果的一部分 —— 就像视觉输出一样。我们的事件处理程序“属于”一个拥有特定 `props` 和 `state` 的特定渲染。

然而，调用一个回调函数读取 `this.props` 的 timeout 会打断这种关联。我们的 `showMessage` 回调并没有与任何一个特定的渲染“绑定”在一起，所以它“失去”了正确的 `props`。从 this 中读取数据的这种行为，切断了这种联系。

### 🐮 五

让我们假设函数式组件不存在。我们将如何解决这个问题？

我们想要以某种方式“修复”拥有正确 `props` 的渲染与读取这些 `props` 的 `showMessage` 回调之间的联系。在某个地方 `props` 被弄丢了。

一种方法是在调用事件之前读取 `this.props`，然后将他们显式地传递到 timeout 回调函数中去：

```js
class ProfilePage extends React.Component {
  showMessage = (user) => {
    alert("Followed " + user);
  };

  handleClick = () => {
    const { user } = this.props;
    setTimeout(() => this.showMessage(user), 3000);
  };

  render() {
    return <button onClick={this.handleClick}>Follow</button>;
  }
}
```

这种方法 会起作用。然而，这种方法使得代码明显变得更加冗长，并且随着时间推移容易出错。如果我们需要的不止是一个 `props` 怎么办？如果我们还需要访问 `state` 怎么办？如果 `showMessage` 调用了另一个方法，然后那个方法中读取了 `this.props.something` 或者 `this.state.something`，我们又将遇到同样的问题。然后我们不得不将 `this.props` 和 `this.state` 以函数参数的形式在被 `showMessage` 调用的每个方法中一路传递下去。

这样的做法破坏了类提供的工程学。同时这也很难让人去记住传递的变量或者强制执行，这也是为什么人们总是在解决 bugs。

同样的，在 `handleClick` 中内联地写 alert 代码也无法解决问题。我们希望以允许将其拆分为多个方法的方式来构造组织代码，但同时也能读取与某次组件调用形成的渲染结果对应的 `props` 和 `state`。这个问题并不是 React 所独有的 —— 你可以在任何一个将数据放入类似 this 这样的可变对象中的 UI 库中重现它。

### 🐮 六

或许，我们可以在构造函数中绑定方法？

```js
class ProfilePage extends React.Component {
  constructor(props) {
    super(props);
    this.showMessage = this.showMessage.bind(this);
    this.handleClick = this.handleClick.bind(this);
  }

  showMessage() {
    alert("Followed " + this.props.user);
  }

  handleClick() {
    setTimeout(this.showMessage, 3000);
  }

  render() {
    return <button onClick={this.handleClick}>Follow</button>;
  }
}
```

不，这没有解决任何问题。记住，我们面对的问题是我们从 `this.props` 中读取数据太迟了——读取时已经不是我们所需要使用的上下文了！然而，如果我们能利用 JavaScript 闭包的话问题将迎刃而解。

通常来说我们会避免使用闭包，因为它会让我们难以想象一个可能会随着时间推移而变化的变量。但是在 React 中，`props` 和 `state` 是不可变得！（或者说，在我们的强烈推荐中是不可变得。）这就消除了闭包的一个主要缺陷。

这就意味着如果你在一次特定的渲染中捕获那一次渲染所用的 `props` 或者 `state`，你会发现他们总是会保持一致，就如同你的预期那样：

```js
class ProfilePage extends React.Component {
  render() {
    // Capture the props!
    const props = this.props;

    // Note: we are *inside render*.
    // These aren't class methods.
    const showMessage = () => {
      alert("Followed " + props.user);
    };

    const handleClick = () => {
      setTimeout(showMessage, 3000);
    };

    return <button onClick={handleClick}>Follow</button>;
  }
}
```

你在渲染的时候就已经“捕获”了~~皮卡丘~~ **props**：
![img](3.gif)
这样，在它内部的任何代码（包括 `showMessage`）都保证可以得到这一次特定渲染所使用的 `props`。React 再也不会“动我们的奶酪”。

然后我们可以在里面添加任意多的辅助函数，它们都会使用被捕获的 `props` 和 `state`。闭包万岁！

### 🐮 七

上面的例子是正确的，但是看起来很奇怪。如果你在 render 方法中定义各种函数，而不是使用 class 的方法，那么使用类的意义在哪里？

事实上，我们可以通过删除类的“包裹”来简化代码：

```js
function ProfilePage(props) {
  const showMessage = () => {
    alert("Followed " + props.user);
  };

  const handleClick = () => {
    setTimeout(showMessage, 3000);
  };

  return <button onClick={handleClick}>Follow</button>;
}
```

就像上面这样，`props` 仍旧被捕获了 —— React 将它们作为参数传递。不同于 this，`props` 对象本身永远不会被 React 改变。

### 🐮 八

如果你在函数定义中解构 `props`，那将更加明显：

```js
function ProfilePage({ user }) {
  const showMessage = () => {
    alert("Followed " + user);
  };

  const handleClick = () => {
    setTimeout(showMessage, 3000);
  };

  return <button onClick={handleClick}>Follow</button>;
}
```

当父组件使用不同的 `props` 来渲染 `ProfilePage` 时，React 会再次调用 `ProfilePage` 函数。但是我们点击的事件处理函数，“属于”具有自己的 `user` 值的上一次渲染，并且 `showMessage` 回调函数也能读取到这个值。它们都保持完好无损。

这就是为什么，在上面那个 demo 的函数式版本中，点击关注 Sophie 的账号，然后改变选择为 Sunil 仍旧会弹出'Followed Sophie'：
![img](4.gif)
这个行为展现是正确的。

现在我们明白了 React 中函数式组件和类组件之间的巨大差别：

函数式组件捕获了渲染所使用的值。

### 🐮 九

使用 Hooks，同样的原则也适用于 state。 看这个例子：

```js

  const handleMessageChange = (e) => {
    setMessage(e.target.value);
  };

  return (
    <>
      <input value={message} onChange={handleMessageChange} />
      <button onClick={handleSendClick}>Send</button>
    </>
  );
}
```

尽管这不是一个非常好的消息应用的 UI，但它说明了同样的观点：如果我发送一条特定的消息，组件不应该对实际发送的是哪条消息感到困惑。这个函数组件的 `message` 变量捕获了“属于”返回了被浏览器调用的单击处理函数的那一次渲染。所以当我点击“发送”时 `message` 被设置为那一刻在 input 中输入的内容。

因此我们知道，在默认情况下 React 中的函数会捕获 `props` 和 `state`。但是如果我们想要读取并不属于这一次特定渲染的，最新的 props 和 state 呢？如果我们想要“从未来读取他们”呢？

在类中，你通过读取 `this.props` 或者 `this.state` 来实现，因为 this 本身时可变的。React 改变了它。在函数式组件中，你也可以拥有一个在所有的组件渲染帧中共享的可变变量。它被成为“ref”：

```js
function MyComponent() {
  const ref = useRef(null);
  // You can read or write `ref.current`.
  // ...
}
```

但是，你必须自己管理它。

一个 ref 与一个实例字段扮演同样的角色。这是进入可变的命令式的世界的后门。你可能熟悉’DOM refs’，但是 ref 在概念上更为广泛通用。它只是一个你可以放东西进去的盒子。

甚至在视觉上，`this.something` 就像是 `something.current` 的一个镜像。他们代表了同样的概念。

默认情况下，React 不会在函数式组件中为最新的 `props` 和 `state` 创造 refs。在很多情况下，你并不需要它们，并且分配它们将是一种浪费。但是，如果你愿意，你可以这样手动地来追踪这些值：

```js
function MessageThread() {
  const [message, setMessage] = useState('');
  const latestMessage = useRef('');

  const showMessage = () => {
    alert('You said: ' + latestMessage.current);
  };

  const handleSendClick = () => {
    setTimeout(showMessage, 3000);
  };

  const handleMessageChange = (e) => {
    setMessage(e.target.value);
    latestMessage.current = e.target.value;
  };
```

如果我们在 `showMessage` 中读取 `message`，我们将得到在我们按下发送按钮那一刻的信息。但是当我们读取 `latestMessage.current`，我们将得到最新的值 —— 即使我们在按下发送按钮后继续输入。

你可以自行查看这 俩 demo 来比较它们之间的不同。ref 是一种“选择退出”渲染一致性的方法，在某些情况下会十分方便。

通常情况下，你应该避免在渲染期间读取或者设置 refs，因为它们是可变得。我们希望保持渲染的可预测性。然而，如果我们想要特定 `props` 或者 `state` 的最新值，那么手动更新 ref 会有些烦人。我们可以通过使用一个 effect 来自动化实现它：

```js
function MessageThread() {
  const [message, setMessage] = useState('');

  // 保持追踪最新的值。
  const latestMessage = useRef('');
  useEffect(() => {
    latestMessage.current = message;
  });

  const showMessage = () => {
    alert('You said: ' + latestMessage.current);
  };
```

我们在一个 effect 内部执行赋值操作以便让 ref 的值只会在 DOM 被更新后才会改变。这确保了我们的变量突变不会破坏依赖于可中断渲染的时间切片和 Suspense 等特性。

通常来说使用这样的 ref 并不是非常地必要。捕获 `props` 和 `state` 通常是更好的默认值。然而，在处理类似于 intervals 和订阅这样的命令式 API 时，ref 会十分便利。记住，你可以像这样跟踪任何值 —— 一个 `prop`，一个 `state` 变量，整个 `props` 对象，或者甚至一个函数。

这种模式对于优化来说也很方便 —— 例如当 `useCallback` 本身经常改变时。然而，使用一个 reducer 通常是一个更好的解决方式。（未来博客文章的主题！）

### 🐮 结尾

在这篇文章中，我们已经看过了类组件中常见的破碎模式，以及闭包如何帮助我们修复它。然而，你可能注意到，当你尝试通过指定一个依赖数组来优化 Hooks 时，你可能会遇到带有过时闭包的问题。这是否意味着闭包是一个问题？我不这么认为。

正如我们上面看到的，闭包实际上帮我们解决了很难注意到的细微问题。同样，它们也使得在并发模式下能更轻松地编写能够正确运行的代码。这是可行的，因为组件内部的逻辑在渲染它时捕获并包含了正确的 `props` 和 `state`。

目前为止我看到的所有情况中，所谓的“陈旧的闭包”问题的出现多是由于错误的假设了“函数不会改变”或者“props 永远是一样的”。事实并非如此，而我希望这篇文章有助于澄清这一点。

函数捕获了他们的 `props` 和 `state` —— 因此它们的标识也同样重要。这不是一个 bug，而是一个函数式组件的特性。例如，对于 `useEffect` 或者 `useCallback` 来说，函数不应该被排除在“依赖数组”之外。（正确的解决方案通常是使用上面说过的 `useReducer` 或者 `useRef` —— 我们将很快会在文档中说明如何在它们两者中进行选择。）

当我们用函数来编写大部分的 React 代码时，我们需要调整关于优化代码和什么变量会随着时间改变的认知与直觉。

正如 Fredrik 所说：

> 到目前为止，我发现的有关于 `hooks` 的最好的心里规则是“写代码时要认为任何值都可以随时更改”。

函数也不例外。这需要一段时间才能成为 React 学习资料中的常识。它需要一些从类的思维方式中进行一些调整。希望这篇文章能够帮助你以新的眼光来看待它，用更多的视角去了解类函数和函数组件的区别。

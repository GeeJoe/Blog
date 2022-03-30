# StatefulWidget 最佳实践

## StatefulWidget 还是 StatelessWidget

我们都知道在 Flutter 的世界中 *everything is widget*, 而 widget 又分为 StatelessWidget 和 StatefulWidget。 看过很多文章包括官网我们大部分人都知道 StatelessWidget 是无状态的，StatefulWidget 是有状态的，当我们需要动态更新状态的时候使用 StatefulWidget 否则使用 StatelessWidget。

但是面对这两个 widget 的抉择的时候，似乎 StatelessWidget 能做的事情 StatefulWidget 都能实现，甚至像 Android Studio 等 IDE 都有直接从 StatelessWidget 转为 StatefulWidget 的快捷方式，那 StatelessWidget 存在的意义是什么呢？Flutter 团队为什么要设计 StatelessWidget 呢?

两者 build 的时机

- StatelessWidget：when it is first inserted in the tree、it's parent changes its configurarion、an InheritedWidget it depends on changes.
- StatefulWidget：以上基础加上 setState

即使 StatefuleWidget 只 build 一次，从减少复杂性考虑，也应该是 StatelessWidget

有没有 repaintBoundry 的影响

## setState 正确写法 

应该把修改状态的代码段放入 `setState` 的参数中

Bad code

```dart 
_count++;
setState(() {});
// ...
_name = "hello world";
_length--; 
setState(() {});
```

Best Practice

```dart
setState(() {
  _count++;
});
// ...
setState(() {
  _name = "hello world";
  _length--; 
});
```

`StatefulWidget` 每一次 `setState` 都应该是有理由的（Reasonable），因此把修改状态/数据的代码段放入 `setState`中，明确表示此次 `setState` 是为了更新这些数据，这种写法有更好的**可读性**和**可维护性**。

这里举例的代码比较简单，可能还感受不到两者的区别；实际项目中我们的代码往往更加复杂，可能一个 widget 中有好几处 `setState`。试想一下如果都用第一种方式，在后续迭代中，因为产品需求变动，我们可能不需要修改某些数据。比如不需要修改 `_name` 和 `_length` 的值了，我们删了这两行代码，但我们不确定是否要删除 `setState(() {})`， 因为很有可能从上下文我们根本无法判断出这个 setState 到底是为了更新谁，因为代码上他和 ` _name`、`_length` 都没有建立联系，这就会导致项目中遗留越来越多无用代码，甚至导致性能问题

## 性能陷阱



There are two primary categories of [StatefulWidget](https://api.flutter.dev/flutter/widgets/StatefulWidget-class.html)s.

The first is one which allocates resources in [State.initState](https://api.flutter.dev/flutter/widgets/State/initState.html) and disposes of them in [State.dispose](https://api.flutter.dev/flutter/widgets/State/dispose.html), but which does not depend on [InheritedWidget](https://api.flutter.dev/flutter/widgets/InheritedWidget-class.html)s or call [State.setState](https://api.flutter.dev/flutter/widgets/State/setState.html). Such widgets are commonly used at the root of an application or page, and communicate with subwidgets via [ChangeNotifier](https://api.flutter.dev/flutter/foundation/ChangeNotifier-class.html)s, [Stream](https://api.flutter.dev/flutter/dart-async/Stream-class.html)s, or other such objects. Stateful widgets following such a pattern are relatively cheap (in terms of CPU and GPU cycles), because they are built once then never update. They can, therefore, have somewhat complicated and deep build methods.

The second category is widgets that use [State.setState](https://api.flutter.dev/flutter/widgets/State/setState.html) or depend on [InheritedWidget](https://api.flutter.dev/flutter/widgets/InheritedWidget-class.html)s. These will typically rebuild many times during the application's lifetime, and it is therefore important to minimize the impact of rebuilding such a widget. (They may also use [State.initState](https://api.flutter.dev/flutter/widgets/State/initState.html) or [State.didChangeDependencies](https://api.flutter.dev/flutter/widgets/State/didChangeDependencies.html) and allocate resources, but the important part is that they rebuild.)

There are several techniques one can use to minimize the impact of rebuilding a stateful widget:

- Push the state to the leaves. For example, if your page has a ticking clock, rather than putting the state at the top of the page and rebuilding the entire page each time the clock ticks, create a dedicated clock widget that only updates itself.
- Minimize the number of nodes transitively created by the build method and any widgets it creates. Ideally, a stateful widget would only create a single widget, and that widget would be a [RenderObjectWidget](https://api.flutter.dev/flutter/widgets/RenderObjectWidget-class.html). (Obviously this isn't always practical, but the closer a widget gets to this ideal, the more efficient it will be.)
- If a subtree does not change, cache the widget that represents that subtree and re-use it each time it can be used. It is massively more efficient for a widget to be re-used than for a new (but identically-configured) widget to be created. Factoring out the stateful part into a widget that takes a child argument is a common way of doing this.
- Use `const` widgets where possible. (This is equivalent to caching a widget and re-using it.)
- Avoid changing the depth of any created subtrees or changing the type of any widgets in the subtree. For example, rather than returning either the child or the child wrapped in an [IgnorePointer](https://api.flutter.dev/flutter/widgets/IgnorePointer-class.html), always wrap the child widget in an [IgnorePointer](https://api.flutter.dev/flutter/widgets/IgnorePointer-class.html) and control the [IgnorePointer.ignoring](https://api.flutter.dev/flutter/widgets/IgnorePointer/ignoring.html) property. This is because changing the depth of the subtree requires rebuilding, laying out, and painting the entire subtree, whereas just changing the property will require the least possible change to the render tree (in the case of [IgnorePointer](https://api.flutter.dev/flutter/widgets/IgnorePointer-class.html), for example, no layout or repaint is necessary at all).
- If the depth must be changed for some reason, consider wrapping the common parts of the subtrees in widgets that have a [GlobalKey](https://api.flutter.dev/flutter/widgets/GlobalKey-class.html) that remains consistent for the life of the stateful widget. (The [KeyedSubtree](https://api.flutter.dev/flutter/widgets/KeyedSubtree-class.html) widget may be useful for this purpose if no other widget can conveniently be assigned the key.)



Provider 最佳实践

## Function call 还是 widget class

不能直接下结论说方法比 widget 性能差，要看我们怎么用，不过相对而言，class 比 function 有更好封装，有助于帮我们写出性能更好的代码

举例

```dart
class _MyAppState extends State<MyApp> {
  int _count = 1;

  @override
  Widget build(BuildContext context) {
    return MaterialApp(
      home: Scaffold(
        body: Column(
          children: [
            NotRebuildText(
              value: _count,
            ),
            _buildText(_count),
          ],
        ),
        floatingActionButton: FloatingActionButton(
          onPressed: () {
            setState(() {});
          },
        ),
      ),
    );
  }
}

Widget _buildText(int value) {
  print("build Text in function call");
  return Text(value.toString());
}

class NotRebuildText extends StatelessWidget {
  const NotRebuildText({Key? key, required this.value}) : super(key: key);

  final int value;

  @override
  Widget build(BuildContext context) {
    print("build Text in class call");
    return Text(value.toString());
  }

  @override
  bool operator ==(Object other) =>
      identical(this, other) ||
      (other is NotRebuildText && other.value == value);

  @override
  int get hashCode => value.hashCode;
}
```

每次点击 fab 会 setState，function 每次都会 build，但是 class 只会 build 一次

class 可以

- 定义 const
- 方便定义 keys 和 context
- class 有更好的封装，更便于写出性能更好的代码

### 参考

[Performance best practices](https://docs.flutter.dev/perf/rendering/best-practices)

[StatefulWidget - Performance considerations](https://api.flutter.dev/flutter/widgets/StatefulWidget-class.html#:~:text=Performance%20considerations)

[Building performant Flutter widgets](https://medium.com/flutter/building-performant-flutter-widgets-3b2558aa08fa)

[StatelessWidget vs a function returning Widgets in terms of performance](https://stackoverflow.com/questions/54824933/statelesswidget-vs-a-function-returning-widgets-in-terms-of-performance)

[Reusable widgets: functions or classes?](https://www.flutterclutter.dev/flutter/basics/reusable-widgets-functions-or-classes/2020/138/)
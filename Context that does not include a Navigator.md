> 最近学习flutter，在学到Navigator的时候，发现按到相关demo运行会提示错误：Navigator operation requested with a context that does not include a Navigator。本文就在此揭秘这个错误的由来。

在一个Navigator相关的demo中，有如下代码:
```dart
class MyApp extends StatelessWidget{
  @override
  Widget build(BuildContext context) {
      return MaterialApp(
      title: 'Welcome to Flutter',
      home: Scaffold(
        appBar: AppBar(
          title: const Text('Welcome to Flutter'),
        ),
        body: Center(
          child: FlatButton(child: const Text('Route'),
                  onPressed: () {
                  //导航到新路由
                  Navigator.of(context).push(MaterialPageRoute(builder: (BuildContext context) {
                      return NewRoute(0); // 0 是我demo中测试参数，忽略
                  }));
            })
          )
        )
    );
  }
}
```
以上代码在点击按钮之后，会抛出异常，相关信息则是：
`Navigator operation requested with a context that does not include a Navigator`这句话的意思是，context对象需要包含一个Navigator才能执行导航操作。这句话其实是有道理的，因为我们并没有给定一个导航对象，那么就必须从上下文（context）中找到才能导航。

那么context是怎么找到Navigator的呢？
```dart
static NavigatorState of(
    BuildContext context, {
    bool rootNavigator = false,
    bool nullOk = false,
  }) {
    final NavigatorState navigator = rootNavigator
        ? context.findRootAncestorStateOfType<NavigatorState>()
        : context.findAncestorStateOfType<NavigatorState>();
    assert(() {
      if (navigator == null && !nullOk) {
        throw FlutterError(
          'Navigator operation requested with a context that does not include a Navigator.\n'
          'The context used to push or pop routes from the Navigator must be that of a '
          'widget that is a descendant of a Navigator widget.'
        );
      }
      return true;
    }());
    return navigator;
  }
```
因为没有传入rootNavigator，所以是从`context.findAncestorStateOfType<NavigatorState>();`获取，继续跟踪下去：

```dart
T findAncestorStateOfType<T extends State<StatefulWidget>>() {
    assert(_debugCheckStateIsActiveForAncestorLookup());
    Element ancestor = _parent;
    while (ancestor != null) {
      if (ancestor is StatefulElement && ancestor.state is T)
        break;
      ancestor = ancestor._parent;
    }
    final StatefulElement statefulAncestor = ancestor;
    return statefulAncestor?.state;
  }
```
由上面代码可知，这个链路是不断往上回溯的，那么问题就回到了context，在MyApp类的build代码里面，context是MyApp的Context，这里没有Navigator（Scaffold组件才有，而Scaffold组件是MyApp的子组件），继续往上回溯就更加找不到了。

所以目前能找到的大多数解法都是把FlatButton新建一个Widget，在这个新Widget里的build方法的context是FlatButton的，往上回溯就能找到Scaffold组件，从而正常使用Navigator了。

另外一种解法是，FlatButton不直接写入，而使用Builder来写，这样会能用Builder中的context来回溯了。

附上两种解法：
使用新Widget的解法：
```dart
class RouteButton extends StatelessWidget {
  @override
  Widget build(BuildContext context) {
    return 
       FlatButton(onPressed: () {
          //导航到新路由
          Navigator.of(context).push<MaterialPageRoute>(MaterialPageRoute(builder: (BuildContext context) {
              return NewRoute();
          }));
          },
        child: const Text('Route'));
  }
}
```

使用Builder的解法：
```dart
class MyApp extends StatelessWidget{
  @override
  Widget build(BuildContext context) {
      return MaterialApp(
      title: 'Welcome to Flutter',
      home: Scaffold(
        appBar: AppBar(
          title: const Text('Welcome to Flutter'),
        ),
        body: Center(
          child: 
          Builder(builder: (BuildContext context){
            return FlatButton(child: const Text('Route'),
                  onPressed: () {
                  //导航到新路由
                  Navigator.of(context).push(MaterialPageRoute(builder: (BuildContext context) {
                      return NewRoute(0);
                  }));
            });
          })
          )
        )
    );
  }
}
```

另外提一句，上面的代码在VSCode 中会报错，但是不影响实际运行：
![image.png](https://upload-images.jianshu.io/upload_images/674044-f3e85fe1574c26a1.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
如果想去掉这个报错，就需要这么写:
```dart
Navigator.of(context).push<MaterialPageRoute>(MaterialPageRoute(builder: (BuildContext context) {
    return NewRoute(0); // 这个0是我这边Widget需要的参数
}));
```
应该是与范型相关的问题，这个应该在dart语法中有讲解。
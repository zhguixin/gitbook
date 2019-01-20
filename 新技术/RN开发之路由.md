使用第三路由框架：

React Router，渲染的根节点变为`Router` ：

```react
ReactDOM.render((
    <Router ...>
        <Route ...>
        <Route ../>
        ...
        </Route>
        <Route .../>
    </Router>), document.getElementById('content'))
```

其中`Route` 标签有两个必须要实现的属性：

* path，表示要跳转到的路径；
* component，路径所要渲染的组件

举例：

```react
ReactDOM.render((
  <Router history={hashHistory}>
    <Route path="/" component={Content} >
      <Route path="/about" component={About} />
      <Route path="/posts" component={Posts} posts={posts}/>
      <Route path="/posts/:id" component={Post}  posts={posts}/>
      <Route path="/contact" component={withRouter(Contact)} />
    </Route>
    <Route path="/login" component={Login}/>
  </Router>
), document.getElementById('content'))
```

其中属性`posts` ，可以在组件通过`props.route.posts` 得到，它的值就是一个json串。


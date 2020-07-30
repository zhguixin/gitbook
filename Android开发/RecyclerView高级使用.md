RecyclerView高级使用

RecyclerView应该说是在Android中做信息展示控件里最常用的了。不同于ListView，只能展示垂直布局的信息，RecyclerView将布局管理器解耦出来，并默认提供了`LinerLayoutManager` 、`GridLayoutManager` 、`StaggeredGridLayoutManager` 三个布局管理器。

在缓存方面，要求用户强制实现`onCreateViewHolder` 和`onBindViewHolder` ，来减少每次加载View。通过使用VIewHolder也可以减少findViewById的次数。

##### 添加HeaderView和FooterView

常规的做法：在编写Adapter时，定义一个特殊的Item，通过定义ViewType来区分，HeaderView或者FooterView。在Adapter中涉及到：`getItemType`、`getItemCount`、`onBindViewHolder`、`onCreateViewHolder` 这几个函数。

如果是GridLayoutManager的话，需要设置FooterView或者HeadView占据多个单元格：

```java
    @Override
    public void onAttachedToRecyclerView(RecyclerView recyclerView) {
        super.onAttachedToRecyclerView(recyclerView);
        RecyclerView.LayoutManager manager = recyclerView.getLayoutManager();
        if (manager instanceof GridLayoutManager) {
            final GridLayoutManager gridManager = (GridLayoutManager)manager;
            gridManager.setSpanSizeLookup(new GridLayoutManager.SpanSizeLookup() {
                @Override
                public int getSpanSize(int position) {
                    // 返回值决定了每个Item占据的单元格数
                    return getItemViewType(position) == FOOTER ?
                            gridManager.getSpanCount() : 1;
                }
            });
        }
    }
```

为了保持Adapter的干净清爽，可以使用**装饰者模式**，将这些特殊的ItemView，单独成一类，比如将FooterView封装为ItemFooterWrapper这个类，在这个类中处理FooterView的逻辑：

```java
public class ItemFooterWrapper extends RecyclerView.Adapter<RecyclerView.ViewHolder> {
  private RecyclerView.Adapter mAdapter；
  
  // 构造函数将原来的Adapter传递过来
  public LoadMoreWrapper(RecyclerView.Adapter adapter) {
        mAdapter = adapter;
  }
  
    @Override
    public RecyclerView.ViewHolder onCreateViewHolder(ViewGroup parent, int viewType) {
      if (viewType == TYPE_FOOTER) {
        // 解析FooterView
      } else {
          mAdapter.onCreateViewHolder(parent, viewType);
      }
    }
  
    @Override
    public void onBindViewHolder(RecyclerView.ViewHolder holder, int position) {
      if (holder instanceof FootViewHolder) {
        // 处理FooterView的逻辑
      } else {
          mAdapter.onBindViewHolder(holder, position);
      }
    }
  
  // ....
}
```

使用的时候：

```java
ItemFooterWrapper itemFooterWrapper = new ItemFooterWrapper(ItemAdapter);
recyclerView.setAdapter(itemFooterWrapper);
```

可以看到使用这种方式是非侵入式的。

#####RecyclerView的View的复用机制原理解析

RecyclerView有多级缓存，常说的4级缓存:

- ArrayList<ViewHolder> mAttachedScrap：缓存屏幕上可见的ViewHolder，不需要创建视图和重新绑定数据

- ArrayList<ViewHolder> mCachedViews：缓存即将离开屏幕的ViewHolder，不需要创建视图和重新绑定数据

- ViewCacheExtension mViewCacheExtension：用户自定义缓存，默认为空实现

- RecycledViewPool mRecyclerPool：ViewHolder的缓存池，每个不同的type会缓存5个，需要重新绑定数据

- RecyclerView重新布局，调用对应的LayoutManager的`onLayoutChildren`

https://blog.csdn.net/singwhatiwanna/article/details/100166495

https://www.jianshu.com/p/e1b257484961

参考[基于滑动场景解析RecyclerView的回收复用机制原理](http://www.jianshu.com/p/9306b365da57)




#### 基本概念

MVVM架构，在MVP基础上，将P层换成了VM层。

即Presenter -- >> ViewModel

#### 架构设计

View层依然作为界面展示层，View层和ViewModel层通过LiveData进行数据交互，并遵循以下原则：

* ViewModel层不持有Activity、Fragment以及其他View的引用。这主要是为了避免配置发生变化时，Activity被销毁，ViewModel持有Activity的引用，而无法被回收导致内存泄漏。

* ViewModel暴露给View的数据是LiveData对象

* 在View显示层，观察相应的LiveData

  ```java
  // Update the list of products when the underlying data changes.
  viewModel.getProducts().observe(this, new Observer<List<ProductEntity>>() {
      @Override
      public void onChanged(@Nullable List<ProductEntity> myProducts) {
          if (myProducts != null) {
              mBinding.setIsLoading(false);
              mProductAdapter.setProductList(myProducts);
          } else {
              mBinding.setIsLoading(true);
          }
      }
  });
  ```

* 在Model层，获取到的数据，返回为LiveData类型

  ```java
  @Query("select * from products")
  LiveData<List<ProductEntity>> loadProducts();
  ```




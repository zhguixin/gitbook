在Android开发过程中，经常会实现跨进程通信的操作，这个时候需要将复杂的数据结构进行序列化后，才能跨进程通信。

在Java中，有**Serializable** 这么一个标准序列化接口。在Android中，Google工程师提供了**Parcelable** 序列化接口，号称可以比**Serializable** 快10倍。

> 参考：Parcelable和Serializable的效率对比 [Parcelable vs Serializable](http://www.developerphil.com/parcelable-vs-serializable/) 

### Parcelable序列化接口的使用

在**Parcelable** 接口源码中提供了这么一个示例：

```java
public class MyParcelable implements Parcelable {
    private int mData;
    // 负责描述文件,一般情况下返回0即可
    public int describeContents() {
      return 0;
    }
    // 序列化，写入数据
    public void writeToParcel(Parcel out, int flags) {
        out.writeInt(mData);
    }
    // 创建Creator实例，覆写 createFromParcel、newArray方法
    public static final Parcelable.Creator<MyParcelable> CREATOR
            = new Parcelable.Creator<MyParcelable>() {
        public MyParcelable createFromParcel(Parcel in) {
          return new MyParcelable(in);
        }
        public MyParcelable[] newArray(int size) {
          return new MyParcelable[size];
        }
    };
    // 反序列化，读取数据
    private MyParcelable(Parcel in) {
      mData = in.readInt();
    }
}
```

从中可以看出，**MyParcelable** 是通过`Parcel` 写入和恢复数据的，并且必须要有一个非空的静态变量 `CREATOR` 

### Parcel序列化原理

实现`Parcelable` 序列化接口后，主要就是借助于Parcel进行序列化操作了：

```java
writeInt(i);
writeString(str);
// ...

readInt();
readString();
// ...
```

打开上述方法的实现，都是native方法：

```java
// writeString
public final void writeString(String val) {
    mReadWriteHelper.writeString(this, val);
}
public void writeString(Parcel p, String s) {
    nativeWriteString(p.mNativePtr, s);
}
// readString
public final String readString() {
    return mReadWriteHelper.readString(this);
}
public String readString(Parcel p) {
    return nativeReadString(p.mNativePtr);
}
```

Parcel通过调用native方法，可以将序列化之后的数据写入到一个共享内存中，其他进程通过Parcel可以从这块共享内存中读出字节流，并反序列化成对象。

其中`mNativePtr` 指向C++侧，指向了起始内存地址。
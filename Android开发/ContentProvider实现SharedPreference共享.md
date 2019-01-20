### 简介

ContentProvider是Android四大组件中的一个，被称为内容提供器，它主要提供在不同应用程序间实现数据共享，并且保证了共享数据的安全性。

如果想要访问内容提供器中共享的数据，就一定要借助**ContentResolve**类，可以通过 Context 中的 `getContentResolver()`方法获取到该类的实例。 ContentResolver 中提供了一系列的方法用于对数据进行 CRUD 操作。

### 实现

创建自己的内容提供器，继承**ContentProvider** 

```java
public class SPContentProvider extends ContentProvider {
    private static final String SEPARATOR = "/";
    private static final String VALUE= "value"

    @Override
    public boolean onCreate() {
        return true;
    }

    @Nullable
    @Override
    public Cursor query(Uri uri, String[] projection, String selection, String[] selectionArgs, String sortOrder) {
        return null;
    }

    @Nullable
    @Override
    public String getType(Uri uri) {
        String[] path= uri.getPath().split(SEPARATOR);
        String type=path[1];
        String key=path[2];
        return  "" + SPHelperImpl.get(getContext(),key,type);
    }

    @Nullable
    @Override
    public Uri insert(Uri uri, ContentValues values) {
        String[] path = uri.getPath().split(SEPARATOR);
        String key = path[2];
        Object obj = (Object) values.get(VALUE);
        if (obj != null) {
            SPHelperImpl.save(getContext(), key, obj);
        }
        return null;
    }

    @Override
    public int delete(Uri uri, String selection, String[] selectionArgs) {
        String[] path= uri.getPath().split(SEPARATOR);
        String type=path[1];
        String key=path[2];
        
        if (SPHelperImpl.contains(getContext(),key)){
            SPHelperImpl.remove(getContext(),key);
        }
        return 0;
    }

    @Override
    public int update(Uri uri, ContentValues values, String selection, String[] selectionArgs) {
        insert(uri,values);
        return 0;
    }
}
```

从上述代码可以看出需要覆写`query` 、`insert`、`update` 和`delete` 方法，供使用这个内容提供器的用户程序实现CRUD操作，这几个方法都有一个参数：**Uri** ，通过匹配后的Uri才能进行数据操作，保证了数据安全性。

另外，还有一个要覆写的方法：即` getType()`方法，它是所有的内容提供器都必 须提供的一个方法，用于获取 Uri 对象所对应的 MIME 类型。在这里，是为SharedPreference提供数据共享，需要特殊处理：

```java
public String getType(Uri uri) {
    String[] path= uri.getPath().split(SEPARATOR);
    String type = path[1];
    String key = path[2];
    return  "" + SPHelperImpl.get(getContext(),key,type);
}

static String get(Context context, String name, String type) {
    Object value = get_impl(context, name, type);
    return value + "";
}

private static Object get_impl(Context context, String name, String type) {
    if (!contains(context, name)) {
        return null;
    } else {
        if (type.equalsIgnoreCase(TYPE_STRING)) {
            return getString(context, name, null);
        } else if (type.equalsIgnoreCase(TYPE_BOOLEAN)) {
            return getBoolean(context, name, false);
        } else if (type.equalsIgnoreCase(TYPE_INT)) {
            return getInt(context, name, 0);
        } else if (type.equalsIgnoreCase(TYPE_LONG)) {
            return getLong(context, name, 0L);
        } else if (type.equalsIgnoreCase(TYPE_FLOAT)) {
            return getFloat(context, name, 0f);
        }
        return null;
    }
}
```

因为是要为SharedPreference提供数据接口，我们约定用户程序传入的Uri的形式为：

```java
"content://com.zgx.myapplication/boolean/isReboot/value"
```

因此通过上述解析，type为`boolean` ，key为`isReboot` 。

然后在**AndroidManifest**中注册这个ContentProvider：

```xml
<provider
   android:authorities="com.zgx.myapplication"
   android:name=".sphelper.SPContentProvider"
   android:directBootAware="true"
   android:readPermission="com.zgx.myapplication.read"
   android:writePermission="com.zgx.myapplication.write"
   android:exported="true"
   tools:targetApi="n" />
```

`android:authorities`指定了ContentProvider监听URI的host为：`com.zgx.myapplication` ，scheme固定为：`content` ，因此完整的URI路径为：**content://com.zgx.myapplication**

> 在provider标签中，还指定了访问权限，权限定义如下：
>
> ```xml
> <permission
>     android:name="com.zgx.myapplication.read"
>     android:label="provider read permission"
>     android:protectionLevel="normal" />
> <permission
>     android:name="com.zgx.myapplication.write"
>     android:label="provider write permission"
>     android:protectionLevel="normal" />
> ```
>
> 只有访问用户声明了权限才能使用，进一步保证了数据安全性。
>
> ```xml
> <uses-permission android:name="com.zgx.myapplication.write" />
> <uses-permission android:name="com.zgx.myapplication.read" />
> ```

到这里，一个完整的内容提供器就创建完成了，现在任何一个应用程序都可以使用 **ContentResolver** 来访问我们程序中的数据。  

### 使用

在另外一个用户程序中，使用如下：

```java
private static final String CONTROL_URI = "content://com.zgx.myapplication/boolean/";
private static final String KEY = "isReboot";

try {
    ContentResolver cr = mContext.getContentResolver();
    Uri uri = Uri.parse(CONTROL_URI + KEY);
    ContentValues cv = new ContentValues();
    cv.put("value", value);
    cr.update(uri, cv, null, null);
} catch (Exception e) {
    e.printStackTrace();
}
```

ContentProvider作为内容提供器，肯定不会只有一个程序调用，如果我们的程序需要”被动“获取内容提供器的数据，怎么办呢，在内容提供器中的数据发生了变化后通知我们。

1. 如果ContentProvider的访问者需要知道ContentProvider中的数据发生了变化，可以在ContentProvider 发生数据变化时调用**getContentResolver().notifyChange(uri, null)** 来通知注册在此URI上的访问者

   ```java
   public Uri insert(Uri uri, ContentValues values) {
       String[] path = uri.getPath().split(SEPARATOR);
       String key = path[2];
       Object obj = (Object) values.get(VALUE);
       if (obj != null) {
           SPHelperImpl.save(getContext(), key, obj);
           // 增加如下一句
           getContext().getContentResolver().notifyChange(uri, null);
       }
       return null;
   }
   ```

2. 如果ContentProvider的访问者需要得到数据变化通知，必须使用**ContentObserver**对数据（数据采用uri描述）进行监听，当监听到数据变化通知时，系统就会调用ContentObserver的`onChange()`方法

   ```java
   getContentResolver().registerContentObserver(Uri.parse(CONTROL_URI+KEY), true, new   ContentObserver() {
       
       @Override
       public void onChange(boolean selfChange) {
           // 此处可以进行相应的业务处理
       }
   });
   
   ```

> **ContentObserver**的构造函数中可以传入一个Handler，这个Handler的创建线程决定了`onChange`中业务处理的线程，默认不传的会，`onChange` 工作在UI线程。

完结:smile:
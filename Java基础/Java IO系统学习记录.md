---

title: Java IO系统学习记录
date: 2017-10-20
tags: Android

categories:

---

在Java中有关输入输出流的概念，之前一直比较模糊。今天抽了一天学习记录一下。

#### 流的概念

java对存储在硬盘上的文件或者目录，都用**File**这个类来描述，比如用`file`表示硬盘上的一个文件：

```java
  String path = "/data/misc/wifi/";
  String name = "wpa_supplicant.conf";
  File file = new File(path, name);
```

File的构造函数比较多：

```java
public File(String pathname)
public File(String parent, String child)
public File(File parent, String child)
// ...详细的可以查看API
```

#####字节流

拿到了`file`这个”文件“，我们可以通过输入流（InputStream）读到内存中，这里要注意，你不能一次性读到内存中，要不然会造成内存溢出。所以需要有个缓冲区，先读到缓存区（buffer）里。然后再从缓存区里读取。

```java
  int len;// len被赋值为每次读取到buf中的大小，文件读取完毕后，len小于0
  byte[] buf = new byte[1024]; // 缓存区大小为1024，每次读取的大小即为1024(文件大小超过了1024)
  InputStream inputStream = null;
  try {
    inputStream = new FileInputStream(file);
    while ((len = inputStream.read(buf)) != -1) {
      stringBuilder.append(buf);
    }
  } catch (IOException e) {
    e.printStackTrace();
  }
```

上面这个是从文件中，以字节流的方式读取到内存。以字节（byte）为单位。

##### 字符流

为了方便处理文本文件，java引入字符流。看一个例子：

```java
    try {
      InputStream is = new FileInputStream(file);
      InputStreamReader input = new InputStreamReader(is, "UTF-8");
      BufferedReader reader = new BufferedReader(input);
      while ((str = reader.readLine()) != null) { //可以按行读取
        stringBuilder.append(str);
        stringBuilder.append("\n");
      }
	 catch (IOException e) {
      // TODO Auto-generated catch block
      e.printStackTrace();
    }
```

其中，先将字节流`inputStream`包装成`InputStreamReader`（此时我们可以指定编码格式）。随后，将`InputStreamReader`放到`BufferedReader`中，通过这个`BufferedReader`我们就可以按行读取文本文件了。



Java IO流中，基础的**字节流**接口有：`InputStream`和` OutputStream` ，**字符流** 接口有：`Reader` 和`Writer`。


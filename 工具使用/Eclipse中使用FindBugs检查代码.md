### 1.什么是FindBugs
FindBugs 是一个静态分析工具，它检查类或者 JAR 文件，将字节码与一组缺陷模式进行对比以发现可能的问题。有了静态分析工具，就可以在不实际运行程序的情况对软件进行分析。
### 2.如何安装FindBugs
FindBugs在Eclipse中的安装同其它形式的插件的安装一样，主要有两种：
第一种就是在线安装：点击菜单栏，Help--->Install New SoftWare…在打开的界面，点击Add，输入：`http://findbugs.cs.umd.edu/eclipse`
 
第二种就是离线安装，将附件中的压缩包解压后放到Eclipse安装目录中的plugins目录下即可。
这时重启Eclipse，为了验证是否安装成功，可以在菜单栏点击Window--->Preference，在搜索框输入：findbugs
 
如果出现界面上出现FindBugs的相关信息说明安装成功。

### 3.如何使用FindBugs
FindBugs安装成功后，就可以对项目工程findbugs检查。检查的方法也很简单，直接在项目工程点击右键，选择FingBugs即可。可以将FindBugs的视窗打开： 
在打开的FingBugs视窗中，可以看到FingBugs对整个项目的检查结果，[FindBugs 网站](http://findbugs.sourceforge.net/bugDescriptions.html)提供了完整的类型清单。FindBugs相关的修改说明，参考之前发的pdf文件。
 
找出的bug有3中颜色，黑色的臭虫标志是分类，红色的臭虫表示严重bug发现后必须修改代码，橘黄色的臭虫表示潜在警告性bug尽量修改。
双击bug项目就可以在右边编辑窗口自动打开相关代码文件并链接到代码片段。点击行号旁边的小臭虫图标后在eclipse下方输出区将提供详细的bug描述，以及修改建议等信息。我们可以根据此信息进行修改。


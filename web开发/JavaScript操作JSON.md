#### JSON使用

Json是一种独立的数据格式、语言。常用在JavaScript中，用来从服务器读取数据（通常是字符串），解析后（调用JSON.parse(jsonStr)）得到JSON格式的对象，在js中直接使用。

使用 AJAX 从服务器请求 JSON 数据，并解析为 JavaScript 对象

```javascript
var xmlhttp = new XMLHttpRequest();
xmlhttp.onreadystatechange = function() {
    if (this.readyState == 4 && this.status == 200) {
        myObj = JSON.parse(this.responseText);
        document.getElementById("demo").innerHTML = myObj.name;
    }
};
xmlhttp.open("GET", "/try/ajax/json_demo.txt", true);
xmlhttp.send();
```

反之，js可以调用`JSON.stringify(jsonObj)`将JSON对象转换为字符串上报给服务器。



在JavaScript操作JSON：

```javascript
	// 如果是字符串就需要装换为js对象
	//var obj = JSON.parse(str);
      var text = {
      "username": "andy",
      "age": 20,
      "info": {
          "tel": "123456",
          "cellphone": "98765"
      },
      "address": [
          {
              "city": "beijing",
              "postcode": "222333"
          },
          {
              "city": "newyork",
              "postcode": "555666"
          }
      ]
	}
	document.write(text.username + " " + text.age)
	document.write("<br>")
	document.write(text.info.tel + " " + text.info.cellphone)
	document.write("<br>")
	for(var i = 0; i<text.address.length;i++){
		document.write(text.address[i].city + " " + text.address[i].postcode)
		document.write("<br>")
	}
```

我们在JavaScript中可以操作相应的html标签，利用`getElementById`等操作：

```javascript
document.getElementById("username").innerHTML = text.username;
```

向id为**username**的标签中插入内容。

为了简化操作，可以使用JQuery库。

基础语法： **$(selector).action()**

- 美元符号定义 jQuery
- 选择符（selector）"查询"和"查找" HTML 元素
- jQuery 的 action() 执行对元素的操作

比如：

- $(this).hide() - 隐藏当前元素
- $("p").hide() - 隐藏所有 <p> 元素
- $("p.test").hide() - 隐藏所有 class="test" 的 <p> 元素
- $("#test").hide() - 隐藏所有 id="test" 的元素

如果希望异步加载数据，就要用的AJAX，调用jQuery的load方法去请求数据，load() 方法从服务器加载数据，并把返回的数据放入被选元素中即：`$(selector)`

```javascript
$(selector).load(URL,data,callback);
```

其中，url要访问的地址，data可选：可携带参数，callback可选：回调函数
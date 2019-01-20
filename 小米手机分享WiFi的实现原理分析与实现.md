---
title: 小米手机分享WiFi的实现原理分析与实现
date: 2017-10-17
tags: Android
categories:
---

小米的MIUI从很早的版本就集成了一个功能，已连接WiFi的可以生成二维码，供小米手机扫描直接连接。这里面有三个技术点：

- 得到已连接wifi的相关信息
- 生成二维码
- 二维码的解析

接下来分别介绍之。

#### 获取已连接WiFi的信息

wifi密码保存在`/data/misc/wifi/wpa_supplicant.conf` 。

#### 二维码的生成

二维码的开发使用我们大多都是使用Google提供的zxing这个类库（[Github](https://github.com/zxing/zxing) ）。[Jar包下载地址](http://central.maven.org/maven2/com/google/zxing/core/3.2.1/core-3.2.1.jar)

二维码图片其实就是个Bitmap，生成二维码Bitmap的代码如下：

```java
private Bitmap generateBitmap(String content,int width, int height) {  
    QRCodeWriter qrCodeWriter = new QRCodeWriter();  
    Map<EncodeHintType, String> hints = new HashMap<>();  
    hints.put(EncodeHintType.CHARACTER_SET, "utf-8");  
    try {  
        BitMatrix encode = qrCodeWriter.encode(content, BarcodeFormat.QR_CODE, width, height, hints);  
        int[] pixels = new int[width * height];  
        for (int i = 0; i < height; i++) {  
            for (int j = 0; j < width; j++) {
                if (encode.get(j, i)) {  
                    pixels[i * width + j] = 0x00000000;  // 含有信息，填充为黑色
                } else {  
                    pixels[i * width + j] = 0xffffffff;  // 没有信息，白色
                }  
            }  
        }  
        return Bitmap.createBitmap(pixels, 0, width, width, height, Bitmap.Config.RGB_565);  
    } catch (WriterException e) {  
        e.printStackTrace();  
    }  
    return null;  
}
```

#### 二维码的解析


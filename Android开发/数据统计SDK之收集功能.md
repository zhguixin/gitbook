#### 数据统计SDK之收集功能

客户端调用：`prepareEvent()`方法，触发数据收集流程，收集到的数据转换为JSON串后以字符串的形式存到数据库。



```sequence
title: 数据统计上报
participant ZTEStatistics as A
participant ChainedEventObj as B
participant ActionManager as C
participant InfoRouter as D
participant EventsDao as E

A->B:prepareEvent()
B->B:push()
B->B:param()
B->B:emit()
B->>B:insertEvent()

B->C:onEvent()

note right of C: 插入到长度为100的阻塞队列中

C->D:insert()
D-->>C:insertEventToDB()

C->E:insert()
```



#### 数据统计SDK之数据上报功能

在数据收集存储到数据库时，还会检查是否要上报：

```java
private void checkIfUpload() {
    if (timeToUploadData() || new EventsDao().getCounts() >= 500) {
        PrefercenUtil.setLastUploadDataTime(mContext, System.currentTimeMillis()/1000);
        PayloadManager.getInstance().ensureRunning(null, mContext);
    }
}
```

满足上报条件后，在**ZTEStatisticsClient**中通过`HttpsUrlConnection` 上报至服务器，上报的数据从数据库查询得到，经过AES加密、GZIP压缩后传递到服务器端。

```sequence
participant ChainedEventObj as A
participant PayloadManager as B
participant EventsDao as C
participant ZTEStatisticsClient as D

A->A:emit()
A->>A:checkIfUpload()
A->B:ensureRunning()

B->C:getJsonString()
B->D:postJSON()
B->C:deleteRecord()

```

与服务器的连接使用了`javax.net.ssl`包下的**HttpsURLConnection**类来建立HTTPS连接。

```java
private int postHttpsJSON(String json) {
    IOException e;
    Throwable th;
    try {
        byte[] buf = getAesStr(json);
        if (buf == null) {
            return -1;
        }
        int code;
        Log.d("postHttpsJSON buf length=" + buf.length, new Object[0]);
        // 在HTTP协议之上加入了验证
        SSLContext sc = SSLContext.getInstance("TLS");
        sc.init(null, new TrustManager[]{new MyTrustManager()}, new SecureRandom());
        HttpsURLConnection.setDefaultSSLSocketFactory(sc.getSocketFactory());
        HttpsURLConnection.setDefaultHostnameVerifier(new MyHostnameVerifier());
        HttpsURLConnection conn = (HttpsURLConnection) new URL(ENDPOINT_BASE).openConnection();
        conn.setRequestMethod("POST");
        conn.setRequestProperty("Content-Type", "application/json");
        conn.setRequestProperty("Accept", "application/json");
        conn.setRequestProperty("Content-Encoding", "gzip");
        conn.setRequestProperty("Version-Code", ConstantDefine.VERSION_CODE);
        conn.setDoOutput(true);
        conn.setDoInput(true);
        conn.connect();
        GZIPOutputStream zos = null;
        try {
            GZIPOutputStream zos2 = new GZIPOutputStream(conn.getOutputStream());
            try {
                zos2.write(buf, 0, buf.length);
                if (zos2 != null) {
                    zos2.close();
                    zos = zos2;
                }
            } catch (IOException e2) {
                e = e2;
                zos = zos2;
                try {
                    e.printStackTrace();
                    if (zos != null) {
                        zos.close();
                    }
                    code = conn.getResponseCode();
                    conn.disconnect();
                    return code;
                } catch (Throwable th2) {
                    th = th2;
                    if (zos != null) {
                        zos.close();
                    }
                    throw th;
                }
            } catch (Throwable th3) {
                th = th3;
                zos = zos2;
                if (zos != null) {
                    zos.close();
                }
                throw th;
            }
        } catch (IOException e3) {
            e = e3;
            e.printStackTrace();
            if (zos != null) {
                zos.close();
            }
            code = conn.getResponseCode();
            conn.disconnect();
            return code;
        }
        code = conn.getResponseCode();
        conn.disconnect();
        return code;
    } catch (Exception e4) {
        Log.e(e4.getMessage(), new Object[0]);
        return -1;
    }
}

```

上述采用了HTTPS协议，但是验证证书的过程中，默认信任全部证书。

> HTTPS采用非对称加密的技术建立连接，证书的作用，就是将服务端的公钥包裹后发到客户端，来表明：这个公钥确实是来自服务器的公钥。

```java
private class MyTrustManager implements X509TrustManager {
    private MyTrustManager() {
    }

    // 检查客户端证书
    public void checkClientTrusted(X509Certificate[] chain, String authType) throws 
        CertificateException {
    }

    // 检查服务器端证书
    public void checkServerTrusted(X509Certificate[] chain, String authType) throws
        CertificateException {
    }

    // 返回受信任的X509证书数组
    public X509Certificate[] getAcceptedIssuers() {
        return null;
    }
}


private class MyHostnameVerifier implements HostnameVerifier {
    private MyHostnameVerifier() {
    }

    public boolean verify(String hostname, SSLSession session) {
        return true;
    }
}
```

信任全部证书，这种方式虽然安全级别不高，但毕竟是还是HTTPS链接，安全性上也还是要高于HTTP链接的。

那么如何使用自己的证书来完成HTTPS链接呢？

首先将获取到的证书放到APP工程的`assets` 目录下，然后读取，返回SSLContext：

```java
public static SSLContext getSSLContext(Context inputContext){
    SSLContext context = null;
    try {
        CertificateFactory cf = CertificateFactory.getInstance("X.509");
        InputStream in = inputContext.getAssets().open("root.crt");
        Certificate ca = cf.generateCertificate(in);
        KeyStore keystore = KeyStore.getInstance(KeyStore.getDefaultType());
        keystore.load(null, null);
        keystore.setCertificateEntry("ca", ca);
        String tmfAlgorithm = TrustManagerFactory.getDefaultAlgorithm();
        TrustManagerFactory tmf = TrustManagerFactory.getInstance(tmfAlgorithm);
        tmf.init(keystore);
        // Create an SSLContext that uses our TrustManager
        context = SSLContext.getInstance("TLS");
        context.init(null, tmf.getTrustManagers(), null);
    } catch (Exception e){
        e.printStackTrace();
    }
    return context;
}
```

建立连接的时候，使用自己指定的证书：

```java
conn.setSSLSocketFactory(getSSLContext(context).getSocketFactory());
```


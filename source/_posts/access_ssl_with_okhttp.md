---
title: 使用OkHttp访问ssl（https）网络
category:
  - Android
tags:
  - Android
  - 填坑
date: 2015-06-11 07:55:40
description: "在android项目中使用OkHttp的过程，和两个小坑"
---

直接抄网上的示例，发现会证书认证失败：unable to find valid certification path to requested target 
需要先配置SSLcontext和SSLSocketFactory。

直接上代码
```
try {
sslContext = SSLContext.getInstance("TLS");
} catch (NoSuchAlgorithmException e) {
e.printStackTrace();
}

TrustManager tm = new X509TrustManager() {
@Override
public void checkClientTrusted(X509Certificate[] chain,
String authType) throws CertificateException {
}

@Override
public void checkServerTrusted(X509Certificate[] chain,
String authType) throws CertificateException {
}

@Override
public X509Certificate[] getAcceptedIssuers() {
return null;
}
};

try {
sslContext.init(null, new TrustManager[] { tm }, null);
} catch (KeyManagementException e) {
e.printStackTrace();
}

javax.net.ssl.SSLSocketFactory factory = sslContext.getSocketFactory();  

client.setSslSocketFactory(factory);

client.setHostnameVerifier(new HostnameVerifier() {

@Override
public boolean verify(String arg0, SSLSession arg1) {
return true;
}
});

FormEncodingBuilder builder=new FormEncodingBuilder();
if (postContent != null && postContent.size() > 0) {
Iterator> i = postContent.entrySet().iterator();
while (i.hasNext()) {
Entry entry = i.next();
if (entry.getValue() == null) {
log.info("null -->" + entry.getKey());
continue;
}
builder.add(entry.getKey(), (String)entry.getValue());
}

RequestBody body = builder.build();

Request request = new Request.Builder().url(url).post(body).build();

Response response = client.newCall(request).execute();
//注意這裡，response.body().string()只能運行一次，response的內容只能取一次
sb = response.body().string();
System.out.println("sb is " + sb);
```

## SpringBoot设置请求参数长度和文件大小限制

### 项目环境

客户端和服务端都是 spring boot 2.1.4

### 1. 问题

客户端post请求服务端，发送的数据大小为5MB。客户端提示如下异常：

```
Exception in thread "main" org.springframework.web.client.ResourceAccessException: I/O error on POST request for "http://localhost:9100/account/upload": Software caused connection abort: recv failed; nested exception is java.net.SocketException: Software caused connection abort: recv failed
	at org.springframework.web.client.RestTemplate.doExecute(RestTemplate.java:744)
	at org.springframework.web.client.RestTemplate.execute(RestTemplate.java:670)
	at org.springframework.web.client.RestTemplate.postForObject(RestTemplate.java:414)
	at org.dmqk.cloud.service.pay.service.impl.ReconciliationRecordServiceImpl.main(ReconciliationRecordServiceImpl.java:252)
Caused by: java.net.SocketException: Software caused connection abort: recv failed
	at java.net.SocketInputStream.socketRead0(Native Method)
	at java.net.SocketInputStream.socketRead(SocketInputStream.java:116)
	at java.net.SocketInputStream.read(SocketInputStream.java:171)
	at java.net.SocketInputStream.read(SocketInputStream.java:141)
	at java.io.BufferedInputStream.fill(BufferedInputStream.java:246)
	at java.io.BufferedInputStream.read1(BufferedInputStream.java:286)
	at java.io.BufferedInputStream.read(BufferedInputStream.java:345)
	at sun.net.www.http.HttpClient.parseHTTPHeader(HttpClient.java:704)
	at sun.net.www.http.HttpClient.parseHTTP(HttpClient.java:647)
	at sun.net.www.protocol.http.HttpURLConnection.getInputStream0(HttpURLConnection.java:1569)
	at sun.net.www.protocol.http.HttpURLConnection.getInputStream(HttpURLConnection.java:1474)
	at java.net.HttpURLConnection.getResponseCode(HttpURLConnection.java:480)
	at org.springframework.http.client.SimpleClientHttpResponse.getRawStatusCode(SimpleClientHttpResponse.java:55)
	at org.springframework.web.client.DefaultResponseErrorHandler.hasError(DefaultResponseErrorHandler.java:55)
	at org.springframework.web.client.RestTemplate.handleResponse(RestTemplate.java:766)
	at org.springframework.web.client.RestTemplate.doExecute(RestTemplate.java:736)
	... 3 more
```

### 2. 原因

spring boot项目中Tomcat默认的post请求最大为2MB

### 3. 解决办法

在配置文件中增加如下配置，修改最大请求限制

```yaml
server:
  tomcat:
    max-http-post-size: -1	# -1表示没有限制
```

### 4. 扩展

- 发送大数据量请求时，可以考虑对请求压缩处理
- header中传输大量的内容，同样存在超过最大限制的问题
- 上传文件大小限制




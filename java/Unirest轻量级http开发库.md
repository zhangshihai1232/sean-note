## 轻量级http开发库Unirest
## 一. 特点
可以被PHP、Ruby、Python、Java、Objective-C等语言调用
支持GET、POST、PUT、UPDATE、DELETE操作，调用方法和返回类型对所有语言都是相同的
可以利用下面代码发送httprequest

```java
Unirest.post("http://httpbin.org/post")
	  .queryString("name", "Mark")
	  .field("last", "Polo")
	  .asJson()
```

特点：
- 请求：GET, POST, PUT, PATCH, DELETE, HEAD, OPTIONS
- 异步+同步模式
- 支持表单参数、文件上传
- gzip、基本的本地认证、proxy、timeout
- 默认headers、HttpClient、HttpAsyncClient、转json

maven

```java
<dependency>
    <groupId>com.mashape.unirest</groupId>
    <artifactId>unirest-java</artifactId>
    <version>1.4.9</version>
</dependency>
```

## 二. 使用
**创建request**
基本的post请求

```java
HttpResponse<JsonNode> jsonResponse = Unirest.post("http://httpbin.org/post")
            .header("accept", "application/json")
            .queryString("apiKey", "123")
            .field("parameter", "value")
            .field("foo", "bar")
            .asJson();
```
当执行了as[Type]()方法之后，Requests就被生成了，可以转换的内容为`Json, Binary, String, Object`
.body(String|JsonNode|Object)，当使用.body(Object) 时需要额外的配置参数
.fields(Map<String, Object> fields)会把每个k-v填入form表单
.headers(Map<String, String> headers)填入headers参数


**Serialization**
当执行`asObject(Class)`或者`.body(Object)`之前，需要一个ObjectMapper的定制实现，在最初就要运行，因为ObjectMapper是全局共享的
比如使用Jackson序列化json，如下

```java
// Only one time
Unirest.setObjectMapper(new ObjectMapper() {
    private com.fasterxml.jackson.databind.ObjectMapper jacksonObjectMapper
                = new com.fasterxml.jackson.databind.ObjectMapper();

    public <T> T readValue(String value, Class<T> valueType) {
        try {
            return jacksonObjectMapper.readValue(value, valueType);
        } catch (IOException e) {
            throw new RuntimeException(e);
        }
    }

    public String writeValue(Object value) {
        try {
            return jacksonObjectMapper.writeValueAsString(value);
        } catch (JsonProcessingException e) {
            throw new RuntimeException(e);
        }
    }
});

// Response to Object
HttpResponse<Book> bookResponse = Unirest.get("http://httpbin.org/books/1").asObject(Book.class);
Book bookObject = bookResponse.getBody();

HttpResponse<Author> authorResponse = Unirest.get("http://httpbin.org/books/{id}/author")
    .routeParam("id", bookObject.getId())
    .asObject(Author.class);

Author authorObject = authorResponse.getBody();

// Object to Json
HttpResponse<JsonNode> postResponse = Unirest.post("http://httpbin.org/authors/post")
        .header("accept", "application/json")
        .header("Content-Type", "application/json")
        .body(authorObject)
        .asJson();
```

**Route Parameters**
在RUL中添加动态参数，可以通过URL中添加占位符，然后再利用routeParam函数

```java
Unirest.get("http://httpbin.org/{method}")
  .routeParam("method", "get")
  .queryString("name", "Mark")
  .asJson();
```
这里的占位符会被get代替


**异步请求**
很多时候，需要使用asynchronous模式，使用异步回调

```java
Future<HttpResponse<JsonNode>> future = Unirest.post("http://httpbin.org/post")
					  .header("accept", "application/json")
					  .field("param1", "value1")
					  .field("param2", "value2")
					  .asJsonAsync(new Callback<JsonNode>() {
    public void failed(UnirestException e) {
        System.out.println("The request has failed");
    }

    public void completed(HttpResponse<JsonNode> response) {
         int code = response.getStatus();
         Map<String, String> headers = response.getHeaders();
         JsonNode body = response.getBody();
         InputStream rawBody = response.getRawBody();
    }

    public void cancelled() {
        System.out.println("The request has been cancelled");
    }
});
```

**文件上传**
使用java创建multipart请求很琐碎，下面代码简化这一个过程

```java
HttpResponse<JsonNode> jsonResponse = Unirest.post("http://httpbin.org/post")
  .header("accept", "application/json")
  .field("parameter", "value")
  .field("file", new File("/tmp/file"))
  .asJson();
```
**定制实体body**

```java
HttpResponse<JsonNode> jsonResponse = Unirest.post("http://httpbin.org/post")
  .header("accept", "application/json")
  .body("{\"parameter\":\"value\", \"foo\":\"bar\"}")
  .asJson();
```

**比特流body**

```java
final InputStream stream = new FileInputStream(new File(getClass().getResource("/image.jpg").toURI()));
final byte[] bytes = new byte[stream.available()];
stream.read(bytes);
stream.close();
final HttpResponse<JsonNode> jsonResponse = Unirest.post("http://httpbin.org/post")
  .field("name", "Mark")
  .field("file", bytes, "image.jpg")
  .asJson();
```

**InputStream body**

```java
HttpResponse<JsonNode> jsonResponse = Unirest.post("http://httpbin.org/post")
  .field("name", "Mark")
  .field("file", new FileInputStream(new File(getClass().getResource("/image.jpg").toURI())), ContentType.APPLICATION_OCTET_STREAM, "image.jpg")
  .asJson();
```

**Basic Authentication**
调用`basicAuth(username, password)`

```java
HttpResponse<JsonNode> response = 
	Unirest.get("http://httpbin.org/headers").basicAuth("username", "password").asJson();
```

## 三. Request & Response
Java版本的是构造这模式,通过下面这些方式获取HttpRequest 

```java
GetRequest request = Unirest.get(String url);
GetRequest request = Unirest.head(String url);
HttpRequestWithBody request = Unirest.post(String url);
HttpRequestWithBody request = Unirest.put(String url);
HttpRequestWithBody request = Unirest.patch(String url);
HttpRequestWithBody request = Unirest.options(String url);
HttpRequestWithBody request = Unirest.delete(String url);
```
收到反馈时，返回一个Object，object对于不同的语言应该有相同的keys 

```java
.getStatus() - HTTP Response Status Code (Example: 200)
.getStatusText() - HTTP Response Status Text (Example: "OK")
.getHeaders() - HTTP Response Headers
.getBody() - Parsed response body where applicable, for example JSON responses are parsed to Objects / Associative Arrays.
.getRawBody() - Un-parsed response body
```

## 四. 高级配置
可以设置一些高级的配置，调整Unirest
定制HTTP clients,可以修改HttpClient和HttpAsyncClient的实现，然后设置进Unirest中

```java
Unirest.setHttpClient(httpClient);
Unirest.setAsyncHttpClient(asyncHttpClient);
```
Timeouts以ms为单位
默认连接超时10000，socket超时60000，设置为0为关闭

```java
Unirest.setTimeouts(long connectionTimeout, long socketTimeout);
```
默认的Request Headers,每个request都会发出去

```java
Unirest.setDefaultHeader("Header1", "Value1");
Unirest.setDefaultHeader("Header2", "Value2");
```
清除默认的头

```java
Unirest.clearDefaultHeaders();
```
并发,可以设置并发级别，通过设置同步或者异步的client
默认pool中全部的连接限制，maxTotal设置为200，并且maxPerRoute目标host设置为20

```java
Unirest.setConcurrency(int maxTotal, int maxPerRoute);
```
代理

```java
Unirest.setProxy(new HttpHost("127.0.0.1", 8000));
```

**退出应用**
Unirest开启后台一个`event loop`,需要手动调用执行退出

```java
Unirest.shutdown();
```

python使用的例子
http://www.open-open.com/lib/view/open1415237017090.html
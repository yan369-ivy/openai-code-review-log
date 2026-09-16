本次 diff 主要为 SDK 增加了微信模板消息通知能力，并在测试中补充了手动调用入口。功能方向合理，但当前实现存在**严重安全风险**和若干健壮性、架构问题，建议修复后再合并。

---

## 一、严重问题（必须修复）

### 1. 敏感信息硬编码并提交到 Git
`WXAccessTokenUtils` 中直接写死了微信 `APPID` 和 `SECRET`：

```java
private static final String APPID = "wxd890da243fd44459";
private static final String SECRET = "434f03c26272a28a0e74c4448d07feb3";
```

`Message` 中也硬编码了真实 `touser` 和 `template_id`：

```java
private String touser = "o8ofB7cQZL7Kznncsecgqs-3MR7g";
private String template_id = "xyVK2F15YUunmpShfsYSfxe-kxzq-tqZbqn_iUHVKKk";
```

**风险**：AppSecret 泄露后，攻击者可获取 access_token、发送模板消息、调用其他微信接口。即使后续提交删除，Git 历史中仍然存在。

**建议**：
- 立即在微信公众平台重置 AppSecret。
- 所有敏感配置改为环境变量、配置中心或密钥管理服务，例如：
  ```java
  private static final String APP_ID = System.getenv("WX_APP_ID");
  private static final String APP_SECRET = System.getenv("WX_APP_SECRET");
  ```
- `touser`、`template_id` 也应外部化，不要写死在实体类默认值中。

### 2. access_token 被打印到日志
```java
System.out.println(accessToken);
System.out.println("Response: " + response.toString());
```
access_token 等同于临时密钥，打印到控制台或日志后可能被收集、泄露。

**建议**：删除这些打印，或仅打印脱敏后的状态信息（如“token 获取成功，有效期 7200s”）。

### 3. access_token 未缓存，频繁请求微信接口
`WXAccessTokenUtils.getAccessToken()` 每次都重新请求微信接口。微信对 access_token 获取有频率限制，且返回的 `expires_in` 未被使用。

**建议**：
- 增加内存缓存，过期前刷新。
- 考虑并发场景，使用 `volatile` + 双重检查或 `AtomicReference`。
- 示例：
  ```java
  private static volatile String cachedToken;
  private static volatile long expireAt;
  ```

---

## 二、主要问题（建议修复）

### 1. 未处理 accessToken 为 null
`pushMessage` 中：
```java
String accessToken = WXAccessTokenUtils.getAccessToken();
...
String url = String.format("...access_token=%s", accessToken);
```
如果获取失败，`accessToken` 为 `null`，URL 会变成 `access_token=null`，请求失败但错误不明确。

**建议**：获取失败时抛异常或提前返回，并记录明确日志。

### 2. HTTP 请求缺少超时与错误响应处理
`OpenAiCodeReview.sendPostRequest` 和 `ApiTest.sendPostRequest` 都使用原生 `HttpURLConnection`，但：
- 未设置 `connectTimeout` / `readTimeout`，可能长时间阻塞。
- 主代码未检查 `responseCode`，非 2xx 时 `getInputStream()` 会抛异常，无法读取微信返回的 `errcode` / `errmsg`。
- `ApiTest` 虽处理了 `errorStream`，但未判断 `responseStream` 是否为 null，可能 NPE。

**建议**：
- 设置超时，例如：
  ```java
  conn.setConnectTimeout(5000);
  conn.setReadTimeout(5000);
  ```
- 统一处理成功流与错误流，并解析微信响应中的 `errcode`。
- 推荐使用 OkHttp、Apache HttpClient 或 Java 11 `HttpClient`，减少样板代码。

### 3. 资源未正确关闭
`WXAccessTokenUtils` 中：
```java
BufferedReader in = new BufferedReader(new InputStreamReader(connection.getInputStream()));
...
in.close();
```
未使用 try-with-resources，异常时可能泄漏。`HttpURLConnection` 也未调用 `disconnect()`。

**建议**：
```java
try (BufferedReader in = new BufferedReader(...)) {
    ...
} finally {
    connection.disconnect();
}
```

### 4. 异常处理过于宽泛且静默
```java
} catch (Exception e) {
    e.printStackTrace();
}
```
捕获所有异常并只打印堆栈，调用方无法感知失败，容易造成“消息没发出去但流程认为成功”。

**建议**：
- 捕获具体异常（如 `IOException`、`InterruptedException`）。
- 使用日志框架记录上下文。
- 根据业务决定是抛出运行时异常还是返回明确结果。

### 5. 命名不符合 Java 规范
`Token` 类字段与 getter：
```java
private String access_token;
private Integer expires_in;
public String getAccess_token()
```
不符合 Java 驼峰命名。

**建议**：
```java
public static class Token {
    @JSONField(name = "access_token")
    private String accessToken;
    @JSONField(name = "expires_in")
    private Integer expiresIn;
    // getAccessToken / setAccessToken
}
```

### 6. 代码重复
`sendPostRequest` 在 `OpenAiCodeReview` 和 `ApiTest` 中几乎完全重复，且测试版更完善。这种重复会导致后续修复不同步。

**建议**：抽取到独立工具类，例如 `HttpUtils.postJson(url, jsonBody)`，主代码与测试共用。

---

## 三、架构与设计建议

### 1. 职责过重，建议抽离通知服务
`OpenAiCodeReview` 入口类现在同时负责：
- 获取 GitHub token
- 写日志
- 获取微信 access_token
- 发送模板消息

这违反了单一职责原则。建议抽象为：

```java
public interface NotificationService {
    void sendReviewResult(String logUrl);
}

public class WeChatNotificationService implements NotificationService {
    // 封装 accessToken 缓存、模板消息发送、错误处理
}
```

入口类只依赖接口，便于替换通知渠道（企业微信、钉钉、飞书等）。

### 2. 配置外部化
微信相关配置应集中管理，例如：

```yaml
wechat:
  app-id: ${WX_APP_ID}
  app-secret: ${WX_APP_SECRET}
  template-id: ${WX_TEMPLATE_ID}
  touser: ${WX_TOUSER}
```

如果项目不是 Spring 项目，也建议提供 `WeChatConfig` 类从环境变量或配置文件读取。

### 3. 使用现代 HTTP 客户端
原生 `HttpURLConnection` 代码冗长，连接池、超时、错误流处理都不方便。推荐：
- OkHttp
- Apache HttpClient 5
- Java 11+ `java.net.http.HttpClient`

### 4. 日志规范
SDK 中不建议直接使用 `System.out.println`。可以：
- 使用 `java.util.logging` 作为最小依赖；
- 或提供日志抽象，由使用方接入 SLF4J。

---

## 四、测试评审

### 优点
- 对真实外部调用加了 `@Ignore` 并注明原因，避免 CI 误执行。
- 补充了 `test_wx` 手动验证入口。

### 问题
1. `ApiTest.sendPostRequest` 与主代码重复，且未来容易不一致。
2. 测试中硬编码真实 OpenID、URL、项目名。
3. 没有断言，只是打印结果。
4. 仍然依赖真实微信接口，无法自动化验证错误分支。

**建议**：
- 使用 WireMock / MockWebServer 模拟微信 API。
- 对 `WXAccessTokenUtils` 做单元测试，验证：
  - 正常返回 token；
  - 微信返回 `errcode` 时抛出异常；
  - 缓存生效，不重复请求。
- 手动测试保留 `@Ignore` 可以，但配置应来自环境变量。

---

## 五、具体修改示例

### 1. 移除硬编码并增加判空
```java
private static void pushMessage(String logUrl) {
    String accessToken = WXAccessTokenUtils.getAccessToken();
    if (accessToken == null || accessToken.isEmpty()) {
        throw new IllegalStateException("获取微信 access_token 失败");
    }

    Message message = new Message();
    message.put("project", "big-market");
    message.put("review", logUrl);
    message.setUrl(logUrl);

    String url = String.format(
        "https://api.weixin.qq.com/cgi-bin/message/template/send?access_token=%s",
        accessToken
    );
    String response = sendPostRequest(url, JSON.toJSONString(message));
    // 解析 response，检查 errcode
}
```

### 2. 统一 HTTP 工具并处理错误流
```java
private static String sendPostRequest(String urlString, String jsonBody) throws IOException {
    HttpURLConnection conn = (HttpURLConnection) new URL(urlString).openConnection();
    conn.setRequestMethod("POST");
    conn.setRequestProperty("Content-Type", "application/json;charset=UTF-8");
    conn.setConnectTimeout(5000);
    conn.setReadTimeout(5000);
    conn.setDoOutput(true);

    try (OutputStream os = conn.getOutputStream()) {
        os.write(jsonBody.getBytes(StandardCharsets.UTF_8));
    }

    int code = conn.getResponseCode();
    InputStream stream = (code >= 200 && code < 300)
        ? conn.getInputStream()
        : conn.getErrorStream();

    if (stream == null) {
        throw new IOException("HTTP " + code + " 且无响应体");
    }

    try (Scanner scanner = new Scanner(stream, StandardCharsets.UTF_8.name())) {
        scanner.useDelimiter("\\A");
        return scanner.hasNext() ? scanner.next() : "";
    } finally {
        conn.disconnect();
    }
}
```

### 3. access_token 缓存示意
```java
private static volatile String cachedToken;
private static volatile long expireAt;

public static String getAccessToken() {
    long now = System.currentTimeMillis();
    if (cachedToken != null && now < expireAt) {
        return cachedToken;
    }
    synchronized (WXAccessTokenUtils.class) {
        if (cachedToken != null && now < expireAt) {
            return cachedToken;
        }
        Token token = fetchToken();
        cachedToken = token.getAccessToken();
        expireAt = now + (token.getExpiresIn() - 300) * 1000L;
        return cachedToken;
    }
}
```

---

## 六、结论

当前实现可以完成“获取微信 token 并发送模板消息”的基本功能，但存在**高危安全泄露、token 未缓存、错误处理缺失、代码重复、职责不清**等问题。

**合并前必须处理**：
1. 移除并重置所有硬编码的 AppID / AppSecret / OpenID / template_id。
2. 删除 access_token 日志打印。
3. 增加 access_token 缓存与失败判空。
4. 完善 HTTP 超时、错误响应解析和资源关闭。

**建议后续优化**：
- 抽离 `WeChatNotificationService`，入口类通过接口调用。
- 配置外部化，敏感信息走环境变量或配置中心。
- 抽取 HTTP 工具类，消除重复代码。
- 测试使用 Mock Server，减少对外部真实接口的依赖。

整体评价：功能原型可用，但生产就绪度不足，建议修复上述问题后再合入主分支。

### 환경
- JDK 11
- httpclient5-fluent:5.1.3
### 문제 상황
- Apache HttpClient5를 사용해 API 요청 시 `org.apache.hc.core5.http.NoHttpResponseException: The target server failed to respond`가 불규칙적으로 발생한다
### 해결 과정
#### NoHttpResponseException 발생 이유
- HTTP 1.1에서는 커넥션 비용을 줄이기 위해서 `Keep-Alive` 기능을 사용한다
- 어떠한 이유로(요청이 많다거나) 서버에서 클라이언트의 커넥션을 Shut Down 했고, 클라이언트에서는 해당 커넥션을 재활용 하는 경우 `NoHttpResponseException`가 발생할 수 있다
- MockServer 라이브러리를 사용하여 해당 상황을 재현 해본다

```Java
public class ApiTest {  
    private static ClientAndServer mockServer;  
  
    @BeforeAll  
    static void beforeAll() {  
       mockServer = ClientAndServer.startClientAndServer(1080);  
    }  
  
    @AfterAll  
    static void afterAll() {  
       mockServer.stop();  
    }  
  
    void test() throws IOException {  
       // given  
       CloseableHttpClient closeableHttpClient = HttpClientBuilder.create()  
          .setConnectionManager(  
             PoolingHttpClientConnectionManagerBuilder.create()  
                .useSystemProperties()  
                .setMaxConnPerRoute(1)  
                .setMaxConnTotal(1)  
                .setValidateAfterInactivity(TimeValue.ofSeconds(1L))  
                .setConnectionTimeToLive(TimeValue.ofMinutes(1L))  
                .setDefaultSocketConfig(  
                   SocketConfig.custom()  
                      .setSoTimeout(Timeout.ofSeconds(1L))  
                      .build()  
                )  
                .build()  
          )  
          .useSystemProperties()  
          .evictExpiredConnections()  
          .evictIdleConnections(TimeValue.ofMinutes(1L))  
          .disableAutomaticRetries()
          .build();  
  
       Executor executor = Executor.newInstance(closeableHttpClient);  
  
       mockServer.when(  
          request("http://localhost:1080")  
             .withPath("/api/v1/test")  
       ).respond(HttpResponse.response()  
             .withConnectionOptions(ConnectionOptions.connectionOptions().withCloseSocket(true))  
          .withStatusCode(200)  
       );  
  
       // when  
       executor.execute(Request.create(Method.GET.name(), "http://localhost:1080/api/v1/test")  
          .bodyStream(InputStream.nullInputStream()));  
  
	Assertions.assertThrows(NoHttpResponseException.class, () -> executor.execute(Request.create(Method.GET.name(), "http://localhost:1080/api/v1/test")  
	    .bodyStream(InputStream.nullInputStream())));
    }  
}
```

- `withConnectionOptions(ConnectionOptions.connectionOptions().withCloseSocket(true))` 설정을 주어 커넥션을 Shut Down 시킨다
- 두 번째 요청에서 클라이언트는 Shut Down된 커넥션을 재활용해 `NoHttpResponseException`이 발생한다
	- 커넥션을 재활용 하지 않을 경우에는 해당 예외가 발생하지 않을 수 있다

#### 해결1 - 재시도 로직 실행
- Apache HttpClient5는 기본적으로 요청에 예외가 발생할 경우 재시도를 한다
- 재시도 로직은 `HttpRequestRetryExec`에 구현되어 있으며, 기본 재시도 전략은 `DefaultHttpRequestRetryStrategy`이다

```Java
    void test() throws IOException {  
       // given  
       CloseableHttpClient closeableHttpClient = HttpClientBuilder.create()  
          .setConnectionManager(  
             PoolingHttpClientConnectionManagerBuilder.create()  
                .useSystemProperties()  
                .setMaxConnPerRoute(1)  
                .setMaxConnTotal(1)  
                .setValidateAfterInactivity(TimeValue.ofSeconds(1L))  
                .setConnectionTimeToLive(TimeValue.ofMinutes(1L))  
                .setDefaultSocketConfig(  
                   SocketConfig.custom()  
                      .setSoTimeout(Timeout.ofSeconds(1L))  
                      .build()  
                )  
                .build()  
          )  
          .useSystemProperties()  
          .evictExpiredConnections()  
          .evictIdleConnections(TimeValue.ofMinutes(1L))  
          .build();  

    }  
```

- `disableAutomaticRetries()` 설정을 지운다

#### 해결2 - body 타입 변경
- `disableAutomaticRetries()` 설정을 지워 재시도를 해도 계속 `NoHttpResponseException`이 발생하는데, 그 이유는 `Request`의 Body 타입이 스트림이기 떄문이다
- `Request` 인스턴스는 한 번 생성이되고 스트림은 한 번 소진하면 되돌릴 수 없다

```java
for (int execCount = 1;; execCount++) {  
    final ClassicHttpResponse response;  
    try {  
         response = chain.proceed(currentRequest, scope);  
    } catch (final IOException ex) {  
        if (scope.execRuntime.isExecutionAborted()) {  
            throw new RequestFailedException("Request aborted");  
        }  
        final HttpEntity requestEntity = request.getEntity();  
        if (requestEntity != null && !requestEntity.isRepeatable()) {  
            if (LOG.isDebugEnabled()) {  
                LOG.debug("{}: cannot retry non-repeatable request", exchangeId);  
            }  
            throw ex;  
        }  
        if (retryStrategy.retryRequest(request, ex, execCount, context)) {  
	        ...
 
        } else {  
            ...
        }  
    }
```

- `HttpRequestRetryExec`의 내부 로직에는 `IOException`이 발생할 경우 `requestEntity`가 반복 가능(isRepeatable)하면 재시도를 수행한다
- `Request`를 생성할 때 `bodyStream`을 호출하면 `InputStreamEntity`를 생성한다. `InputStreamEntity`은 반복 불가능하다

```Java
public class InputStreamEntity extends AbstractHttpEntity {
	...
	@Override  
	public final boolean isRepeatable() {  
	    return false;  
	}
}
```

- 따라서 Body의 타입을 반복가능한 `StringEntity` 혹은 `ByteArrayEntity`로 변경해주어야 한다

```Java
public class StringEntity extends AbstractHttpEntity {
	...
	@Override  
	public final boolean isRepeatable() {  
	    return true;  
	}
}
```

```Java
public class ByteArrayEntity extends AbstractHttpEntity {
	...
	@Override  
	public final boolean isRepeatable() {  
	    return true;  
	}
}
```


### 환경
- Kotlin, Spring, Spring Data Redis
### 문제 상황
- Spring Boot의 기본 설정에 따르면 레디스 key를 직렬화 할 때 예상치 못한 값이 추가된다

```text
127.0.0.1:6379> KEYS *
1) "\xac\xed\x00\x05t\x00\x0crunner-user0"
2) "\xac\xed\x00\x05t\x00\x1erunning-user0:2024-01-02T12:00"
3) "\xac\xed\x00\x05t\x00\x1erunning-user0:2024-01-01T12:00"
```
- 예상치 못한 바이트 값이 추가되면 레디스에서 데이터를 검색 및 조작하기 어려워진다

### 해결 과정
- Spring Data Redis를 사용할 경우 기본 직렬화 클래스로 `JdkSerializationRedisSerialzer`를 사용한다
- `JdkSerializationRedisSerialzer`는 내부적으로 직렬화를 위해 자바의  `ObjectOutputStream`을 사용한다
- `ObjectOutputStream`은 직렬화 할 때 특정한 값을 헤더에 추가한다

```java
private void writeString(String str, boolean unshared) throws IOException {  
    handles.assign(unshared ? null : str);  
    long utflen = bout.getUTFLength(str);  
    if (utflen <= 0xFFFF) {  
        bout.writeByte(TC_STRING);  
        bout.writeUTF(str, utflen);  
    } else {  
        bout.writeByte(TC_LONGSTRING);  
        bout.writeLongUTF(str, utflen);  
    }  
}
```

```java
void writeUTF(String s, long utflen) throws IOException {  
    if (utflen > 0xFFFFL) {  
        throw new UTFDataFormatException();  
    }  
    writeShort((int) utflen);  
    if (utflen == (long) s.length()) {  
        writeBytes(s);  
    } else {  
        writeUTFBody(s);  
    }  
}
```

### 결론
- 레디스 키의 직렬화 클래스를 바꿔준다
- 예제에서는 `StringRedisSerialzer`를 사용했다
```kotlin
@Configuration  
class RedisConfiguration {  
    @Bean  
    fun redisTemplate(connectionFactory: RedisConnectionFactory): RedisTemplate<String, Runner> {  
       val template = RedisTemplate<String, Any>()  
       template.connectionFactory = connectionFactory  
  
       val objectMapper = jacksonObjectMapper().apply {  
          registerModule(JavaTimeModule())  
          disable(com.fasterxml.jackson.databind.SerializationFeature.WRITE_DATES_AS_TIMESTAMPS)  
       }  
  
       template.keySerializer = StringRedisSerializer()  // 직렬화 클래스 변경
       template.valueSerializer = Jackson2JsonRedisSerializer(objectMapper, Any::class.java)  
       return template  
    }
```

```text
127.0.0.1:6379> KEYS *
1) "running-user0:2024-01-02T12:00"
2) "runner-user0"
```


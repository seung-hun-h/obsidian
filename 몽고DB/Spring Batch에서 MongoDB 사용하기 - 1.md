## 0. 사용 모델
###### Person
```java 
public record Person(String name, int age, List<Occupation> occupations) {  
}
```
###### Occupation
```java
public record Occupation(String name, String company, int salary, LocalDateTime joinTime) {  
}
```

## 1. MongoDB Driver 의존성 주입
JDBC Driver가 필요하듯 MongoDB Driver가 필요하다
###### build.gradle.kts
```kotlin
dependencies {  
    implementation ("org.mongodb:mongodb-driver-sync:5.2.1")  
}
```

## 2. MongoClient, MongoDatabase 의 사용
MongoDB와 통신하고 데이터를 다룰 수 있는 클래스들을 빈으로 등록한다
- **MongoClient**: MongoDB 서버와 연결을 관리하는 클라이언트
- **MongoDatabase**: MongoClient를 통해 지정된 데이터베이스에 접근할 수 있도록 도와주는 객체
###### application.yml
```yml
mongo:  
  uri: mongodb://localhost:27017/javamongo  
  database: javamongo
```
###### MongoConfigurationV1
```java
@Configuration  
public class MongoConfigurationV1 {  
    private final String uri;  
    private final String database;  
  
    public MongoConfigurationV1(  
       @Value("${mongo.uri}")  
       String uri,  
       @Value("${mongo.database}")  
       String database) {  
       this.uri = uri;  
       this.database = database;  
    }  
  
    @Bean  
    public MongoClient mongoClient() {  
	    // 커스텀 Codec을 등록하여 Person과 Occupation의 직렬화/역직렬화 설정
       CodecRegistry customRegistry = CodecRegistries.fromRegistries(
          CodecRegistries.fromCodecs(new OccupationCodec(), new PersonCodec(new OccupationCodec())),  
          MongoClientSettings.getDefaultCodecRegistry()  
       );  
		// 추가 옵션 적용하여 MongoDB 연결 설정  
       MongoClientSettings settings = MongoClientSettings.builder()
          .codecRegistry(customRegistry)  
          .applyConnectionString(new ConnectionString(uri))  
          .build();  
  
       return MongoClients.create(settings);  
    }  
  
    @Bean  
    public MongoDatabase mongoDatabase(MongoClient mongoClient) {  
       return mongoClient.getDatabase(database);  
    }  
}
```
###### PersonCodec
```java
public class PersonCodec implements Codec<Person> {  
    private final Codec<Occupation> occupationCodec;  
  
    public PersonCodec(Codec<Occupation> occupationCodec) {  
       this.occupationCodec = occupationCodec;  
    }  
  
    @Override  
    public void encode(BsonWriter writer, Person person, EncoderContext encoderContext) {  
       writer.writeStartDocument();  
       writer.writeString("name", person.name());  
       writer.writeInt32("age", person.age());  
  
       // occupations 배열 작성  
       writer.writeName("occupations");  
       writer.writeStartArray();  
       if (person.occupations() != null) {  
          for (Occupation occupation : person.occupations()) {  
             occupationCodec.encode(writer, occupation, encoderContext);  
          }  
       }  
       writer.writeEndArray();  
       writer.writeEndDocument();  
    }  
  
    @Override  
    public Person decode(BsonReader reader, DecoderContext decoderContext) {  
       reader.readStartDocument();  
       String name = null;  
       int age = 0;  
       List<Occupation> occupations = new ArrayList<>();  
  
       while (reader.readBsonType() != BsonType.END_OF_DOCUMENT) {  
          String fieldName = reader.readName();  
          switch (fieldName) {  
             case "name":  
                name = reader.readString();  
                break;  
             case "age":  
                age = reader.readInt32();  
                break;  
             case "occupations":  
                reader.readStartArray();  
                while (reader.readBsonType() != BsonType.END_OF_DOCUMENT) {  
                   Occupation occupation = occupationCodec.decode(reader, decoderContext);  
                   occupations.add(occupation);  
                }  
                reader.readEndArray();  
                break;  
             default:  
                reader.skipValue();  
                break;  
          }  
       }  
       reader.readEndDocument();  
       return new Person(name, age, occupations);  
    }  
  
    @Override  
    public Class<Person> getEncoderClass() {  
       return Person.class;  
    }  
}
```

###### OccupationCodec
```java
public class OccupationCodec implements Codec<Occupation> {  
    @Override  
    public void encode(BsonWriter writer, Occupation occupation, EncoderContext encoderContext) {  
       writer.writeStartDocument();  
       writer.writeString("name", occupation.name());  
       writer.writeString("company", occupation.company());  
       writer.writeInt32("salary", occupation.salary());  
  
       // LocalDateTime을 시스템 타임존 기준 밀리초로 변환하여 저장  
       long millis = occupation.joinTime()  
          .atZone(ZoneId.systemDefault())  
          .toInstant()  
          .toEpochMilli();  
       writer.writeDateTime("joinTime", millis);  
       writer.writeEndDocument();  
    }  
  
    @Override  
    public Occupation decode(BsonReader reader, DecoderContext decoderContext) {  
       reader.readStartDocument();  
       String name = null;  
       String company = null;  
       int salary = 0;  
       LocalDateTime joinTime = null;  
  
       while (reader.readBsonType() != BsonType.END_OF_DOCUMENT) {  
          String fieldName = reader.readName();  
          switch (fieldName) {  
             case "name":  
                name = reader.readString();  
                break;  
             case "company":  
                company = reader.readString();  
                break;  
             case "salary":  
                salary = reader.readInt32();  
                break;  
             case "joinTime":  
                long millis = reader.readDateTime();  
                joinTime = LocalDateTime.ofInstant(Instant.ofEpochMilli(millis), ZoneId.systemDefault());  
                break;  
             default:  
                reader.skipValue();  
                break;  
          }  
       }  
       reader.readEndDocument();  
       return new Occupation(name, company, salary, joinTime);  
    }  
  
    @Override  
    public Class<Occupation> getEncoderClass() {  
       return Occupation.class;  
    }  
}
```

## 3. 배치 잡에서의 사용
MongoDB를 활용한 Spring Batch 잡 예제로, 삽입 → 조회 → 삭제 작업을 수행한다
###### MongoTestJobV1Configuration
```java
@Configuration  
public class MongoTestJobV1Configuration {  
    private final JobRepository jobRepository;  
    private final PlatformTransactionManager platformTransactionManager;  
    private final MongoDatabase mongoDatabase;  
  
    public MongoTestJobV1Configuration(
	    JobRepository jobRepository, 
	    PlatformTransactionManager platformTransactionManager, 
	    MongoDatabase mongoDatabase) {  
       this.jobRepository = jobRepository;  
       this.platformTransactionManager = platformTransactionManager;  
       this.mongoDatabase = mongoDatabase;  
    }  
  
    @Bean  
    public Job mongoTestJobV1() {  
       return new JobBuilder("mongoTestJobV1", jobRepository)  
          .start(new StepBuilder("mongoTestStepV1", jobRepository)  
             .tasklet(  
                (contribution, chunkContext) -> {  
                   System.out.println("===mongoTestJobV1===");  
  
                   Person person = new Person("seunghun", 30, List.of(new Occupation("developer", "google", 10, LocalDateTime.now())));  
  
                   MongoCollection<Person> peopleCollection = mongoDatabase.getCollection("people", Person.class);  
  
                   peopleCollection.insertOne(person);  
                   System.out.printf("%s inserted\n", person);  
  
                   peopleCollection.find(eq("name", "seunghun")).forEach(p -> {  
                      System.out.printf("%s found\n", p);  
                   });  
  
                   peopleCollection.deleteMany(eq("name", "seunghun"));  
  
                   return RepeatStatus.FINISHED;  
                }, platformTransactionManager  
             )  
             .build())  
          .incrementer(new RunIdIncrementer())  
          .build();  
    }  
}
```

## 4. V1의 문제점
- 컬렉션과 클래스 타입을 매번 명시해야 한다
	- `MongoCollection<Person> peopleCollection = mongoDatabase.getCollection("people", Person.class);`
- 커스텀 클래스의 경우 `Codec`을 구현하여 등록해야 한다

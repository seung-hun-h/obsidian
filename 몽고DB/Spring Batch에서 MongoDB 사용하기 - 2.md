## 0. SpringData MongoDB 사용하기
Spring Boot를 사용하면 Spring Data MongoDB가 자동 구성되며, MongoDB에 쉽게 접근할 수 있다
###### build.gradle.kts
```kotlin
dependencies {  
    implementation("org.springframework.boot:spring-boot-starter-data-mongodb")  
}
```
###### application.yml
```yml
spring:  
  data:  
    mongodb:  
      uri: mongodb://localhost:27017/javamongo  
      database: javamongo  
```

## 1. 사용 모델
###### PersonV2
```java
// 저장할 컬렉션 지정
@Document(collection = "people")
public class PersonV2 { 
	@Id // 식별자(_id) 지정
    private String id;  
    private String name;  
    private int age;  
    private List<OccupationV2> occupations;  
  
    public PersonV2(String name, int age, List<OccupationV2> occupations) {  
       this.name = name;  
       this.age = age;  
       this.occupations = occupations;  
    }  
  
    public String getId() {  
       return id;  
    }  
  
    public String getName() {  
       return name;  
    }  
  
    public int getAge() {  
       return age;  
    }  
  
    public List<OccupationV2> getOccupations() {  
       return occupations;  
    }  
  
    @Override  
    public String toString() {  
       return "Person{" +  
          "name='" + name + '\'' +  
          ", age=" + age +  
          ", occupations=" + occupations +  
          '}';  
    }  
}
```
###### OccupationV2
```java
public class OccupationV2 {  
    private String name;  
    private String company;  
    private int salary;  
    private LocalDateTime joinTime;  
  
    public OccupationV2(String name, String company, int salary, LocalDateTime joinTime) {  
       this.name = name;  
       this.company = company;  
       this.salary = salary;  
       this.joinTime = joinTime;  
    }  
  
    public String getName() {  
       return name;  
    }  
  
    public String getCompany() {  
       return company;  
    }  
  
    public int getSalary() {  
       return salary;  
    }  
  
    public LocalDateTime getJoinTime() {  
       return joinTime;  
    }  
  
    @Override  
    public String toString() {  
       return "Occupation{" +  
          "name='" + name + '\'' +  
          ", company='" + company + '\'' +  
          ", salary=" + salary +  
          ", joinTime=" + joinTime +  
          '}';  
    }  
}
```

## 2. MongoOperations, MongoTemplate
Spring Boot는 자동으로 `MongoClient`, `MongoOperations`, `MongoTemplate`을 빈으로 등록한다. 이를 통해 복잡한 설정 없이 MongoDB 쿼리를 작성할 수 있다
###### MongoAutoConfiguration
```java
@AutoConfiguration  
@ConditionalOnClass(MongoClient.class)  
@EnableConfigurationProperties(MongoProperties.class)  
@ConditionalOnMissingBean(type = "org.springframework.data.mongodb.MongoDatabaseFactory")  
public class MongoAutoConfiguration {  
  
    @Bean  
    @ConditionalOnMissingBean(MongoConnectionDetails.class)  
    PropertiesMongoConnectionDetails mongoConnectionDetails(MongoProperties properties) {  
       return new PropertiesMongoConnectionDetails(properties);  
    }  
  
    @Bean  
    @ConditionalOnMissingBean    
    public MongoClient mongo(ObjectProvider<MongoClientSettingsBuilderCustomizer> builderCustomizers,  
          MongoClientSettings settings) {  
       return new MongoClientFactory(builderCustomizers.orderedStream().toList()).createMongoClient(settings);  
    }  
  
    @Configuration(proxyBeanMethods = false)  
    @ConditionalOnMissingBean(MongoClientSettings.class)  
    static class MongoClientSettingsConfiguration {  
  
       @Bean  
       MongoClientSettings mongoClientSettings() {  
          return MongoClientSettings.builder().build();  
       }  
  
       @Bean  
       StandardMongoClientSettingsBuilderCustomizer standardMongoSettingsCustomizer(MongoProperties properties,  
             MongoConnectionDetails connectionDetails, ObjectProvider<SslBundles> sslBundles) {  
          return new StandardMongoClientSettingsBuilderCustomizer(connectionDetails.getConnectionString(),  
                properties.getUuidRepresentation(), properties.getSsl(), sslBundles.getIfAvailable());  
       }  
  
    }  
  
}
```
###### MongoDataAutoConfiguration
```java
@AutoConfiguration(after = MongoAutoConfiguration.class)  
@ConditionalOnClass({ MongoClient.class, MongoTemplate.class })  
@EnableConfigurationProperties(MongoProperties.class)  
@Import({ MongoDataConfiguration.class, MongoDatabaseFactoryConfiguration.class,  
       MongoDatabaseFactoryDependentConfiguration.class })  
public class MongoDataAutoConfiguration {  
  
    @Bean  
    @ConditionalOnMissingBean(MongoConnectionDetails.class)  
    PropertiesMongoConnectionDetails mongoConnectionDetails(MongoProperties properties) {  
       return new PropertiesMongoConnectionDetails(properties);  
    }  
  
}
```
###### MongoDataAutoConfiguration
```java
@Configuration(proxyBeanMethods = false)  
@ConditionalOnBean(MongoDatabaseFactory.class)  
class MongoDatabaseFactoryDependentConfiguration {  
  
    @Bean  
    @ConditionalOnMissingBean(MongoOperations.class)  
    MongoTemplate mongoTemplate(MongoDatabaseFactory factory, MongoConverter converter) {  
       return new MongoTemplate(factory, converter);  
    }
	...
}
```

- `MongoOperations`는 몽고DB에 손쉽게 쿼리할 수 있는 다양한 메서드를 제공한다
###### MongoOperations
```java
public interface MongoOperations extends FluentMongoOperations {
	<T> T save(T objectToSave);
	
	@Nullable  
	<T> T findOne(Query query, Class<T> entityClass);
	
	DeleteResult remove(Object object);
	
	long count(Query query, Class<?> entityClass);
	...
}
```

## 3. 배치 잡에서 사용
```java 
@Configuration  
public class MongoTestJobV2Configuration {  
    private final JobRepository jobRepository;  
    private final PlatformTransactionManager platformTransactionManager;  
    private final MongoOperations mongoOperations;  
  
    public MongoTestJobV2Configuration(
	    JobRepository jobRepository, 
	    PlatformTransactionManager platformTransactionManager, 
	    MongoOperations mongoOperations) {  
       this.jobRepository = jobRepository;  
       this.platformTransactionManager = platformTransactionManager;  
       this.mongoOperations = mongoOperations;  
    }  
  
    @Bean  
    public Job mongoTestJobV2() {  
       return new JobBuilder("mongoTestJobV2", jobRepository)  
          .start(new StepBuilder("mongoTestStepV2", jobRepository)  
             .tasklet(  
                (contribution, chunkContext) -> {  
                   System.out.println("===mongoTestJobV2===");  
  
                   PersonV2 person = new PersonV2("seunghun", 30, List.of(new OccupationV2("developer", "google", 10, LocalDateTime.now())));  
  
                   mongoOperations.insert(person);
                   System.out.printf("%s inserted\n", person);  
  
                   mongoOperations.find(query(Criteria.where("name").is("seunghun")), PersonV2.class).forEach(p -> {  
                      System.out.printf("%s found\n", p);  
                   }); // Filters와 Criteria를 함께 사용하고 있다
  
                   mongoOperations.remove(query(Criteria.where("name").is("seunghun")), PersonV2.class);  
  
                   return RepeatStatus.FINISHED;  
                }, platformTransactionManager  
             )  
             .build())  
          .incrementer(new RunIdIncrementer())  
          .build();  
    }  
}
```
### Criteria의 이점
- `Criteria`는 fluent 스타일로 쿼리 작성이 가능해 `Filter`에 비해 가독성이 좋으며, SpringData MongoDB와 통합이 잘된다는 장점이 있다
- 동적 쿼리를 구성하는 방식은 크게 다르지 않다
#### 가독성
**Filters**
```java
Filters.and( Filters.gte("age", 30), Filters.eq("name", "John") );
```
**Criteria**
```java
query(Criteria.where("age").gte(30).and("name").is("John"));
```
#### SpringData 통합
**Filters**
```java
// MongoDB Java Driver의 Filters API를 사용한 동적 쿼리 예제
List<Bson> filters = new ArrayList<>();
if (name != null && !name.isEmpty()) {
    filters.add(Filters.eq("name", name));
}
if (minAge != null) {
    filters.add(Filters.gte("age", minAge));
}
if (city != null && !city.isEmpty()) {
    filters.add(Filters.eq("address.city", city));
}

Bson filter = filters.isEmpty() ? new Document() : Filters.and(filters);

List<User> results = collection.find(filter).into(new ArrayList<>());
```

**Criteria**
```java
// Spring Data MongoDB의 Criteria API를 사용한 동적 쿼리 예제
Query query = new Query();
List<Criteria> criteriaList = new ArrayList<>();

if (name != null && !name.isEmpty()) {
    criteriaList.add(Criteria.where("name").is(name));
}
if (minAge != null) {
    criteriaList.add(Criteria.where("age").gte(minAge));
}
if (city != null && !city.isEmpty()) {
    criteriaList.add(Criteria.where("address.city").is(city));
}

if (!criteriaList.isEmpty()) {
    query.addCriteria(new Criteria().andOperator(criteriaList.toArray(new Criteria[0])));
}

// 실행 (MongoTemplate과 도메인 객체 사용)
List<User> users = mongoTemplate.find(query, User.class);
```

##  4. V1의 문제점 해결
- 컬렉션과 클래스 타입을 매번 명시해야 한다
	- `@Document`를 선언해서 컬렉션을 매번 명시하지 않아도된다
	- 타입은 그래도 명시해야할 때가 있다
- 커스텀 클래스의 경우 `Codec`을 구현하여 등록해야 한다
	- `MappingMongoConverter`를 통해 `Codec`을 등록하지 않아도 도메인 클래스로 자동 변환이 된다
	- 
## 5. V2의 문제점
- 타입을 명시해야 할 때가 있다
- 간단한 쿼리를 작성할 때도 `Criteria`를 사용해야 해서 번거롭다


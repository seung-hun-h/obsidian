## 0. 사용 모델
###### PersonV3
```java
@Document(collection = "people")  
public class PersonV3 {  
    @Id  
    private ObjectId id;  
    private String name;  
    private int age;  
    private List<OccupationV3> occupations;  
  
    public PersonV3(String name, int age, List<OccupationV3> occupations) {  
       this.name = name;  
       this.age = age;  
       this.occupations = occupations;  
    }  
  
    public ObjectId getId() {  
       return id;  
    }  
    public String getName() {  
       return name;  
    }  
  
    public int getAge() {  
       return age;  
    }  
  
    public List<OccupationV3> getOccupations() {  
       return occupations;  
    }  
}
```
###### OccupationV3
```java
public class OccupationV3 {  
    private String name;  
    private String company;  
    private int salary;  
    private LocalDateTime joinTime;  
  
    public OccupationV3(String name, String company, int salary, LocalDateTime joinTime) {  
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
}
```
## 1. Repository 사용하기
- SpringData에서 제공하는 `MongoRepository`를 사용하면 간단한 쿼리는 직접 작성하지 않아도 된다

###### PersonV3Repository
```java
public interface PersonV3Repository extends MongoRepository<PersonV3, ObjectId> {  
    List<PersonV3> findByName(String name);  
  
    void deleteAllByName(String name);  
}
```

## 2. 배치 잡에서 사용하기 
- `MongoOperations`를 직접 사용할 때와는 다르게 간단한 쿼리의 경우 Query Method를 사용할 수 있다
###### MongoTestJobV3Configuration
```java
@Configuration  
public class MongoTestJobV3Configuration {  
    private final JobRepository jobRepository;  
    private final PlatformTransactionManager platformTransactionManager;  
    private final PersonV3Repository personV3Repository;  
  
    public MongoTestJobV3Configuration(JobRepository jobRepository, PlatformTransactionManager platformTransactionManager, PersonV3Repository personV3Repository) {  
       this.jobRepository = jobRepository;  
       this.platformTransactionManager = platformTransactionManager;  
       this.personV3Repository = personV3Repository;  
    }  
  
    @Bean  
    public Job mongoTestJobV3() {  
       return new JobBuilder("mongoTestJobV3", jobRepository)  
          .start(new StepBuilder("mongoTestStepV3", jobRepository)  
             .tasklet(  
                (contribution, chunkContext) -> {  
                   System.out.println("===mongoTestJobV3===");  
  
                   PersonV3 person = new PersonV3("seunghun", 30, List.of(new OccupationV3("developer", "google", 10, LocalDateTime.now())));  
  
                   personV3Repository.save(person);  
                   System.out.printf("%s inserted\n", person);  
  
  
                   personV3Repository.findByName("seunghun").forEach(p -> {  
                      System.out.printf("%s found\n", p);  
                   });  
  
                   personV3Repository.deleteAllByName("seunghun");  
  
                   return RepeatStatus.FINISHED;  
                }, platformTransactionManager  
             )  
             .build())  
          .incrementer(new RunIdIncrementer())  
          .build();  
    }  
}
```

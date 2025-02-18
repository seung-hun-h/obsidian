## 1. Spring Batch와 통합
- `MongoDatabase`, `MongoOperations`, `MongoRepository` 중 어떤 것이든 Spring Batch에서 사용하는데 문제 없다.
- SpringBatch는 SpringData MongoDB 의존성을 추가하면 사용할 수 있는 `ItemWriter`, `ItemReader`를 제공한다
## 2. RepositoryItemReader, RepositoryItemWriter
- SpringBatch에서 제공하는 `RepositoryItemReader`, `RepositoryItemWriter`를 사용하면 `MongoRepository`를 내부적으로 사용하여 간단하게 스텝을 구성할 수 있다
	- 호출할 메서드의 이름과 인자 목록을 전달해주어야 한다
###### MongoTestJobV4Configuration
```java
@Configuration  
public class MongoTestJobV4Configuration {  
    private final JobRepository jobRepository;  
    private final PlatformTransactionManager platformTransactionManager;  
    private final PersonV4Repository personRepository;  
  
    public MongoTestJobV4Configuration(JobRepository jobRepository, PlatformTransactionManager platformTransactionManager, PersonV4Repository personRepository) {  
       this.jobRepository = jobRepository;  
       this.platformTransactionManager = platformTransactionManager;  
       this.personRepository = personRepository;  
    }  
  
    @Bean  
    public Job mongoTestJobV4() {  
       return new JobBuilder("mongoTestJobV4", jobRepository)  
          .start(saveStep())  
          .next(deleteStep())  
          .incrementer(new RunIdIncrementer())  
          .build();  
    }  
  
    private Step saveStep() {  
       return new StepBuilder("saveStep", jobRepository)  
          .<PersonV4, PersonV4>chunk(1, platformTransactionManager)  
          .reader(listItemReader())  
          .writer(repositoryItemWriter())  
          .build();  
    }  
  
    private ListItemReader<PersonV4> listItemReader() {  
       PersonV4 person1 = new PersonV4("seunghun", 30, List.of(new OccupationV4("developer", "google", 10, LocalDateTime.now())));  
       PersonV4 person2 = new PersonV4("seunghun", 32, List.of(new OccupationV4("developer", "google", 10, LocalDateTime.now())));  
       return new ListItemReader<>(List.of(person1, person2));  
    }  
  
    private RepositoryItemWriter<PersonV4> repositoryItemWriter() {  
       RepositoryItemWriter<PersonV4> writer = new RepositoryItemWriter<>();  
       writer.setRepository(personRepository); // 실제 작업을 수행할 Repository 지정
       writer.setMethodName("save");  
       return writer;  
    }  
  
    private Step deleteStep() {  
       return new StepBuilder("deleteStep", jobRepository)  
          .<PersonV4, PersonV4>chunk(1, platformTransactionManager)  
          .reader(repositoryItemReader())  
          .writer(repositoryDeleteItemWriter())  
          .build();  
    }  
  
    private RepositoryItemReader<PersonV4> repositoryItemReader() {  
       RepositoryItemReader<PersonV4> reader = new RepositoryItemReader<>();  
       reader.setRepository(personRepository);
       reader.setMethodName("findByName");  
       reader.setPageSize(10);  
       reader.setArguments(List.of("seunghun")); // 조회 인자
       reader.setSort(Collections.singletonMap("id", Sort.Direction.ASC)); // 정렬 기준
       return reader;  
    }  
  
    private ItemWriter<PersonV4> repositoryDeleteItemWriter() {  
       return items -> {  
          items.forEach(item -> {  
             personRepository.delete(item);  
             System.out.println("deleted: " + item);  
          });  
       };  
    }  
}
```
## 3. MongoItemWriter, MongoCursorItemReader
- SpringBatch에서 제공하는 `MongoItemWriter`, `MongoCursorItemReader(또는 MongoPagingItemReader)` 를 사용하면 `MongoOperations`를 내부적으로 사용하여, 간단하게 스텝을 구성할 수 있다
###### MongoTestJobV5Configuration
```java
@Configuration  
public class MongoTestJobV5Configuration {  
    private final JobRepository jobRepository;  
    private final PlatformTransactionManager platformTransactionManager;  
    private final MongoOperations mongoOperations;  
  
    public MongoTestJobV5Configuration(JobRepository jobRepository, PlatformTransactionManager platformTransactionManager, MongoOperations mongoOperations) {  
       this.jobRepository = jobRepository;  
       this.platformTransactionManager = platformTransactionManager;  
       this.mongoOperations = mongoOperations;  
    }  
  
    @Bean  
    public Job mongoTestJobV5() {  
       return new JobBuilder("mongoTestJobV5", jobRepository)  
          .start(saveStep())  
          .start(deleteStep())  
          .incrementer(new RunIdIncrementer())  
          .build();  
    }  
  
    private Step saveStep() {  
       return new StepBuilder("saveStep", jobRepository)  
          .<PersonV5, PersonV5>chunk(1, platformTransactionManager)  
          .reader(listItemReader())  
          .writer(mongoItemWriter())  
          .build();  
    }  
  
    private ListItemReader<PersonV5> listItemReader() {  
       PersonV5 person = new PersonV5("seunghun", 30, List.of(new OccupationV5("developer", "google", 10, LocalDateTime.now())));  
       return new ListItemReader<>(List.of(person));  
    }  
  
    private MongoItemWriter<PersonV5> mongoItemWriter() {  
       MongoItemWriter<PersonV5> writer = new MongoItemWriter<>();  
       writer.setTemplate(mongoOperations); // MongoOperations 주입
       writer.setCollection("people"); // 저장할 컬렉션 이름 지정  
       return writer;  
    }  
  
    private Step deleteStep() {  
       return new StepBuilder("deleteStep", jobRepository)  
          .<PersonV5, PersonV5>chunk(1, platformTransactionManager)  
          .reader(mongoItemReader())  
          .writer(mongoDeleteItemWriter())  
          .build();  
    }  
  
    private MongoCursorItemReader<PersonV5> mongoItemReader() {  
       MongoCursorItemReader<PersonV5> reader = new MongoCursorItemReader<>();  
       reader.setQuery(query(Criteria.where("name").is("seunghun"))); // 조회 조건 설정
       reader.setTemplate(mongoOperations); // 4 
       reader.setTargetType(PersonV5.class); // 대상 도메인 타입 지정
       return reader;  
    }  
  
    private ItemWriter<PersonV5> mongoDeleteItemWriter() {  
       return items -> {  
          items.forEach(item -> {  
             System.out.printf("%s deleted\n", item);  
             mongoOperations.remove(query(Criteria.where("name").is(item.getName())), PersonV5.class);  
          });  
       };  
    }  
}
```
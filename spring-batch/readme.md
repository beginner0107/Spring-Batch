# Spring-Batch

### 기술스택

- Spring Boot 2.5.2
- Jdk11

## 기본 세팅

### EnableBatchProcessing

```java
@SpringBootApplication
@EnableBatchProcessing // 스프링 배치가 작동하기 위해 선언해야 하는 어노테이션
public class SpringBatchApplication {

	public static void main(String[] args) {
		SpringApplication.run(SpringBatchApplication.class, args);
	}

}
```

- 총 4개 설정 클래스를 실행시키며 스프링 배치의 모든 초기화 및 실행 구성이 이루이짐.
- 스프링 부트 배치의 자동 설정 클래스가 실행됨으로 빈으로 등록된 모든 Job을 검색해서 초기화와 동시에 Job을 수행하도록 구성됨.

## 스프링 배치 초기화 설정 클래스

1. `BatchAutoConfiguration`

- 스프링 배치가 초기화 될 때 자동으로 실행되는 설정 클래스
- Job을 수행하는 JobLauncherApplicationRunner 빈을 생성

2. `SimpleBatchConfiguration`

- JobBuilderFactory 와 StepBuilderFactory 생성
- 스프링 배치의 주요 구성 요소 생성 - 프록시 객체로 생성됨

3. `BatchConfigurerConfiguration`

- `BasicBatchConfigurer`

  - SimpleBatchFongiuration 에서 생성한 프록시 객체의 실제 대상 객체를 설정하는 설정 클래스
  - 빈으로 의존성 주입 받아서 주요 객체들을 참조해서 사용할 수 있다.

- `JpaBatchConfigurer`

  - JPA 관련 객체를 생성하는 설정 클래스

- **사용자 정의 BatchConfigurer 인터페이스를 구현하여 사용할 수 있음**

## Hello Spring 시작하기

```java
@RequiredArgsConstructor
@Configuration
public class HelloJobConfiguration {

  private final JobBuilderFactory jobBuilderFactory; // Job을 생성하는 빌더 팩토리
  private final StepBuilderFactory stepBuilderFactory; // Step을 생성하는 빌더 팩토리

  @Bean
  public Job helloJob() {
    return jobBuilderFactory.get("helloJob") // Job 생성
        .start(helloStep1())
        .build();
  }

  @Bean
  public Step helloStep1() {
    return stepBuilderFactory.get("helloStep1") // Step 생성
        .tasklet(new Tasklet() { // tasklet
          @Override
          public RepeatStatus execute(StepContribution contribution, ChunkContext chunkContext)
            return RepeatStatus.FINISHED;
          }
        }).build()
        ;
  }
```

1. `@Configuration 선언`

- 하나의 배치 Job을 정의하고 빈 설정

2. `JobBuilderFactory`

- Job을 생성하는 빌더 팩토리

3. `StepBuilderFactory`

- Step을 생성하는 빌더 팩토리

4. `Job`

- helloJob 이름으로 Job 생성

5. `Step`

- helloStep 이름으로 Step 생성

6. `tasklet`

- Step 안에서 단일 테스크로 수행되는 로직 구현

7. **Job 구동 -> Step을 실행 -> Tasklet을 실행**

## yml 설정

1. 테스트 혹은 개발 환경 always로 두고 사용.

- 자동으로 배치 테이블을 생성한다.

```yml
spring:
  config:
    activate:
      on-profile: mysql
  datasource:
    hikari:
      jdbc-url: jdbc:mysql://localhost:3306/springbatch?useUnicode=true&characterEncoding=utf8
      username: root
      password: 1234
      driver-class-name: com.mysql.jdbc.Driver
  batch:
    jdbc:
      initialize-schema: always
```

2. 운영에서는 never

- 자동으로 배치 테이블 생성하는 스크립트 실행하지 않겠다는 것.

## Job 관련 테이블

### BATCH_JOB_INSTANCE

- Job이 실행될 때 JobInstance 정보가 저장
- job_name과 job_key를 키로 하여 하나의 데이터 저장
- 동일한 job_name과 job_key 로 **중복 저장될 수 없다.**

### BATCH_JOB_EXECUTION

- **Job의 실행 정보가 저장**
- Job 생성 / 시작 / 종료 시간 / 실행상태 / 메시지 등을 관리

### BATCH_JOB_EXECUTION_PARAMS

- Job과 함께 실행되는 **JobParameter** 정보를 저장

### BATCH_JOB_EXECUTION_CONTEXT

- Job의 실행동안 여러가지 상태정보, 공유 데이터를 **직렬화(Json 형식)** 해서 저장
- Step간 **서로 공유 가능함**

## Step 관련 테이블

### BATCH_STEP_EXECUTION

- **Step의 실행정보가 저장**되며 생성, 시작, 종료 시간, 실행상태, 메시지 등을 관리

### BATCH_STEP_EXECUTION_CONTEXT

- Step의 실행동안 여러가지 상태정보, 공유 데이터를 **직렬화(Json 형식)** 해서 저장
- Step 별로 저장되며 Step 간 **서로 공유할 수 없음**

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


## 테이블 분석

### BATCH_JOB_INSTANCE
```sql
CREATE TABLE BATCH_JOB_INSTANCE  (
	JOB_INSTANCE_ID BIGINT  NOT NULL PRIMARY KEY ,
	VERSION BIGINT ,
	JOB_NAME VARCHAR(100) NOT NULL,
	JOB_KEY VARCHAR(32) NOT NULL,
	constraint JOB_INST_UN unique (JOB_NAME, JOB_KEY)
) ENGINE=InnoDB;
```
- JOB_INSTANCE_ID : 고유하게 식별할 수 있는 기본 키
- VERSION : 업데이트 될 때 마다 1씩 증가
- JOB_NAME : Job을 구성할 때 부여하는 Job의 이름
- JOB_KEY : job_name과 jobParameter를 합쳐 해싱한 값을 저장

### BATCH_JOB_EXECUTION
```sql
CREATE TABLE BATCH_JOB_EXECUTION  (
	JOB_EXECUTION_ID BIGINT  NOT NULL PRIMARY KEY ,
	VERSION BIGINT  ,
	JOB_INSTANCE_ID BIGINT NOT NULL,
	CREATE_TIME DATETIME(6) NOT NULL,
	START_TIME DATETIME(6) DEFAULT NULL ,
	END_TIME DATETIME(6) DEFAULT NULL ,
	STATUS VARCHAR(10) ,
	EXIT_CODE VARCHAR(2500) ,
	EXIT_MESSAGE VARCHAR(2500) ,
	LAST_UPDATED DATETIME(6),
	JOB_CONFIGURATION_LOCATION VARCHAR(2500) NULL,
	constraint JOB_INST_EXEC_FK foreign key (JOB_INSTANCE_ID)
	references BATCH_JOB_INSTANCE(JOB_INSTANCE_ID)
) ENGINE=InnoDB;
```
- JOB_EXECUTION_ID : JobExecution을 고유하게 식별할 수 있는 기본 키, JOB_INSTANCE와 일대 다 관계
- VERSION : 업데이트 될 때마다 1씩 증가
- JOB_INSTANCE_ID : JOB_INSTANCE의 키 저장
- CREATE_TIMME : 실행(Execution)이 생성된 시점을 DateTime형식으로 기록
- START_TIME : 실행(Execution)이 시작된 시점을 DateTime형식으로 기록
- END_TIME : 실행이 종료된 시점을 DateTime으로 기록하여 Job 실행 도중 오류가 발생해서 Job 이 중단된 경우 값이 저장되지 않을 수 있음
- STATUS : 실행 상태(BatchStatus)를 저장 (COMPLETED, FAILED, STOPPED...)
- EXIT_CODE : 실행 종료코드(ExitStatus)를 저장(COMPLETED, FAILED...)
- EXIT_MESSAGE : Status가 실패일 경우 실패 원인 등의 내용을 저장
- LAST_UPDATED : 마지막 실행(Execution) 시점을 DateTime으로 기록


### BATCH_JOB_EXECUTION_PARAMS
```sql
CREATE TABLE BATCH_JOB_EXECUTION_PARAMS  (
	JOB_EXECUTION_ID BIGINT NOT NULL ,
	TYPE_CD VARCHAR(6) NOT NULL ,
	KEY_NAME VARCHAR(100) NOT NULL ,
	STRING_VAL VARCHAR(250) ,
	DATE_VAL DATETIME(6) DEFAULT NULL ,
	LONG_VAL BIGINT ,
	DOUBLE_VAL DOUBLE PRECISION ,
	IDENTIFYING CHAR(1) NOT NULL ,
	constraint JOB_EXEC_PARAMS_FK foreign key (JOB_EXECUTION_ID)
	references BATCH_JOB_EXECUTION(JOB_EXECUTION_ID)
) ENGINE=InnoDB;
```
- JOB_EXECUTION_ID : JobExecution 식별 키, JOB_EXECUTION 과는 일대다 관계
- TYPE_CD : String, Long, Date, Double 타입 정보
- KEY_NAME : 파라미터 키 값
- STRING_VAL : 파라미터 문자 값
- DATE_VAL : 파라미터 날짜 값
- LONG_VAL : 파라미터 LONG 값
- DOUBLE_VAL : 파라미터 DOUBLE 값
- IDENTIFYING : 식별여부(TRUE, FALSE)

### BATCH_JOB_EXECUTION_CONTEXT
```sql
CREATE TABLE BATCH_STEP_EXECUTION_CONTEXT  (
	STEP_EXECUTION_ID BIGINT NOT NULL PRIMARY KEY,
	SHORT_CONTEXT VARCHAR(2500) NOT NULL,
	SERIALIZED_CONTEXT TEXT ,
	constraint STEP_EXEC_CTX_FK foreign key (STEP_EXECUTION_ID)
	references BATCH_STEP_EXECUTION(STEP_EXECUTION_ID)
) ENGINE=InnoDB;
```
- JOB_EXECUTION_ID : JobExecution 식별 키, JOB_EXECUTION 마다 각 생성
- SHORT_CONTEXT : JOB의 실행 상태정보, 공유데이터 등의 정보를 문자열로 저장
- SERIALIZED_CONTEXT : 직렬화(serialized)된 컨텍스트

### BATCH_STEP_EXECUTION
```sql
CREATE TABLE BATCH_STEP_EXECUTION  (
	STEP_EXECUTION_ID BIGINT  NOT NULL PRIMARY KEY ,
	VERSION BIGINT NOT NULL,
	STEP_NAME VARCHAR(100) NOT NULL,
	JOB_EXECUTION_ID BIGINT NOT NULL,
	START_TIME DATETIME(6) NOT NULL ,
	END_TIME DATETIME(6) DEFAULT NULL ,
	STATUS VARCHAR(10) ,
	COMMIT_COUNT BIGINT ,
	READ_COUNT BIGINT ,
	FILTER_COUNT BIGINT ,
	WRITE_COUNT BIGINT ,
	READ_SKIP_COUNT BIGINT ,
	WRITE_SKIP_COUNT BIGINT ,
	PROCESS_SKIP_COUNT BIGINT ,
	ROLLBACK_COUNT BIGINT ,
	EXIT_CODE VARCHAR(2500) ,
	EXIT_MESSAGE VARCHAR(2500) ,
	LAST_UPDATED DATETIME(6),
	constraint JOB_EXEC_STEP_FK foreign key (JOB_EXECUTION_ID)
	references BATCH_JOB_EXECUTION(JOB_EXECUTION_ID)
) ENGINE=InnoDB;
```
- STEP_EXECUTION_ID : Step의 실행정보를 고유하게 식별할 수 있는 기본 키
- VERSION : 업데이트 될 때마다 1씩 증가
- STEP_NAME : Step을 구성할 때 부여되는 Step 이름
- JOB_EXECUTION_ID : JobExecution 기본키, JobExecution 과는 일대 다 관계
- START_TIME : 실행(Execution)이 시작된 시점을 DateTime 형식으로 기록
- END_TIME : 실행이 종료된 시점을 DateTime 으로 기록하여 Job 실행 도중 오류가 발생해서 Job이 중단된 경우 값이 저장되지 않을 수 있음
- STATUS : 실행 상태(BatchStatus)를 저장 (COMPLETED, FAILED, STOPPED...)
- COMMIT_COUNT : 트랜잭션 당 커밋되는 수를 기록
- READ_COUNT : 실행시점에 Read한 Item 수를 기록
- FILTER_COUNT : 실행도중 필터링된 Item 수를 기록
- WRITE_COUNT : 실행도중 저장되고 커밋된 Item 수를 기록
- READ_SKIP_COUNT : 실행도중 Read가 Skip된 Item 수를 기록
- WRITE_SKIP_COUNT : 실행도중 Write가 Skip된 Item 수를 기록
- PROCESS_SKIP_COUNT : 실행도중 Process가 Skip된 Item 수를 기록
- ROLLBACK_COUNT : 실행도중 rollback이 일어난 수를 기록
- EXIT_CODE : 실행 종료코드(ExitStatus)를 저장(COMPLETED, FAILED...)
- EXIT_MESSAGE : Status가 실패일 경우 실패 원인 등의 내용을 저장
- LAST_UPDATED : 마지막 실행(Execution) 시점을 TimeStamp 형식으로 기록

### BATCH_STEP_EXECUTION_CONTEXT
```sql
CREATE TABLE BATCH_STEP_EXECUTION_CONTEXT  (
	STEP_EXECUTION_ID BIGINT NOT NULL PRIMARY KEY,
	SHORT_CONTEXT VARCHAR(2500) NOT NULL,
	SERIALIZED_CONTEXT TEXT ,
	constraint STEP_EXEC_CTX_FK foreign key (STEP_EXECUTION_ID)
	references BATCH_STEP_EXECUTION(STEP_EXECUTION_ID)
) ENGINE=InnoDB;
```
- STEP_EXECUTION_ID : StepExecution 식별 키, STEP_EXECUTION 마다 각 생성
- SHORT_CONTEXT : STEP의 실행 상태정보, 공유데이터 등의 정보를 문자열로 저장
- SERIALIZED_CONTEXT : 직렬화(serialized)된 전체 컨텍스트

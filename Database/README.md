# Database

# Statement와 PreparedStatement이란

### JDBC API Interface
Statement와 PreparedStatement 둘다 SQL 쿼리를 실행하는데 쓰이는 인터페이스다. 

```
SQL 문 실행 절차
1. 구문 오류 체크 (parse)
2. 공유 영역에서 해당 구문 검색 (parse)
3. 권한 체크 (parse)
4. 실행 계획 수립 (parse)
5. 실행 계획 공유 영역에 저장 (parse)
6. 쿼리 실행 (Excute)
7. 데이터 인출 (patch)
```
```
1~7 모두 실행 : Hard Parsing
```
```
1, 2, 6, 7 만 실행 : Soft Parsing
```
Statement를 사용하여 SELECT 쿼리를 입력했을 때에는 매번 parse부터 fetch까지 모든 과정을 수행한다. (HARD Parsing)

Prepared Statement를 사용하는 경우 parse과정을 최초 1번만 수행하고 이후는 생략할 수 있다. (Soft Parsing)
<hr>

## Statement
Statement는 문자열 기반 SQL 쿼리를 실행할 때 쓰인다.

### 1. 연쇄적인 문자열로 코드를 읽기 어렵다.
```
public void insert(PersonEntity personEntity) {
    String query = "INSERT INTO persons(id, name) VALUES(" + personEntity.getId() + ", '"
    + personEntity.getName() + "')";

    Statement statement = connection.createStatement();
    statement.executeUpdate(query);
}
```
### 2. SQL Injection에 취약하다. 
```
--는 SQL문에서 주석으로 해서되므로 update문이 무시된다.
dao.update(new PersonEntity(1, "hacker' --")); 

insert문은 실패된다.
dao.insert(new PersonEntity(1, "O'Brien"))
```

### 3. 캐시를 활용하지 않는다.
SQL 실행절차 1번 처리구간을 매 요청마다 수행한다. 

### 4. DDL 쿼리에 적합하다.
```
public void createTables() {
    String query = "create table if not exists PERSONS (ID INT, NAME VARCHAR(45))";
    connection.createStatement().executeUpdate(query);
}
```

### 5. 파일이나 배열을 저장/조회 할 수 없다.
 
<hr>

## PreparedStatement
PreparedStatement는 Statement를 상속받은 인터페이스다. 매개 변수화된 SQL 쿼리를 실행할 때 쓰인다. 

### 1. 바인딩 메서드 (setXxx)
다양한 객체 유형을 바인딩하는 메서드가 있습니다. (파일과 배열을 포함해서)
```
public void insert(PersonEntity personEntity) {
    String query = "INSERT INTO persons(id, name) VALUES( ?, ?)";

    PreparedStatement preparedStatement = connection.prepareStatement(query);
    preparedStatement.setInt(1, personEntity.getId());
    preparedStatement.setString(2, personEntity.getName());
    preparedStatement.executeUpdate();
}
```

### 2. SQL Injection을 예방한다. 
Prepared Statement에서 바인딩 변수를 사용하였을 때 쿼리의 문법 처리과정이 미리 선 수행되기 때문에 
바인딩 데이터는 문법적인 의미를 가질 수 없다. 

### 3. 미리 컴파일
Soft Parsing 사용 

### 4. batch 실행을 제공한다.
```
public void insert(List<PersonEntity> personEntities) throws SQLException {
    String query = "INSERT INTO persons(id, name) VALUES( ?, ?)";
    PreparedStatement preparedStatement = connection.prepareStatement(query);
    for (PersonEntity personEntity: personEntities) {
        preparedStatement.setInt(1, personEntity.getId());
        preparedStatement.setString(2, personEntity.getName());
        preparedStatement.addBatch();
    }
    preparedStatement.executeBatch();
}
```
### 5. 파일, 배열 저장/조회

파일: BLOB과 CLOB 데이터 타입 사용

배열: java.sql.Array <-> SQL Array

### 6. getMetadata() 메서드
결과값에 대한 정보를 포함하고 있는 메서드

## 결론
Statement는 DDL(CREATE, ALTER, DROP) 구문을 처리할 때 적합하다.
매 실행시 Query를 다시 파싱하기 때문에 속도가 느리며, SQL Injection공격에 취약하다.
-> 필터나 방어 로직을 추가하는 것이 좋다.

Prepared Statement는 DML(SELECT, INSERT, UPDATE, DELETE)구문 처리에 적합하다.
그리고 캐시에 저장된 Query를 활용하기 때문에 실행이 빠르며 SQL Injection을 막기 위한 방법으로 활용된다.

<hr>

출처
- [Difference Between Statement and PreparedStatement](https://www.baeldung.com/java-statement-preparedstatement)
- [SQL injection 대응 : Prepared Statement](https://velog.io/@seaworld0125/SQL-injection-%EB%8C%80%EC%9D%91%EB%B0%A9%EB%B2%95-Prepared-Statement)
  
<hr>

# Database에서 slow query가 발생했을 경우 어떻게 대처하는지?
## slow query란
DBMS에서 Client로 요청받은 query를 실행할 때 일정 시간 이상 수행되지 못한 query

### slow query를 잡아내는 3가지 방법
- slow query 로그 남기기
- 쿼리 실행계획 로그에 남기기
- 쿼리 실행 통계 보기

PostgreSQL의 경우로 설명
### 1. slow query 로그 남기기
```
slow query 로그는 에러가 발생한 쿼리는 기록하지 않는다. 
general 로그에서는 에러가 발생한 쿼리를 포함해서 모든 쿼리를 기록하므로 두 개의 로그를 모두 활성화하면 프로파일링에 도움이 된다.
```
postgresql.conf에 `log_min_duration_statement = 1000` 를 설정한다. 
- 밀리세컨드 단위로 설정
- 설정값보다 오래 걸리는 쿼리를 로그 파일에 기록
- 0으로 설정하면 모든 로그 기록
- 수정 후 reload 하여 적용 (restart 필요 없음)

부하를 유발하는 단일 쿼리를 파악하기 쉽다. 그러나 처리 시간은 빠르지만, 여러번 호출되서 부하를 발생시키는 쿼리 실행을 인지하기는 어렵다는 단점이 있다. 


### 2. 쿼리 실행계획 로그에 남기기
PostgreSQL의 경우 postgresql.conf에 auto_explain 라이브러리를 추가한다. 
`session_preload_libraries = 'auto_explain';`

실행계획을 로그에 남기면, 당시의 쿼리 실행 계획을 볼 수 있단 장점이 있다.

롱쿼리가 발생한 이후 데이터가 더 쌓이거나 삭제되면, 문제가 된 순간의 실행계획을 알 수 없다. 때문에 이를 확인할 수 있단 장점이 있다. 그런데 EXPLAIN ANALYZE 명령문을 기반으로 로그를 남기기 때문에, 롱쿼리를 다시 실행시킨단 리스크가 있다. 

그리고 단일 롱쿼리만 파악할 수 있기 때문에, 짧지만 여러번 호출되서 문제를 일으키는 쿼리를 알 수 없는 단점이 있다.


### 3. 쿼리 실행 통계 보기
쿼리 실행 통계 라이브러리는 shared memory를 사용하기 때문에, 해당 모듈을 추가 / 삭제할 때는 항상 서버를 restart해줘야한다. `shared_preload_libraries = 'pg_stat_statements'`

이 방식을 사용하면, 빨리 실행되지만 부하를 일으키는 쿼리를 파악하기 좋단 장점이 있다. 

<hr>

출처
- [PostgreSQL 슬로우쿼리를 잡아내는 3가지 방법](https://americanopeople.tistory.com/288)
- [MySQL 성능 개선을 위한 프로파일링 1편: 슬로우 쿼리 로그](https://velogio@breadkingdomMySQL-%EC%84%B1%EB%8A%A5-%EA%B0%9C%EC%84%A0%EC%9D%84-%EC%9C%84%ED%95%9C-%ED%94%84%EB%A1%9C%ED%8C%8C%EC%9D%BC%EB%A7%81-1)
- [PostgreSQL Slow Query 검출](https://brufen97.tistory.com/5)

<hr>

# Redo와 Undo
트랜잭션이 수행되는 동안 시스템에 알 수 없는 오류 또는 물리적으로 문제가 발생했을 때, 트랜잭션의 수행을 되돌려야 합니다. <strong>rollback</strong> 이란 트랜잭션 내의 질의를 수행하면서 문제가 발생했을 경우 수행되는 것이지만, 시스템의 오류 또는 물리적인 문제의 경우 <strong>시스템 상의 문제이므로 트랜잭션이 다시 수행되어야 합니다. </strong>

이를 <strong>시스템 회복(recovery)</strong>이라 합니다.
회복은 데이터의 신뢰성을 보장하며, 트랜잭션의 영속성과 원자성을 보장합니다. 

Checkpoint 이후 Log 기록을 보면서 완료되지 않은 트랜잭션에 대해 수행
- Redo: 이전 상태로 되돌아간 후, 실패가 발생하기 전까지의 과정을 그대로 따라가는 것을 의미합니다.
- Undo: 트래잭션을 이전 상태로 되돌리는 것을 의미

## Redo
DB 장애 시 Buffer Pool에 저장되어 있던 데이터의 유실을 방지(데이터 복구)하기 위해 사용된다.
   
### InnoDB Buffer Pool?
Buffer Pool은 InnoDB 엔진이 Table Caching 및 Index Data Caching을 위해 이용하는 메모리 공간이다. 
다시 말해, Buffer Pool의 크기가 클수록 상대적으로 캐싱되는 데이터의 양이 늘어나기 때문에 Disk에 접근하는 횟수가 줄어들고, 이것은 DB의 성능 향상으로 이어진다.
하지만, Buffer Pool은 메모리 공간이기 때문에 <strong>데이터베이스 장애 시 Buffer Pool에 있는 내용은 휘발된다. </strong> 이것은 ACID를 보장할수 없게 되고, 다시 해석하면 장애를 복구하더라도 데이터는 복구될 수 없다는 것을 의미한다. 

실제 DB에서 Commit이 발생하면 바로 디스크 영역(Table Space)으로 들어가는 것이 아닌 메모리 영역(Buffer Poll & Log Buffer)에 들어가는 것을 확인할 수 있다. (DISK I/O 절약)

### Redo Log
Redo를 하기 위해서는 정상적으로 실행 되기까지의 과정을 기록해야 하는데, 이를 <strong>Redo Log</strong> 라고 한다. DML(SELECT 제외), DDL, TCL 등 데이터 변경이 일어나는 모든 것을 Redo Log 에 기록한다.

- DML (Data Manipulation Language)
  - SELECT, INSERT, UPDATE, DELETE

- DDL (Data Definition Language)
  - CREATE, ALTER, DROP, TRUNCATE

- TCL (Transactioin Control Language)
  - COMMIT, ROLLBACK

### Redo Log File
Log Buffer는 메모리 영역이기에 용량이 제한적이다. 용량이 제한적이기 때문에 Checkpoint 이벤트 발생시점에 Redo Log Buffer에 있던 데이터들을 Disk에 File로 저장하게 된다. 이 파일을 <strong>Redo Log File</strong> 라고 한다.
```
Checkpoint란 메모리상의 수정된 데이터 블럭과 디스크의 데이터 파일을 동기화시키는 데이터베이스 이벤트

다음을 목적으로 한다.
1. 데이터의 일관성 보장
2. 데이터베이스의 빠른 복구
```
Redo Log File은 두 개의 파일로 구성된다. 하나의 파일이 가득차면 Log switch가 발생하며 다른 파일에 쓰게 된다. Log switch가 발생할 때 마다 Checkpoint 이벤트도 발생되는데, 이때 InnoDB Buffer Pool Cache에 있던 데이터들이 백그라운드 스레드에 의해 Disk에 기록된다.

```
Checkpoint 이벤트가 발생하기 전에 장애가 발생한다면 Buffer Pool에 있던 데이터들은 휘발되지만, 
마지막 Checkpoint가 수행된 시점까지의 데이터가 Redo Log File로 남아있기 때문에 이 파일을 사용하여 데이터를 복구할 수 있다. 
```

### 과정
1. 실제 데이터 변경 전에 Redo Log Buffer에 데이터 변경에 대한 내용을 먼저 저장한다. (선 로그 기법: Log Ahead)
2. DB Cache에도 데이터 변경에 대한 내용을 기록한다.
3. LGWR 백그라운드 프로세스에 의해 Redo Log Buffer에 있는 내용을 Redo Log File에 저장한다.
4. Commit이 발생하게 되면 Control File에 트랜잭션의 고유한 번호(SCN)를 기록한다.
5. Log Switch가 발생하게 되면 Checkpoint 발생. 이 때 Checkpoint 프로세스가 DBWR 프로세스에게 Checkpoint 신호를 전달하면서 DBWR은 DB Cache 블록을 Data File로 저장한다.
6. Checkpoint가 DBWR에게 Checkpoint 신호를 전달하면서 Checkpoint SCN, 꽉 찬 로그파일 안의 제일 마지막 번호를 알려준다. 그 번호는 Data File과 Control File에 저장된다.

LGWR 자세히 
: [4장. 로그 기록자 백그라운드 프로세스(Log Writer, LGWR)](https://1duffy.tistory.com/24)

<hr>

## Undo
실행 취소 로그 레코드의 집합으로 트랜잭션 실행 후 <strong>Rollback</strong> 시 <strong>Undo Log를 참조해 이전 데이터로 복구</strong>할 수 있도록 로깅 해놓은 영역이다.

### Undo Log가 필요한 이유 
작업 수행 중에 수정된 페이지들이 버퍼 관리자의 버퍼 교체 알고리즘에 따라서 디스크에 출력될 수 있다.

버퍼 교체는 전적으로 버퍼의 상태에 따라 결정되며, 일관성 관점에서 봤을 때는 임의의 방식으로 일어나게 된다. 즉 아직 완료되지 않은 트랜잭션이 수정한 페이지들도 디스크에 출력될 수 있으므로, 만약 해당 트랜잭션이 어떤 이유든 정상적으로 종료될 수 없으면 트랜잭션이 변경한 페이지들은 원상 복구되어야 한다. 이러한 복구를 UNDO라고 한다.

### 어떻게 파일로 저장되는가?
Undo Log도 Redo Log와 마찬가지로 Log Buffer에 기록된다. Undo Records영역에 기록되는 것이다.
저장되는 데이터는 PK 값과 변경되기 전의 데이터 값이다. 

Redo Log가 트랜잭션 Commit과 CheckPoint 시 디스크에 기록되지만, Undo Log는 CheckPoint 시 디스크에 기록된다.

<strong>데이터를 수정함과 동시에 Rollback을 대비하기 위해, 업데이트 전의 데이터를 Undo Records로 기록하는 것이다. </strong>

<hr>

출처
- [Mysql Redo / Undo Log](https://velog.io/@pk3669/Mysql-Redo-Undo-Log)
- [🙈[DB이론] 신뢰성과 회복(Recovery)🐵](https://victorydntmd.tistory.com/130)

<hr>

# DELETE, TRUNCATE, DROP 차이
## DELETE
```
DELETE [LOW_PRIORITY] [QUICK] [IGNORE] FROM tbl_name [[AS] tbl_alias]
    [PARTITION (partition_name [, partition_name] ...)]
    [WHERE where_condition]
    [ORDER BY ...]
    [LIMIT row_count]
```
- WHERE 절을 사용하여 데이터를 하나하나 선택해서 제거하는 방식
  - `DELETE FROM dbtable WHERE {조건};`
- WHERE 절을 사용하지 않고 테이블의 모든 데이터를 삭제하더라도, 내부적으로는 한 줄씩 제거된다.
  - `DELETE FROM dbtable;` 
- 원하는 데이터만 골라서 삭제할 때는 DELETE 사용 / 전체 데이터를 삭제할 때는 TRUNCATE 사용
- 데이터를 삭제하더라도 데이터가 담겨있던 Storage는 Release 되지 않는다.
- DELETE된 데이터는 COMMIT 명령어를 사용하기 전이라면 ROLLBACK 명령어를 통해 되돌릴 수 있다.
- 모든 작업을 로그에 남긴다.

## TRUNCATE
```
TRUNCATE [TABLE] tbl_name
```
- 테이블을 완전히 비운다. `DROP` 권한이 필요하다.
- 논리적으로 모든 행을 삭제하는 `DELETE` 문 또는 연속의 `DROP TABLE` 과 `CRATE TABLE` 문과 유사하다.
- 고성능을 위해 데이터 삭제의 DML을 bypass(우회)한다. 따라서 `ON DELETE` 트리거가 발생하지 않으며 부모-자녀 외부 키 관겨가 있는 InnoDB 테이블에 대해 수행할 수 없으며 DML 처럼 롤백할 수 없다. 그러나 automic DDL을 지원하는 스토리지 엔진을 사용하는 테이블의 `TRUNCATE TABLE` 작업은 서버가 작업 중에 중지되면 완전히 커밋되거나 롤백된다.
- `DELETE` 문과 유사하지만 DML문이 아닌 DDL문으로 분류된다. 다음과 같은 점에서 DELETE와 다르다.

### TRUNCATE가 DELETE와 다른 점 
- 행을 하나씩 삭제하는 `DELETE`보다 테이블을 drop 후 re-create하는 `TRUNCATE`가 속도가 더 빠르다.
- `TRUNCATE`는 암시적 커밋을 실행하므로 롤백할 수 없다.
- 이진 로깅과 replication(복제)를 위해 DDL로 처리되며 항상 로깅된다.


## DROP
```
DROP [TEMPORARY] TABLE [IF EXISTS]
    tbl_name [, tbl_name] ...
    [RESTRICT | CASCADE]
```
- 하나 이상의 테이블을 제거한다.
- 테이블이 파티셔닝 된 경우, 테이블의 정의, 테이블의 모든 파티션, 해당 파티션에 저장된 모든 데이터 및 삭제된 테이블과 관련된 모든 파티션 정의가 제거된다.
- 테이블을 삭제하면 테이블에 대한 트리거도 삭제된다.
- `TEMPORY` 키워드와 함께 사용되는 경우를 제외하고 암시적 커밋을 실행한다.
- 테이블이 삭제되어도 테이블에 대해 특별히 부여된 권한은 자동으로 삭제되지 않는다. 수동으로 삭제해야 한다. 

<hr>

출처
- [Delete vs truncate vs drop 차이점](https://itblacksmith.tistory.com/72)
- [13.1.37 TRUNCATE TABLE Statement](https://dev.mysql.com/doc/refman/8.0/en/truncate-table.html)
- [13.1.32 DROP TABLE Statement](https://dev.mysql.com/doc/refman/8.0/en/drop-table.html)

<hr>

# ORM 
## Object-Relational-Mapping
- 객체-관계 매핑
- 프로그래밍 언어의 객체 지향 패러다임을 사용하여 SQL 쿼리를 작성할 수 있는 아이디어
- 즉, SQL 대신 우리가 선택한 언어를 사용하여 데이터베이스와 상호작용 하려고 한다.
- 여기서 Object-relational-mapper가 등장한다. 대부분의 사람들이 "ORM"이라고 말하면 이 기술을 구현하는 라이브러리를 나타낸다.
- 대부분의 경우 객체 지향 프로그램과 관계형 데이터베이스 사이의 다리를 만드는 데 사용되는 기술이다. 즉, ORM은 객체 지향 프로그래밍(OOP)을 관계형 데이터베이스에 연결하는 계층으로 볼 수 있다. 
- 웹 애플리케이션과 데이터베이스 사이에 위치하는  미들웨어 어플리케이션 또는 도구이다.
- Model 클래스가 데이터베이스의 테이블이 되고 각 인스턴스가 테이블의 행이 되는 방식으로 매핑을 수행한다.

## 장점과 단점
장점
- 객체 지향적인 코드로 인해 더 직관적이다. 비즈니스 로직에 집중할 수 있게 도와준다.
  - SQL의 절차적이고 순차적인 접근이 아닌 객체 지향적인 접근으로 인해 생산성이 증가한다.
- 복잡한 과정을 숨김으로써 고수준의 구현만 신경쓰면 된다. 
- 데이터베이스 시스템을 추상화하여 데이터베이스를 변경해도 코드를 변경할 필요가 없다.

단점
- ORM이 복잡한 세부 정보를 추상화하는 데 도움이 되더라도 각 명령의 결과를 알고 있어야 한다.
- ORM을 사용하여 복잡한 쿼리를 코딩하는 데 성능 문제가 발생할 수 있다.

<hr>

출처
- [What is an ORM – The Meaning of Object Relational Mapping Database Tools](https://www.freecodecamp.org/news/what-is-an-orm-the-meaning-of-object-relational-mapping-database-tools/)
- [ORM Tools in Java](https://www.javatpoint.com/orm-tools-in-java)
- [ORM이란](https://gmlwjd9405.github.io/2019/02/01/orm.html)
- [What is an ORM and Why You Should Use it](https://blog.bitsrc.io/what-is-an-orm-and-why-you-should-use-it-b2b6f75f5e2a)

<hr>

# 데이터베이스 튜닝 (Database tuning)
## 개념
- 데이터베이스 어플리케이션, 데이터베이스 시스템, 운영체제의 이해를 바탕으로, 불합리한 요소를 찾아 제거/수정하여 성능을 개선하기 위한 일련의 작업
- 특히 데이터베이스 어플리케이션이 높은 작업 처리량과 짧은 응답시간을 갖도록 하는 것이 중요하다.
## 목적 
- 데이터베이스 설계 및 활용에 존재하는 문제점을 파악하여 분석하고, 이렇게 분석된 문제점들을 해결함으로써 데이터베이스의 활용 성능을 최적화시킨다.
- 데이터베이스 튜닝을 수행함으로써 데이터베이스를 활용하는 시스템을 안정시키고, 또한 사용자의 만족과 관리자의 관리 능력을 향상시킬 수 있다. 

## 튜닝 3단계 
### 1. DB 설계 튜닝 (모델링 관점)
- <strong>데이터베이스 설계 단계</strong>에서 성능을 고려하여 설계
- 데이터 모델링, 인덱스 설계
- 데이터파일, 테이블 스페이스 설계
- 데이터베이스 용량 산정
- Ex. 반정규화, 분산파일, 배치

### 2. DBMS 튜닝 (환경 관점)
- 성능을 고려하여 메모리나 블록 크기 지정
- CPU, 메모리 I/O에 대한 관점
- Ex. Buffer 크기, Cache 크기

### 3. SQL 튜닝 (APP 관점)
- SQL 작성 시 성능 고려
- Join, Indexing, SQL Execution Plan
- Ex. Hash, Join

<hr>

출처
- [데이터베이스 튜닝 (DB Tuning)](http://blog.skby.net/%EB%8D%B0%EC%9D%B4%ED%84%B0%EB%B2%A0%EC%9D%B4%EC%8A%A4-%ED%8A%9C%EB%8B%9D-db-tuning/)
- [데이터베이스 튜닝](https://dataonair.or.kr/db-tech-reference/d-lounge/report/?mod=document&uid=239679)
- [데이터베이스 튜닝이란?](http://wiki.gurubee.net/pages/viewpage.action?pageId=14024765)

<hr>

# 함수 종속성 (Functional Dependency)
- 함수 종속성은 데이터베이스와 관련된 **두 속성 집합 사이의 제약 조건**이다.
- 일반적으로 테이블 내의 기본 키 속성과 키가 아닌 속성 사이에 존재한다.
- 함수 종속성은 화살표(→)로 표시된다.
- **A가 B를 함수적으로 결정**하면, `A → B` 로 표시된다.
- Ex. `employee_id → employee_name` 
    - `employee_id`가 함수적으로 `employee_name`을 결정.
    - `employee_name`은 함수적으로 `employee_id`에 의존.
- A는 **결정 인자(determinant set)**, B는 **종속 인자(dependent set)**
- `A → B`는 A의 특정 값의 모든 인스턴스에 B의 동일한 값이 있음을 의미한다.
  - 아래 표는 `A → B`는 참이지만 `B → A`는 참이 아니다. `B = 3` 에 대한 A 값이 다르기 때문이다.
    ```
    A   B
    ------
    1   3
    2   3
    4   0
    1   3
    4   0
    ```
## 완전 함수 종속성 (Fully Functional Dependency)
- X와 Y는 속성집합
- Y는 X에 의존한다. 그리고 X의 어떤 적절한 **부분 집합**에도 의존하지 않는다면, **Y는 X에 완전히 의존한다.**
- Ex. `ABC → D`
  - D는 ABC의 어떤 부분 집합에도 의존하지 않고 ABC에 완전히 의존한다. 
  - AB, BC, A, B 등과 같은 ABC의 부분 집합은 D를 결정할 수 없다.

## 부분 함수 종속성 (Partial Functional Dependency)
- Y는 X에 의존한다. 그리고 X의 부분 집합에 의존하면, **Y는 X에 부분 함수적으로 의존한다.**
- Ex. `AC → B`, `A → D`, `D → B` 
  -  `A → D → B` 로 A는 단독으로 B를 결정할 수 있다. 
  -  이는 B가 부분적으로 AC에 의존한다는 것을 의미한다.

## Trivial Functional Dependency
- 만약 B가 A의 부분 집합인 경우 `A → B`는 trivial 함수 종속성을 갖는다.
    ```
    {Employee_id, Employee_Name} → Employee_Id
    ```
- `A → A`, `B → B` 의 경우도 trivial 함수 종속성을 갖는다.


## Non-trivial Functional Dependency
- 만약 B가 A의 부분 집합이 아닌 경우 `A → B`는 non-trivial 함수 종속성을 갖는다.
    ```
    ID   →    Name
    Name →    DOB  
    ```
<hr>

출처
- [Introduction of Database Normalization](https://www.geeksforgeeks.org/introduction-of-database-normalization/)
- [Differentiate between Partial Dependency and Fully Functional Dependency](https://www.geeksforgeeks.org/differentiate-between-partial-dependency-and-fully-functional-dependency/)
- [Functional Dependency](https://www.javatpoint.com/dbms-functional-dependency)

<hr>

# 정규화 (Normalization)
- 데이터 중복성을 줄이거나 없애기 위해 데이터베이스의 속성을 구성하는 과정이다. 
- 데이터 중복은 동일한 데이터가 여러 곳에서 반복되기 때문에 데이터베이스의 크기를 불필요하게 증가시킨다. 
- 삽입, 삭제, 업데이트 작업 중에도 불일치(Inconsistency) 문제가 발생한다.

## 장점과 단점
장점
- 데이터베이스를 표준화해 데이터베이스 크기가 줄어들고 데이터 중복을 방지한다. 
- 여러 위치에서 중복되는 데이터가 없으므로 삽입과 업데이트가 빠르게 실행된다.
- 정규화된 테이블은 정규화되지 않은 테이블보다 작다. 이는 일반적으로 테이블이 버퍼에 들어갈 수 있으므로 더 빠른 성능을 제공한다. 
- 데이터 무결성과 일관성이 유지된다. 
- 데이터베이스 변경 시 이상 현상(Anomaly)이 제거된다. 

단점
- 데이터가 중복되지 않으므로 테이블 조인이 필요하다. 쿼리가 더 복잡해지고 읽기 시간이 느려진다. 
- 조인이 필요하기 때문에 인덱싱이 효율적으로 작동하지 않는다. 읽기 시간이 느려진다. 

<hr>

출처
- [Pros and Cons of Database Normalization](https://dzone.com/articles/pros-and-cons-of-database-normalization)

<hr>

# 이상현상 (Anomaly)
## 이상현상이란
- 테이블 내의 데이터들이 불필요하게 중복되어 테이블을 조작할 때 발생하는 데이터 불일치 현상이다.
- 테이블을 잘못 설계하여 삽입, 삭제, 갱신할 때 오류가 발생하게 되는 것이다.
- 이상현상에는 크게 3가지가 있으며, 정규화를 통해 이상현상을 해결할 수 있다.
  - **삽입 이상**: 원하지 않는 자료가 삽입된다든지, key가 없어 삽입하지 못하는(불필요한 데이터를 추가해야 삽입할 수 있는) 문제점
  - **갱신 이상**: 일부만 변경하여 데이터가 불일치하는 모순, 또는 중복되는 튜플이 존재하게 되는 문제점
  - **삭제 이상**: 하나의 자료만 삭제하고 싶지만, 그 자료가 포함된 튜플 전체가 삭제됨으로 원하지 않는 정보 손실이 발생하는 문제점

<img src="../imgs/db-anomaly-1.png">

## 1. 삽입 이상
- "melon" 이라는 아이디, "성원용"이라는 이름, "gold" 등급을 가진 
- 신규 고객의 데이터는 이벤트참여 릴레이션에 삽입할 수 없다.
- 참여하지 않은 임시 이벤트 번호(불필요한 데이터)를 추가해야 삽입할 수 있다.

<img src="../imgs/db-anomaly-2.png">

## 2. 갱신 이상
- 아이디가 "apple"인 고객의 등급이 "gold"에서 "vip"로 변경되었는데,
- 일부 튜플에 대해서만 등급이 수정된다면 
- "apple" 고객이 서로 다른 등급을 가지는 모순이 발생한다. 

<img src="../imgs/db-anomaly-3.png">

## 3. 삭제 이상
- 아이디가 "orange"인 고객이 이벤트 참여를 취소하여
- 관련 튜플을 삭제하게 되면,
- 이벤트와 관련이 없는 고객 아이디, 고객 이름, 등급 데이터까지 손실된다.

<img src="../imgs/db-anomaly-4.png">

<hr>

출처
- [이상현상, 함수종속, 정규화](https://velog.io/@hjhj4232/6.-%EC%9D%B4%EC%83%81%ED%98%84%EC%83%81%EA%B3%BC-%ED%95%A8%EC%88%98%EC%A2%85%EC%86%8D)
- [정규화 & 함수 종속성 & 이상현상](https://rebro.kr/159)

<hr>

# 정규화 레벨 
## 제 1 정규화 (First Normal Form, 1NF)
- 각 칼럼은 **원자 값**을 갖어야 한다.
  - 원자 값 = 더 이상 논리적으로 분해될 수 없는 값
- 하나의 컬럼은 같은 종류나 타입(type)을 가져야 한다.
- 각 컬럼이 유일한(unique) 이름을 가져야 한다.
- 칼럼의 순서가 상관없어야 한다.

## 제 2 정규화 (Second Normal Form, 2NF)
- 1NF를 만족해야 한다.
- 모든 칼럼이 **부분적 함수 종속(Partial Functional Dependency)** 이 없어야 한다. = 모든 칼럼이 **완전 함수 종속(Fully Functional Dependency)** 을 만족해야 한다.

## 제 3 정규화 (Third Normal Form, 3NF)
- 2NF를 만족해야 한다.
- **이행적 함수 종속성(Transitive Functional Dependency)** 가 존재하지 않아야 한다.
  - `A → D`, `D → B` 이면 `A → B`를 만족하게 되는 것을 의미한다.
  
## Boyce-Codd Normal Form (BCNF)
- 3NF를 만족해야 한다.
- 다음 한목 중 적어도 하나를 만족해야 한다.
  - (1) 모든 Functional Dependency는 Trivial 해야 한다.
  - (2) 모든 Functional Dependency의 Determinant Set은  슈퍼키여야 한다.

<hr>

출처
- [Database Normalization & Functional Dependency](https://ju-hy.tistory.com/104)
- [정규형 (1NF, 2NF, 3NF, BCNF)](https://rebro.kr/160)

<hr>

# 역정규화 (Denormalization)
- 하나 이상의 테이블에 중복 데이터를 추가하는 데이터베이스 최적화 기술이다.
- 이것은 우리가 관계형 데이터베이스에서 비용이 높은 조인을 피하는데 도움이 될 수 있다.
- denormalization은 'reversing normalization' 또는 'not to normalization'을 의미하지 않는다.
    > not to normalization: 기본적으로 정규화 된 스키마를 가져 와서 정규화되지 않게 만드는 프로세스를 **비정규화**라고 한다.
- 역정규화는 정규화 후 적용되는 최적화 기법이다. 
- 기존의 정규화된 데이터베이스에서는 데이터를 별도의 논리 테이블에 저장하고 중복 데이터를 최소화하려고 시도한다. 정규화는 데이터 수정 면에서 장점이 있지만, 테이블이 크면 조인을 수행하는데 불필요하게 오랜 시간을 소비할 수 있다는 단점이 있다. 

## 장점과 단점
장점
- 더 적은 조인을 수행하기 때문에 데이터 검색이 더 빠르다.

단점
- 데이터 update와 insert는 비용이 비싸다.
- 역정규화는 update와 insert 코드 작성을 더 어렵게 만들 수 있다.
- 데이터 불일치 문제
- 데이터 중복으로 더 많은 스토리지가 쓰인다.

## 역정규화 기법
자세히는 [여기](https://dataonair.or.kr/db-tech-reference/d-guide/sql/?mod=document&uid=333) 참고

<hr>

출처
- [반정규화와 성능](https://dataonair.or.kr/db-tech-reference/d-guide/sql/?mod=document&uid=333)
- [Denormalization in Databases](https://www.geeksforgeeks.org/denormalization-in-databases/)

<hr>

# MySQL에서 대량의 데이터(500만개 이상)를 Insert해야하는 경우엔 어떻게 해야할까요?
삽입 속도를 최적화하려면 많은 small 작업을 하나의 large 작업으로 결합하라.
### 1. multiple VALUES
```
INSERT INTO yourtable VALUES (1,2), (5,5), ...;
```
- 동일한 클라이언트에 여러 행을 동시에 삽입하는 경우 여러 `VALUES` 목록이 있는 `INSERT` 문을 사용하여 한 번에 여러 행을 삽입한다. 
- 단일 행 INSERT 문 여러 개를 사용하는 것보다 상당히 빠르다.
- 비어있지 않은 테이블에 데이터를 추가하는 경우 `bulk_insert_buffer_size` 변수를 조정해 데이터 삽입 속도를 빠르게 할 수 있다. 
- `INSERT ... VALUES` 문으로 하드코딩하는 경우 클라이언트가 데이터베이스 서버로 보내는 SQL 문의 길이를 제한하는 `max_allowed_packet` 변수의 값을 조절해야 한다.

### 2. LOAD DATA
```
LOAD DATA INFILE 'data.txt' INTO TABLE db2.my_table;
```
- 텍스트 파일을 이용해 데이터를 삽입한다. 
- 보통 `INSERT` 문을 사용하는 것보다 20배 더 빠르다. 

<hr>

출처
- [MySQL 8.0 Reference Manual - 8.2.5.1 Optimizing INSERT Statements](https://dev.mysql.com/doc/refman/8.0/en/insert-optimization.html)
- [MySQL - how many rows can I insert in one single INSERT statement?](https://stackoverflow.com/questions/3536103/mysql-how-many-rows-can-i-insert-in-one-single-insert-statement)
- [MySQL 에서의 Bulk Inserting 성능 향상](https://jins-dev.tistory.com/entry/MySQL-%EC%97%90%EC%84%9C%EC%9D%98-Bulk-Inserting-%EC%84%B1%EB%8A%A5-%ED%96%A5%EC%83%81)

<hr>

# 데이터베이스 파티셔닝(Database Partitioning)
## 배경
- 서비스의 크기가 점점 커지고 데이터베이스에 저장하는 데이터의 규모도 대용량화 → 기존에 사용하던 데이터베이스 시스템의 용량(storage) 한계와 성능(performance)의 저하
- VLDB(Very Large DBMS)와 같이 전체 데이터베이스가 하나의 DBMS에 다 들어가기 힘들어지는 DBMS가 자연스럽게 등장했고 하나의 DBMS가 많은 Table을 관리하다 보니 느려지는 이슈가 발생하게 되었다. 
- 이러한 이슈를 해결하기 위한 하나의 방법이 파티셔닝(Partitioning)이다.

## 파티셔닝(Partitioning)이란
- 큰 테이블이나 인덱스를 관리하기 쉬운 단위로 분리하는 방법이다.
- 단위는 파티션(partition)
- 파티션 테이블: 논리적으로는 하나의 테이블이지만 물리적으로는 여러 개의 파티션으로 나뉘어 데이터들이 각각의 세그먼트에 저장되는 테이블

## 장점과 단점
### 장점
#### 가용성(availability)
- 물리적인 파티셔닝으로 인해 전체 데이터의 훼손 가능성이 줄어들고 데이터 가용성이 향상된다.
#### 관리용이성(manageability)
- 큰 테이블들을 제거해 관리를 쉽게 해준다.
#### 성능
- 특정 DML이나 Query의 성능을 향상시킨다. 주로 대용량 Data Write 환경에서 효율적이다.
- 많은 Insert가 있는 OLTP 시스템에서 Insert 작업들을 분리된 파티션들로 분산시켜 경합을 줄인다. 
### 단점
- 테이블 간의 JOIN에 대한 비용이 증가한다.
- 테이블과 인덱스를 별도로 파티셔닝 할 수 없다. 같이 파티셔닝해야 한다.

## 파티셔닝 종류
### 수평 분할
- 하나의 테이블의 각 행을 여러 테이블에 분산시킨다.
- 샤딩(Sharding)과 동일한 개념이다.
  - 스키마를 복제한 후 샤드키를 기준으로 데이터를 나눈다.
  - 즉 샤딩은 스키마가 같은 데이터를 두 개 이상의 테이블에 나누어 저장하는 것을 말한다.
- 성능, 가용성을 위해 KEY를 기반으로 분산 저장한다.
- 데이터 조회 과정이 복잡해지기 때문에 지연 시간(latency)이 증가한다.
- 일반적으로 분산 저장 기술에서 말하는 파티셔닝은 수평 분할을 의미한다.

### 수직 분할 
- 모든 칼럼들 중 특정 칼럼들을 쪼개서 따로 저장한다. 테이블의 일부 열을 빼내는 형태로 분할한다. 스키마를 나누고 데이터가 따라 옮겨지는 것을 말한다.
- 하나의 엔터티를 2개 이상으로 분리하는 작업이다.
- 자주 사용되는 칼럼 등을 분리시켜 성능을 향상시킬 수 있다.
  - 한 테이블을 SELECT 하면 모든 컬럼을 메모리에 올리게 된다. 수직 분할을 하면 필요한 칼럼만 올려져 훨씬 많은 수의 ROW를 메모리에 올릴 수 있으니 성능상의 이점이 있다.
  - 같은 타입의 데이터가 저장되기 때문에 저장 시 데이터 압축률을 높일 수 있다.
## 파티셔닝 분할 기준
- 데이터베이스 관리 시스템은 각종 기준(분할 기법)을 제공하고 있다. 분할을 '분할 키(partitioning key)'를 사용한다.

### Range partitioning
- 분할 키 값이 범위 내에 있는지 여부로 구분한다. 연속적인 숫자나 날짜 기준으로 파티셔닝한다.
- Ex. 우편번호, 일별, 월별, 분기별 

### List partitioning
- 특정 파티션에 저장 될 데이터에 대한 명시적 제어가 가능하다.
- 분포도가 비슷하며 해당 컬럼에 대한 조건이 많이 들어오는 경우 유용하다.
- Multi-Column Partition Key 제공하기 힘들다.
- Ex. [한국, 일본, 중국 -> 아시아] [노르웨이, 스웨덴, 핀란드 -> 북유럽]

### Composite partitioning
- 파티션의 서브 파티셔닝(Sub-Partitioning)을 말한다.
- 큰 파티션에 대한 I/O 요청을 여러 파티션으로 분산할 수 있다.
- Range Partitioning 할 수 있는 컬럼이 있지만, 파티셔닝 결과 생성된 파티션이 너무 커서 효과적으로 관리할 수 없을 때 유용하다.
- Range-list, Range-hash

### Hash partitioning
- 파티션 키의 해시 값에 의한 파티셔닝이다. 균등한 데이터 분할이 가능하다.
- 특정 데이터가 어느 해시 파티션에 있는 판단이 불가능하다.
- 파티션을 위한 범위가 없는 데이터에 적합하다.

<hr>

출처
- [Database의 파티셔닝(Partitioning)이란?](https://nesoy.github.io/articles/2018-02/Database-Partitioning)
- [DB 파티셔닝(Partitioning)이란](https://gmlwjd9405.github.io/2018/09/24/db-partitioning.html)
- [파티션 테이블(Partition Table)이란 무엇인가?](https://coding-factory.tistory.com/840)

<hr>

# `WHERE` 과 `HAVING` 의 차이  
## `WHERE`
```
SELECT * From db_table [WHERE where_condition]
```
- 집계(그룹) 함수를 제외하고 MySQL이 지원하는 모든 함수 및 연산자를 사용할 수 있다.

## `HAVING`
```
SELECT * From db_table [GROUP BY {col_name | expr | position}, ... [WITH ROLLUP]] [HAVING where_condition]
```
- `WHERE` 절과 마찬가지로 선택 조건을 지정한다.
- 일반적으로 `GROUP BY` 절에 의해 형성되는 그룹의 조건을 지정한다. 쿼리 결과에는 `HAVING` 조건을 만족하는 그룹만 포함된다. `GROUP BY` 가 없는 경우 모든 행이 암시적으로 단일 집계 그룹을 형성한다. 
- `HAVING` 절은 최적화없이 항목이 전송되기 직전에 거의 마지막에 적용된다.

<hr>

출처
- [MySQL 8.0 Reference Manual - 13.2.13 SELECT Statement](https://dev.mysql.com/doc/refman/8.0/en/select.html)

<hr>

# JOIN
## JOIN이란
- 데이터베이스에서 데이터는 다수의 테이블에 나뉘어 저장되어 있다. 데이터의 중복을 제거하고 무결성을 보장하기 위해서이다. 
    ```
    데이터 무결성은 컴퓨팅 분야에서 완전한 수명 주기를 거치며 데이터의 정확성과 일관성을 유지하고 보증하는 것을 가리키며 데이터베이스나 RDBMS 시스템의 중요한 기능이다.
    ```
- 테이블별로 분리되어 있는 데이터를 연결하여 마치 하나의 테이블인 것처럼 활용하는 것이다.
- 테이블을 연결하려면 적어도 하나의 칼럼은 서로 공유되고 있어야 한다. 조인 키 칼럼이라고 한다. 보통 Primary Key나 Foreign Key로 테이블을 연결한다. 

## JOIN 종류

데이터 집합 A, B의 집합 연산에 따른 JOIN 종류
<img src="../imgs/db-sqljoins.png">

### 1. INNER JOIN

<img src="../imgs/db-innerjoin.png">

```SQL
SELECT *
   FROM TABLE_1 A
        INNER JOIN TABLE_2 B
     ON (A.COL1     = B.COL1
         AND A.COL2 = B.COL2)    
```

```SQL
SELECT * 
  FROM Orders 
      INNER JOIN Customers 
    ON Orders.CustomerID = Customers CustomerID;
```

- `INNER JOIN`은 교집합( A ∩ B ) 연산과 같다.
- JOIN KEY 칼럼의 값이 테이블 A, B에 <strong>모두 공통적으로 존재</strong>하는 경우에만 데이터 조인 
- `INNER` 키워드는 생략이 가능하다.

### 2. LEFT OUTER JOIN

<img src="../imgs/db-leftouterjoin.png">

```SQL
SELECT *
   FROM TABLE_1 A
        LEFT OUTER JOIN TABLE_2 B
     ON (A.COL1     = B.COL1
         AND A.COL2 = B.COL2)    
```

```SQL
SELECT * 
  FROM Orders 
      LEFT JOIN Customers 
    ON Orders.CustomerID = Customers.CustomerID;
```

- `LEFT OUTER JOIN`은 교집합 연산 결과와 차집합 연산 결과를 합친 것 ( (A ∩ B) ∪ (A - B) )와 같다.
- JOIN KEY 칼럼의 값이 테이블 A, B에 모두 공통적으로 존재하는 데이터와 + 기준 테이블(그림에서는 A)의 데이터를 조인한다. 
- `OUTER` 키워드는 생략 가능하다.

### 3. RIGHT OUTER JOIN

<img src="../imgs/db-rightouterjoin.png">

```SQL
SELECT *
   FROM TABLE_1 A
        RIGHT OUTER JOIN TABLE_2 B
     ON (A.COL1     = B.COL1
         AND A.COL2 = B.COL2)    
```

```SQL
SELECT * 
  FROM Orders 
      RIGHT JOIN Customers 
    ON Orders.CustomerID = Customers.CustomerID;
```

- `RIGHT OUTER JOIN`은 `LEFT OUTER JOIN` 와 유사하다. 기준 테이블이 반대일 뿐이다.
- JOIN KEY 칼럼의 값이 테이블 A, B에 모두 공통적으로 존재하는 데이터와 + 기준 테이블(그림에서는 B)의 데이터를 조인한다. 
- `OUTER` 키워드는 생략 가능하다.

### 4. FULL OUTER JOIN

<img src="../imgs/db-fullouterjoin.png">

```SQL
SELECT *
   FROM TABLE_1 A
        FULL OUTER JOIN TABLE_2 B
     ON (A.COL1     = B.COL1
         AND A.COL2 = B.COL2)
```

```SQL
SELECT * 
  FROM Orders 
      FULL JOIN Customers 
    ON Orders.CustomerID = Customers.CustomerID;
```

MySQL에서 FULL OUTER JOIN 하는법
```SQL
(SELECT * FROM employees.employees A LEFT JOIN employees.titles B ON A.emp_no = B.emp_no) UNION (SELECT * FROM employees.employees A LEFT JOIN employees.titles B ON A.emp_no = B.emp_no)
```

- `FULL OUTER JOIN`은 합집합 연산 결과와 같다.
- JOIN KEY 칼럼의 값이 테이블 A, B에 모두 공통적으로 존재하는 데이터 + 한쪽 테이블에 존재하는 데이터를 조인한다.


<hr>

출처
- [테이블 조인 종류(Table Join Type)](https://sparkdia.tistory.com/17)
- [SQL - JOIN문, JOIN 종류 (Inner Join,Natural Join,Outer Join,Cross Join)](https://doh-an.tistory.com/30)
- [JOIN 종류와 사용법](https://theplace.tistory.com/m/24)

<hr>

# Scale Out & Scale Up
## DB 확장
- 수평적으로 부하를 분산하는 **스케일아웃**(Scale Out)
- 해당 서버의 용량 자체를 올리는 **스케일업**(Scale Up)

## 1) Scale Out
- 기존 서버만으로 용량이나 성능의 한계에 도달했을 때, 비슷한 사양의 서버를 추가로 연결한다.
- 기대효과
  - 처리할 수 있는 데이터 용량이 증가한다.
  - 기존 서버의 부하를 분담해 성능이 향상한다.
- '수평 스케일링(horizontal scaling)'로 불리기도 한다.
- 서버가 여러 대가 되기 때문에 각 서버에 걸리는 부하를 균등하게 해주는 '**로드밸런싱**'이 필수적으로 동반되어야 한다.
- Ex. **OLAP(Online Analytical Processing)** 환경
  - 빅데이터의 데이터 마이닝이나 검색엔진 데이터 분석 처리를 위해 대량의 데이터 처리와 복잡한 쿼리가 이루어지기 때문이다.

### 장점
- 확장성
  - 하나의 장비에서 처리하던 일을 여러 장비에 나눠서 처리함.
  - 수평 확장이며, 지속적 확장 가능.
- 장애
  - 서버 한 대가 장애로 다운되더라도 다른 서버로 서비스 제공이 가능.
- 비용
  - 저렴한 서버를 사용하므로 일반적으로 **비용부담이 적음.**

### 단점
- 관리
  - 여러 노드를 연결해 병렬 컴퓨터 환경을 구성하고 유지하려면 아키텍처에 대한 높은 이해도가 요구.
  - 서버의 수가 늘어날수록 관리가 힘들어짐.
- 로드밸런싱
  - 여러 노드에 부하를 균등하게 분산시키기 위해 로드 밸런싱(load balancing)이 필요.

## 2) Scale Up
- 기존의 서버를 보다 높은 사양으로 업그레이드하는 것.
- 하드웨어적인 예를 들면, 성능이나 용량 증강을 목적으로 하나의 서버에 디스크를 추가하거나 CPU나 메모리를 업그레이드시키는 것이다.
- '수직 스케일링(vertical scaling)'로 불리기도 한다.
- 소프트웨어적인 예로는 AWS의 EC2 인스턴스 사양을 micro에서 small, small에서 medium 등으로 높이는 것을 생각하면 된다.
- Ex. **OLTP(Online Transaction Processing)** 환경
  - 온라인 금융거래와 같이 워크플로우 기반의 빠르고 정확한 단순한 처리가 필요하다.

### 장점
- 관리
  - 추가적인 네트워크 연결 없이 용량을 증강할 수 있어서 관리 비용이나 운영 이슈가 적음.
  - 사양만 올리면 되는 것이기 때문에 비교적 쉬움.

### 단점
- 확장성
  - CPU 변경, RAM 추가 등으로 하드웨어 장비의 성능을 높임.
  - 수직 확장이며, 성능 확장에 한계가 있음.
- 비용
  - 성능 증가에 따른 비용 증가폭이 큼.
- 장애
  - 한 대의 서버에 부하가 집중되어 장애 영향도가 큼.

<hr>

출처
- [Scale-up과 Scale-out에 대해 알아보자!](https://tecoble.techcourse.co.kr/post/2021-10-12-scale-up-scale-out/)

<hr>

# 트래픽이 높아질 때, DB는 어떻게 관리를 할 수 있을까요?
- 우선 DB Replication & Cache을 고려해볼 수 있다. 
- 그래도 느리다면, DB 파티셔닝 또는 샤딩을 고려해본다. 

## DB Replication & Cache
### DB Replication
- 쓰기용 DB와 읽기용 DB로 나누는 것이다.
- 데이터 일관성을 위해 쓰기용 DB는 한개만 두고, 읽기용 DB `Read Replica`를 만들어준다. `Read Replica`는 쓰기용 DB를 빠르게 복사하여 데이터 동기화를 하는 특징이 있다.
  - AWS Aurora 서비스를 이용하면 손쉽게 다룰 수 있다.
- 만약 쓰기용 DB의 서버가 죽는다면 자동으로 읽기용 DB가 쓰기용 DB로 승격되는 Master/Slave 구조로도 사용된다. master DB는 Insert, Update, Delete 기능을 수행하고, Slave DB는 Select 기능을 수행한다.
- 하지만 DB 커넥션은 비싸기 때문에, 자주 바뀌지 않는 데이터라면 DB에 접근하는 대신 메모리 캐시를 이용해서 빠르게 가져올 수 있다. 

### Cache
- 먼저 메모리 캐시에 데이터가 있는지 확인 후 데이터가 없다면 DB를 참고하는 방식으로 쿼리를 줄일 수 있다.
- 기존 데이터가 수정되거나 새롭게 쓸 경우 Redis에 업데이트하여 데이터의 일관성을 지킬 수 있다.

## DB 파티셔닝
- 큰 테이블이나 인덱스를 관리하기 쉬운 크기로 분리하는 방법이다.
### 장점
- **가용성(Avaliability)**
  - 물리적인 노드 분리에 따라 전체 DB 내의 데이터 손상 가능성이 줄어들고, 데이터 가용성이 향상된다.
- **관리용이성(Manageability)**
  - 큰 테이블을 제거하여 관리를 쉽게 할 수 있다.
- **성능(Performance)**
  - 특정 DML과 Query 성능을 향상시키며 대용량 데이터 write 환경에서 효율적이다.
  - insert가 많은 OLTP 환경 시스템에서 insert 작업들을 로드 밸런싱을 통해 분산시켜 성능 상의 이점이 있다.

### 단점
- **테이블 간 join 비용 증가**
- **파티션 제약**
  - 테이블과 인덱스를 별도로 파티션 할 수 없다.

## DB 샤딩 
- DBMS 한 개로 처리할 수 있는 데는 한계가 있으므로 데이터베이스 여러 개를 사용하는 방식으로 데이터 조회 한계를 극복해야 한다. 
- 같은 테이블 스키마를 가진 데이터들을 다수의 DB에 분산하여 저장하는 '수평 파티셔닝(horizontal partitioning)' 방법으로 해당 테이블의 인덱스 크기를 줄이고, 작업 동시성을 늘리는 방법이다.
- 한 샤드에 데이터가 몰리지 않도록 기준을 잘 삼아야 한다. 기준을 삼을 때는 두 가지 고려야 할 것이 있다.
  - (1) 각 샤드의 데이터 용량이 비슷하고 각 사드에서 비슷한 속도로 데이터가 증가해야 한다.
  - (2) 각 샤드에 대한 초당 연결 수가 거의 동일하도록 노력해야한다. 
- 잘 쪼개진 DB는 데이터가 분산되었기 때문에 쿼리 퍼포먼스가 증가한다. 
- 하지만 코드가 복잡해지거나 테이블 join 등 여러 데이터를 동시에 다루기 어렵다는 단점이 있다. 필요한 경우에만 잘 사용해야 한다.

<hr>

출처
- [파티셔닝(Partitioning)과 샤딩(Sharding)](https://seokbeomkim.github.io/posts/partition-and-sharding/#fn:2)
- [높은 트래픽 유저를 수용하는 확장 가능한 시스템 설계 with AWS](https://pypy.dev/web/%EB%86%92%EC%9D%80-%ED%8A%B8%EB%9E%98%ED%94%BD-%EC%9C%A0%EC%A0%80%EB%A5%BC-%EC%88%98%EC%9A%A9%ED%95%98%EB%8A%94-%ED%99%95%EC%9E%A5-%EA%B0%80%EB%8A%A5%ED%95%9C-%EC%8B%9C%EC%8A%A4%ED%85%9C-%EC%84%A4%EA%B3%84-with-aws/)

<hr>

# 스키마(Schema)
## 정의
- 한마디로 정의하면 '데이터의 구조' 또는 '데이터베이스의 설계'를 의미한다.
- 데이터베이스(DB)의 구조와 제약조건에 관한 전반적인 명세를 기술한 메타데이터의 집합이다.
- 스키마는 객체의 특성을 나타내는 **속성(Attribute)**, 속성들의 집합으로 이루어진 **개체(Entity)**, 개체 사이에 존재하는 **관계(Relation)** 에 대한 정의와 이들이 유지해야 할 **제약조건** 을 기술한 것.


## 특징
- **데이터 사전(Data Dictionary)** 에 저장된다.
  > 데이터 사전(Data Dictionary): 데이터베이스 자원을 효율적으로 관리하기 위한 다양한 정보를 저장하는 시스템 테이블이다. 데이터 딕셔너리의 내용은 데이터베이스 안의 모든 객체에 대한 정의로서 객체들에 할당된 공간과 사용된 공간, 무결성 제약 조건, 사용자 정보, 사용자에게 부여된 권한과 역할(ROLE) 정보, 감사 정보 등 다양한 정보를 알려준다.
- 현실세계의 특정한 한 부분의 표현으로써 **특정 데이터 모델**을 이용하여 만들어진다.
- 시간에 따라 불변인 특성을 갖는다.
- 데이터의 구조적 특성을 의미하며, 인스턴스에 의해 규정된다.

## 스키마의 3계층
<img src="../imgs/database-schema.jpg">

스키마는 하나의 데이터베이스를 ***사용자의 관점에 따라*** 세 개의 단계로 나누는데 이를 '스키마의 3계층'이라고 한다.
- 외부 스키마
- 개념 스키마
- 내부 스키마 

<HR>

### 1. 외부 스키마 (External Schema) = 사용자 뷰(View)
- 개별 사용자들의 입장에서 필요한 **데이터베이스의 논리적 구조**를 정의한 것이다.
- '**서브 스키마**' 또는 '**사용자 뷰(view)**' 라고도 부른다.
- **동일한 데이터에 대해, 서로 다른 관점을 정의할 수 있도록 허용한다.**
- 하나의 데이터베이스 시스템에는 여러개의 외부 스키마가 존재 가능하며, 하나의 외부 스키마를 여러개의 응용 프로그램이나 사용자가 공용할 수도 있다.
- 일반 사용자는 SQL를 이용하여 DB를 쉽게 사용할 수 있다.
- 응용 프로그래머는 C, JAVA 등의 언어를 사용하여 DB에 접근한다.


### 2. 개념 스키마 (Conceptual Schema) = 전체적인 뷰(View)
- 데이터베이스의 전체 조직에 대한 논리적인 구조로, 물리적인 구현은 고려하지 않는다.
- **모든 응용시스템과 사용자가 필요로 하는 데이터를 통합한 조직 전체의 데이터베이스로 하나만 존재한다.**
- **개체 간의 관계(Relation)** 및 **무결성 제약 조건**에 대한 **명세**를 정의한다.
  > 무결성 제약 조건: 데이터베이스의 정확성, 일관성을 보장하기 위해 저장, 삭제, 수정 등을 제약하기 위한 조건
- 일반적으로 이야기하는 '스키마'가 개념스키마다.


### 3. 내부 스키마 (Internal Schema) = 저장 스키마(Storage Schema)
- 물리적 저장장치의 입장에서 본 데이터베이스 구조이다.
- 구체적으로 개념 스키마를 디스크 기억장치에 물리적으로 구현하기 위한 방법을 기술한 것이다. 주된 내용은 실제로 저장될 내부 레코드 형식, 내부 레코드의 물리적 순서, 인덱스의 유/무 등에 관한 것.
  - 즉 **기계적 처리를 하는 일련의 과정**을 의미한다.
- 물리적 설계이기 때문에 사용자들이 함부로 접근할 수 없고 DBA가 직접 관리한다.


<HR>

출처
- [스키마(Schema)란?](https://www.hedleyonline.com/ko/blog/%EC%8A%A4%ED%82%A4%EB%A7%88%EC%97%90-%EB%8C%80%ED%95%9C-%EB%AA%A8%EB%93%A0%EA%B2%83-2022/)
- [스키마(Schema)란?](https://iingang.github.io/posts/DB-schema/)

<HR>

# DBCP (Database Connection Pool)
## DBCP 등장 배경
### 웹 어플리케이션을 지탱하는 WAS에서 DB 서버에 접근을 시작하고 데이터를 가져오기까지의 단계에서 가장 비용이 많이 드는 부분이 어디일까?

```java
String driverPath = "net.sourceforge.jtds.jdbc.Driver";
String address = "jdbc:jtds:sqlserver://IP/DB";
String userName = "user";
String password = "password";
String query = "SELECT ... where id = ?";
try {
  Class.forName(driverPath);
  Connection connection = DriverManager.getConnection(address, userName, password);
  PreparedStatement ps = con.prepareStatement(query);
  ps.setString(1, id);
  ResultSet rs = get.executeQuery();
  // ....
} catch (Exception e) { }
} finally {
  rs.close();
  ps.close();
}
```
1. DB 서버 접속을 위해 JDBC 드라이버를 로드한다.
2. DB 접속 정보와 `DriverManager.getConnection()`을 통해 `DB Connection` 객체를 얻는다.
3. `Connection` 객체로부터 쿼리를 수행하기 위한 `PreparedStatement` 객체를 받는다.
4. `executeQuery` 를수행하여 그 결과로 `ResultSet` 객체를 받아서 데이터를 처리한다.
5. 처리가 완료되면 처리에 사용된 리소스들을 `close` 하여 반환한다.

위의 예제를 통해서 Database에서 원하는 데이터를 얻어 오기 까지의 과정에서 처리 시간이 0.1초가 소요된다면, 어느 과정에서 비용이 가장 많이 발생할까?

### A. 가장 느린 부분은 웹 서버에서 물리적으로 DB 서버에 최초로 연결되어 `Connection` 객체를 생성하는 부분이다.

## DBCP 목적 
<img src="../imgs/db-connectionthread.png">

웹 어플리케이션은 HTTP 요청에 따라 Thread를 생성하게 되고 대부분의 비즈니스 로직은 DB 서버로부터 데이터를 얻게 된다. 만약 위와 같이 모든 요청에 대해 `DB 접속을 위한 Driver를 로드하고 Connection 객체를 생성하여 연결한다면` 물리적으로 DB 서버에 지속적으로 접속해야 될 것이다.

이러한 상황에서 DB Connection 객체를 생성하는 부분에 대한 비용과 대기시간을 줄이고, 네트워크 연결에 대한 부담을 줄일 수 있는 방법이 있는데, 바로 DBCP (Database Connection Pool)이다.

## DBCP 란
DBCP는 HTTP 요청에 매번 위의 `1-5 단계`를 거치지 않기 위한 방법이다. Connection Pool을 이용하면 다수의 HTTP 요청에 대한 Thread를 효율적으로 처리할 수 있다.
> WAS가 실행될 때 애플리케이션에서는 Connection Pool 라이브러리를 통해 Connection Pool 구현체를 사용할 수 있는데, Apache의 Commons DBCP가 오픈소스 라이브러리로 제공되고 있다.
> 
> http://commons.apache.org/

### Connection Pool 구현체의 역할
1. WAS가 실행되면서 미리 일정량의 DB Connection 객체를 생성하고 ***Pool*** 이라는 공간에 저장해 둔다.
2. HTTP 요청에 따라 필요할 때 Pool에서 Connection 객체를 가져다 쓰고 반환한다.
3. 이와 같은 방식으로 HTTP 요청 마다 DB Driver를 로드하고 물리적인 연결에 의한 Connection 객체를 생성하는 비용이 줄어들게 된다.
   <img src="../imgs/db-connectionpool.gif">

<hr>

출처
- [DB Connection Pool에 대한 이야기](https://www.holaxprogramming.com/2013/01/10/devops-how-to-manage-dbcp/)

<hr>

# 테이블의 데이터를 읽는 방식: Table Full Scan, Index Range Scan
테이블의 데이터를 읽는 방식으로는 크게 Table Full Scan, Index Range Scan으로 나뉜다.
## Table FUll Scan
테이블 풀 스캔은 테이블에 존재하는 모든 데이터를 읽어 가면서 조건에 맞으면 결과로서 추출하고 조건에 맞지 않으면 버리는 방식이다. Oracle의 경우, 테이블의 고수위 마크(HWM, High Water Mark)아래의 모든 블록을 읽는다.
> 고수위 마크(HWM, High Water Mark): 테이블에 데이터가 쓰여졌던 블록 상의 최상위 위치 (현재는 지워져서 데이터가 존재하지 않음을 의미한다.)

일반적으로 블록들은 서로 인접되어 있기 때문에, 테이블 풀 스캔은 한 번의 I/O에 여러 블록을 옮겨온다. 즉 한 번의 I/O에 데이터를 다중 블록 단위로 메모리에 가져오기 때문에, Row 당 소요되는 입출력 비용이 인덱스 스캔에 비해 적다. 메모리에 옮겨진 블록들은 순차적으로 읽힌다.

결과를 찾기 위해 모든 블록을 읽은 것이다. 그래서 풀 테이블 스캔으로 읽은 블록들은 메모리에서 곧 제거될 수 있도록 관리된다. 

테이블 풀 스캔이 발생하는 경우는 다음과 같다.
- 적용 가능한 인덱스가 없는 경우
  - (1) 결합 인덱스의 선두 칼럼이 존재하지 않을 때
  - (2) 적용할 인덱스가 있지만 칼럼 가공 or 연산하여 그 인덱스를 사용할 수 없을 때
- 넓은 범위의 데이터 액세스
  - 인덱스 처리 범위가 넓어 풀 테이블 스캔이 더 적은 비용이 들 경우
- 소량의 테이블 액세스
  - 고수위 마크(HWM) 내에 있는 블록이 `DB_FILE_MULTIBLCOK_READ_COUNT` 이내에 있는 경우 테이블 풀 스캔을 할 수도 있다.
  > DB_FILE_MULTIBLCOK_READ_COUNT: 한번에 I/O 작업으로 읽어들이는 최대 블럭 수
- 병럴처리 액세스
  - 병렬처리는 풀 테이블 스캔을 더욱 효과적으로 수행하게 되므로 병렬처리로 수행되는 실행계획을 수립할 때는 항상 풀 테이블 스캔을 사용한다.
- 힌트를 적용한 경우
  - Full 힌트를 사용했을 때. 단, Full 힌트가 적절하지 않다면 옵티마이저는 이를 무시할 수 있다.


## Index Range Scan란
쿼리 예제
```SQL
SELECT * FROM employees WHERE first_name BETWEEN 'Ebbe' AND 'Gad';
```

인덱스 레인지 스캔은 **검색해야 할 인덱스의 범위**가 결정됐을 떄 사용하는 방식이다.

루트 노드에서부터 비교를 시작해 브랜치 노드를 거치고 최종적으로 리프노드까지 찾아들어가야만 비로소 필요한 레코드의 시작 지점을 찾을 수 있다. 일단 시작해야 할 위치를 찾으면 그때부터는 **리프 노드의 레코드만 순서대로 읽으면 된다.** 이처럼 차례대로 쭉 읽는 것을 스캔이라고 표현한다.

만약 스캔하다가 리프 노드의 끝까지 읽으면 **리프 노드 간의 링크를 이용해 다음 리프 노드를 찾아서 다시 스캔한다.** 그리고 최종적으로 스캔을 멈춰야 할 위치에 다다르면 **지금까지 읽은 레코드를 사용자에게 반환하고 쿼리를 끝낸다.**


중요한 것은 어떤 방식으로 스캔하든 관계없이, 해당 인덱스를 구성하는 **칼럼의 정순 또는 역순으로 정렬된 상태로 레코드를 가져온다**는 것이다. 이는 별도의 정렬 과정이 수반되는 것이 아니라 **인덱스 자체의 정렬 특성** 때문에 자동으로 그렇게 된다.

또 한 가지 중요한 것은 인덱스의 리프 노드에서 검색 조건에 일치하는 건들은 데이터 파일에서 레코드를 읽어오는 과정이 필요하다는 것이다. 이때 리프 노드에 저장된 레코드 주소로 데이터 파일의 레코드를 읽어오는데, **레코드 한 건 한 건 단위로 랜덤 I/O가 한 번씩 일어난다.** 그래서 인덱스를 통해 데이터 레코드를 읽는 작업은 비용이 많이 드는 작업으로 분류된다. 그리고 인덱스를 통해 읽어야 할 데이터 레코드가 **20~25%** 를 넘으면 **인덱스를 통한 읽기보다 테이블의 데이터를 직접 읽는 것이 더 효율적**인 처리 방식이 된다.

### 3단게 과정
1. 인덱스에서 조건을 만족하는 값이 저장된 위치를 찾는다. 이 과정을 인덱스 탐색(Index seek)이라고 한다.
2. 1번에서 탐색된 위치부터 필요한 만큼 인덱스를 차례대로 쭉 읽는다. 이 과정을 인덱스 스캔(Index scan)이라고 한다. (1번과 2번을 합쳐서 인덱스 스캔으로 통칭하기도 한다.)
3. 2번에서 읽어 들인 키와 레코드 주소를 이용해 레코드가 저장된 페이지를 가져오고, 최종 레코드를 읽어온다.

쿼리가 필요로 하는 데이터에 따라 3번 과정은 필요하지 않을 수도 있는데, 이를 커버링 인덱스라고 한다. 커버링 인덱스로 처리되는 쿼리는 디스크의 레코드를 읽지 않아도 되기 때문에 랜덤 읽기가 상당히 줄어들고 성능은 그만큼 빨라진다. 

<HR>

출처
- [Full Table Scan 이해하기](https://yunhyeonglee.tistory.com/17)
- [인덱스 스캔과 전체 테이블 스캔](https://hoon93.tistory.com/53)

<hr>

# Clustering & Replication
일반적인 데이터베이스 구조는 데이터베이스 서버 1개, 스토리지 1개로 구성된다. 이렇게 구성된 데이터베이스는 서버가 제대로 동작하지 않으면 먹통이 될 것이다. 이러한 문제점을 해결하기 위해 가장 먼저 떠오르는 방법은 서버를 하나 더 늘리는 것이다. 이렇게 서버를 여러개로 구성하는 것을 Clustering이라 한다.
## Clustering
- Clustering은 데이터베이스 서버를 2대 이상, 스토리지를 1대로 구성하는 형태이다. 
  - 이때 DB 서버는 서로 다른 인스턴스에서 작동한다.
- 모든 NoSQL이 그런 건 아니지만 MongoDB나 Redis 처럼 많이 쓰이고 성숙한 NoSQL 제품들은  기능이 자체 탑재되어 있고 여러 클러스터링 전략 중 하나를 간단한 설정만으로 사용할 수 있다. 

### Active & Active 
- DB 서버 2대를 모두 `Active`한 상태로 운영.
- DB 서버 1대가 죽더라도 DB 서버 1대는 살아있어, 서비스는 정상적으로 작동한다. 
- 장점
  - 여러 대의 DB 서버로 트래픽을 분산하여, CPU와 Memory에 대한 부하가 적어짐
- 단점
  - DB 서버들이 DB 스토리지를 공유하여, DB 스토리지에 병목이 생김.

### Active & Stand-By 
- DB 서버 1대는 `Active`, 다른 1대는 `Stand-by` 상태로 운영.
-  `Stand-By `상태의 서버는 `Active` 상태의 서버에 문제가 생겼을 때, failover 하여 장애에 대응할 수 있다. 
    > failover: 컴퓨터 서버, 시스템, 네트워크 등에서 이상이 생겼을 때 예비 시스템으로 자동 전환되는 기능
-  장점
   -  DB 스토리지에 대한 병목 현상 해결
- 단점
  - failover가 발생하는 시간 동안은 작업 불가
  -  DB 서버 비용은 Active & Active과 동일하나, 가용률은 이전에 비해 대략 1/2로 줄어든다.

Clustering은 DB 스토리지를 1대만 사용하기 때문에 DB 스토리지 단일 장애점이 될 수 있다. DB 스토리지에 문제가 발생하면, 데이터를 복구할 수 없는 치명타가 생긴다.  그래서 이번에는 스토리지도 여러개 가지는 Replication에 대하여 알아보자.


## Replication
- 두 개 이상의 DBMS 시스템을 Master/Slave로 나눠서 동일한 데이터를 저장하는 방식이다. 각 DB 서버가 각자 DB 스토리지를 갖고 있는 형태.
- 노드가 다른 노드가 정상적으로 작동하고 있음을 확인하는 신호를 hearbeat라고 한다.

### Master & Slave
- Master DB에는 데이터의 수정사항을 반영만 하고 Replication 하여 Slave DB에 실제 데이터를 복사한다.
-  Master는 쿼리 로그에 해당하는 **바이너리 로그(binary log)** 를 남기는데, 이를 Slave가 비동기적으로(asynchronous) 읽어가서 자신의 데이터베이스에 반영한다. 
-  장점
   -  Master DB에 Insert/Update/Delete를 하고, Slave DB에 Select를 하는 방식으로 각각 DB에 트래픽을 분산할 수 있다.
- 단점
  -  각각의 서로 다른 서버로 운영하다보니 버전을 관리해야한다. 이때 Master와 Slave의 데이터베이스 버전을 동일하게 맞춰주는 것이 좋다. 버전이 다를 경우 적어도 Slave 서버가 상위 버전이어야 한다.
  -  Master에서 Slave로 비동기 방식으로 데이터를 동기화하기 때문에 일관성 있는 데이터를 얻지 못할 수도 있다. 동기화 방식으로 Replication 할 수 있지만 이럴 경우 속도가 느려진다는 문제점이 있다. 
  - 마지막으로 Master 서버가 다운이 될 경우 복구 및 대처가 까다롭다는 단점이 있다.

### 로그기반 복제(Binary log)
- **Statement** Based : SQL문장을 복사하여 진행
  - issue : SQL에 따라 결과가 달라지는 경우(Timestamp, UUID, …)
- **Row** Based : SQL에 따라 변경된 Row 라인만 기록하는 방식
  - issue : 데이터가 많이 변경된 경우 데이터 커질 수 밖에 없다.
- **Mixed** : 기본적으로 Statement Based로 진행하면서 필요에 따라 Row - Based를 사용한다.

## Clustering vs Replication
- Clustering은 데이터베이스 서버만을 늘려 처리하는 것이고,
- Replication은 서버와 스토리지 모두 복제하여 구성하는 방법이다.

<HR>

출처
- [Replication과 Clustering](https://tecoble.techcourse.co.kr/post/2021-09-18-replication_clustering/)
- [DB Replication을 구성한 이유](https://2021-pick-git.github.io/why-db-replication-is-set-up/)
- [Database의 리플리케이션(Replication)이란?](https://nesoy.github.io/articles/2018-02/Database-Replication)
- [데이터베이스 REPLICATION](https://www.joinc.co.kr/w/man/12/replication)

<HR>

# PostgreSQL
## 특징
- **ACID 및 트랜잭션 지원**
- **유연한 Full-text search 기능**
- **NoSQL 및 다양한 데이터 유형 지원**
  - 기본 요소: 정수, 숫자, 문자열, 부울
  - 구조화: 날짜/시간, 배열, 범위/다중 범위, UUID
  - 문서: JSON/JSONB, XML, 키-값(Hstore)
  - 기하학: 점, 선, 원, 다각형
  - 사용자 정의: 복합, 사용자 정의 유형
- **단순히 RDBMS가 아닌 ORDBMS**: 객체지향 데이터베이스 시스템과 관계형 데이터베이스 시스템을 기반으로하며 복잡한 객체가 중심 역할을 하는 DBMS이다. ORBMS는 엄격한 관계형 모델에 맞지 않는 데이터를 처리할 때 탁월하다.
  - 테이블 상속, 함수 오버로딩 등의 기능
- **복잡한 쿼리**: 복잡한 읽기-쓰기 작업에 좋다. 다만 읽기가 많은 경우 다른 DB보다 성능이 떨어진다.
- **초대형 데이터베이스 관리용으로 설계**: 데이터베이스의 크기에 제한을 두지 않는다. 
- **다중 버전 동시성 제어(MVCC)**: 사용자는 갱신/변경된 데이터를 다른 그 이전 데이터와 버전을 달리해 관리하고, 이를 기반으로 일관성을 유지한다. 따라서 데이터와 통신해야 할 때마다 읽기-쓰기 잠금을 할 필요가 없으므로 효율성이 향상한다.
  - 기존 데이터는 삭제 표시, 더 이상 필요하지 않을 때 데이터 정리

<HR>

출처
- [PostgreSQL과 MySQL 비교: 주요 차이점](https://www.integrate.io/ko/blog/postgresql-vs-mysql-the-critical-differences-ko/)
- [PostgreSQL이란? 및 설치 방법](https://learning-e.tistory.com/25)
- [한눈에 살펴보는 PostgreSQL](https://d2.naver.com/helloworld/227936)
- [MVCC 구조와 이해](https://mozi.tistory.com/561)

<HR>

# 쿼리의 성능을 확인하기 위해 어떤 쿼리문을 작성해야 할까요?
##  `EXPLAIN ANALYZE`  명령
- MySQL 8.0.18 버전부터는 쿼리의 실행 계획과 단계별 소요된 시간 정보를 확인할 수 있는 `EXPLAIN ANALYZE` 기능이 추가됐다. 
- 물론 `SHOW PROFILE ` 명령으로 어떤 부분에서 시간이 많이 소요되는지 확인할 수 있지만  `SHOW PROFILE ` 명령의 결과는 실행 계획의 단계별로 소요된 시간 정보를 보여주지 않는다.'
- `EXPLAIN ANALYZE`  명령의 결과에는 단계별로 실제 소요된 시간(actual time)과 처리한 레코드 건수(rows), 반복 횟수(loops)가 표시된다. 

<hr>

출처
- Real MySQL 8.0 책 

<HR>

# 인덱스(Index)
## 인덱스란 
- 데이터베이스의 테이블에 대한 검색 속도를 향상시켜주는 자료구조이다.
- 테이블의 특정 칼럼에 인덱스를 생성하면,
- 해당 칼럼의 데이터를 정렬한 후 별도의 메모리 공간에 key-value 형태로 데이터의 물리적 주소와 함께 저장된다. 
  - key: 칼럼의 데이터
  - value: 칼럼의 데이터의 물리적 주소

## 장점
- 테이블을 검색하는 속도와 성능이 향상된다. 
- where 문으로 특정 조건의 데이터를 찾기 위해 '풀 테이블 스캔' 필요 없이, 인덱스를 이용하면 데이터들이 정렬되어 있기 때문에 빠르게 찾을 수 있다.
- ORDER BY 문이나 MIN/MAX 문에  빠르게 수행 가능하다.

## 단점 
- 인덱스를 관리하기 위한 추가 작업이 필요하다. 
  - INSERT: 새로운 데이터에 대한 인덱스 추가
  - DELETE: 삭제하는 데이터의 인덱스를 사용하지 않는다는 작업 수행
  - UPDATE: 기존의 인덱스를 사용하지 않음 처리와 갱신된 데이터에 대한 인덱스 추가
- 추가 저장 필요
- 잘못 사용하는 경우 오히려 검색 성능 저하
- 특히 UPDATE의 경우 인덱스를 제거하는 것이 아니라, '사용하지 않음'으로 처리하고 남겨두기 때문에 저장 공간이 낭비될 수 있다.

## 인덱스 종류
1. 클러스터형 인덱스(Clustered Index)
2. 보조 인덱스(세컨더리 인덱스, 비클러스터형 인덱스, Nonclustered Index)


<HR>

출처
- [인덱스(Index) - (1) 개념, 장단점, B+Tree 등](https://rebro.kr/167)

<HR>

# Elasticsearch
## Elasticsearch란
- Apache Lucene 기반의 Java 오픈소스 분산형 RESTful 검색 및 분석 엔진이다.
- 방대한 양의 데이터에 대해 실시간으로 저장과 검색 및 분석 등의 작업을 수행햘 수 있다.
- 형 데이터, 비정형 데이터, 지리 데이터 등 모든 타입의 데이터를 처리 가능하다.
- Elasticsearch는 JSON 문서(Document)로 데이터를 저장하기 때문이다.

## Elasticsearch는 언제 사용하는가?
- Elasticsearch는 단독 검색을 위해 사용하거나, ELK(Elasticsearch & Logstash & Kibana) 스택을 기반으로 사용한다.
- **Filebeat**
  - 로그를 생성하는 서버에 설치해 로그를 수집합니다.
  - Logstash 서버로 로그를 전송합니다.
- **Logstash**
  - 로그 및 트랜잭션 데이터를 수집과 집계 및 파싱하여 Elasticsearch로 전달합니다.
  - 정제 및 전처리를 담당합니다.
- **Elasticsearch**
  - Logstash로부터 전달받은 데이터를 저장하고, 검색 및 집계 등의 기능을 제공합니다.
- **Kibana**
  - 저장된 로그를 Elasticsearch의 빠른 검색을 통해 가져오며, 이를 시각화 및 모니터링하는 기능을 제공합니다.
- ELK의 목적은 주로 로드밸런싱되어 있는 WAS의 흩어져 있는 로그를 한 곳으로 모으고, 원하는 데이터를 빠르게 검색한 뒤 시각화해 모니터링하기 위해 사용한다.

## 구조
- Elasticsearch는 클러스터로 구성되며, 클러스터 안에 노드, 노드 안에 인덱스, 인덱스 안에 샤드, 샤드 안에 세그먼트로 구성된다.
- **클러스터**
  - Elasticsearch에서 가장 큰 시스템 단위를 의미하며, 최소 하나 이상의 노드로 이루어진 노드의 집합이다.
  - 서로 다른 클러스터는 데이터의 접근, 교환을 할 수 없는 독립적인 시스템으로 유지된다.
  - 여러 대의 서버가 하나의 클러스터를 구성할 수 있고, 한 서버에 여러 클러스터가 존재할 수 있다.
- **노드**
  - 하나의 물리적 서버를 사용하는 Elasticsearch의 인스턴스이다.
  - 노드를 여러 개 묶어서 하나로 사용할 경우, 여러 Elasticsearch의 인스턴스를 하나의 클러스터라고 부른다.
  - 클러스터에 포함된 단일 서버로서 데이터를 저장하고 클러스터의 색인화 및 검색 기능에 참여한다.
  - 노드는 역할에 따라 Master-eligible, Data, Ingest, Tribe 노드로 구분할 수 있다.
  - 노드 내에 인덱스를 논리적으로 나눈 단위인 '샤드'가 저장된다.
- **인덱스**
  - RDBMS에서 데이터베이스와 대응하는 개념이다.
  - 데이터를 하나 생성한다고 하면, 인덱스 내에 하나의 '문서(document)'가 생성된다.
  - 문서는 JSON 형태로 저장되며, JSON 에서의 key를 '필드(field)'라고 한다.
  - 인덱스를 생성하면, 인덱스의 데이터는 여러 샤드에 나누어져 저장된다.
- **샤드**
  - 인덱스 내부에는 색인된 데이터들이 존재하는데, 이 데이터들을 하나로 뭉쳐서 존재하지 않고 물리적 공간에 여러 부분으로 나뉘어 존재한다. 이러한 부분을 샤드라고 한다.
  - 스케일 아웃을 위해 하나의 인덱스를 여러 샤드로 쪼갠 것이다.
- **세그먼트**
  - Elasitcsearch의 빠른 문서 검색을 위해 설계된 자료구조이며, 샤드의 데이터를 가지고 있는 물리적인 파일이다.
  - 각 샤드는 다수의 세그먼트로 구성되어 있으므로 검색 요청을 분산 처리하여 훨씬 효율적인 검색이 가능하다.
  - 샤드에서 검색 시, 먼저 각 세그먼트를 검색하여 결과를 조합한 후 최종 결과를 해당 샤드의 결과로 반환하게 된다. 
  - 이때 세그먼트는 내부에 색인된 데이터가 역색인 구조로 있으므로 검색 속도가 매우 빠르다.
  - 그런데, 매 요청마다 새로운 세그먼트를 만들어주면 엄청나게 많은 세그먼트가 생성될 것이고, 이로 인해 다른 요청에 지장이 생길 수 있다. 이를 방지하기 위해 인메모리 버퍼를 생성한다.
  - 인메모리 버퍼에 쌓인 내용을 일정 시간이 지나거나 버퍼가 가득차면 flush를 취하고, flush 작업이 수행되면 시스템 캐시에 세그먼트가 생성된다. 이 시점부터 데이터가 비로소 검색이 가능해진다. 하지만 이 상태는 세그먼트가 시스템 캐시에 저장된 상태이지, 디스크에 저장된 상태는 아니다.

## 장점
- **Scale out** : 샤드를 통해 규모가 수평적으로 늘어날 수 있음
- **고가용성**: Replica를 통해 데이터의 안정성을 보장
- **Schema Free**: Json 문서를 통해 데이터 검색을 수행하므로 스키마 개념이 없음
- **RESTful** : 데이터의 CRUD 작업은 HTTP REST API를 통해서 수행된다.
- **역색인 구조**

## 단점
- **실시간 처리가 불가능하다**
  - Elasticsearch는 인메모리 버퍼를 사용하므로 쓰기와 동시에 읽기 작업을 할 경우, 세그먼트가 생성되기 전까지는 해당 데이터를 검색할 수 없다
- **트랜잭션을 지원하지 않는다.**
  - 분산시스템 구성 특징 때문에 시스템적으로 비용 소모가 큰 트랜잭션 및 롤백을 지원하지 않는다.
- **진정한 의미의 업데이트를 지원하지 않는다.**
  - 세그먼트에서 데이터가 삭제될 경우 Soft-Delete를 한다. (삭제 flag = true)
  - 세그먼트에서 데이터가 수정될 경우 Soft-Delete를 하고, 수정된 데이터를 새로운 세그먼트로 생성한다.
  - 이는 RDBMS의 Index와 유사한 동작이다.

<HR>

출처
- 예전에 작성한 글로 출처 기록이 없습니다..😢 양해부탁드립니다.

<HR>

# 트랜잭션, 잠금, 트랜잭션의 격리 수준
## 개요
- 트랜잭션은 작업의 완전성을 보장해 주는 것이다. 즉 논리적인 작업 셋을 모두 완벽하게 처리하거나, 처리하지 못할 경우에는 원 상태로 복구해서 작업의 일부만 적용되는 현상(Partitial update)이 발생하지 않게 만들어주는 기능이다.
- 잠금(Lock)과 트랜잭션은 서로 비슷한 개념 같지만 
  - 사실 **잠금은 동시성을 제어하기 위한 기능**이고
  - **트랜잭션은 데이터의 정합성을 보장하기 위한 기능**이다.  
- 하나의 회원 정보 레코드를 여러 커넥션에서 동시에 변경하려고 하는데 잠금이 없다면 하나의 데이터를 여러 커넥션에서 동시에 변경할 수 있게 된다. 결과적으로 해당 레코드의 값은 예측할 수 없는 상태가 된다. 
- 잠금은 여러 커넥션에서 동시에 동일한 자원(레코드나 테이블)을 요청할 경우 순서대로 한 시점에는 하나의 커넥션만 변경할 수 있게 해주는 역할을 한다. 
- 격리 수준이라는 것은 하나의 트랜잭션 내에서 또는 여러 트랜잭션 간의 작업 내용을 어떻게 공유하고 차단할 것인지를 결정하는 레벨을 의미한다.

## 1. 트랜잭션
#### 트랜잭션을 사용할 경우 주의할 사항
- **트랜잭션 범위**는 최소화 하라. 
	- 일반적으로 데이터베이스 커넥션은 개수가 제한적이어서 커넥션을 소유하고 있는 시간을 최소화해야 한다.
- 메일 전송이나 FTP 파일 전송 작업 또는 네트워크를 통해 원격 서버와 통신하는 등과 같은 작업은 어떻게 해서든 DBMS의 트랜잭션 내에서 제거하는 것이 좋다.
	- 이런 실수로 인해 DBMS 서버가 높은 부하 상태로 빠지거나 위험한 상태로 빠지는 경우가 빈번히 발생한다. 

## 2. MySQL 엔진의 잠금
- MySQL에서 사용되는 잠금은 크게 **스토리지 엔진 레벨**과 **MySQL 엔진 레벨**로 나눌 수 있다. 
- MySQL 엔진은 MySQL 서버에서 스토리지 엔진을 제외한 나머지 부분으로 이해하면 된다.
- MySQL 엔진 레벨의 잠금은 모두 스토리지 엔진에 영향을 미치지만, 스토리지 엔진 레벨의 잠금은 스토리지 엔진 간 상호 영향을 미치지는 않는다.
- MySQL 엔진에서는 테이블 데이터 동기화를 위한 테이블 락, 테이블의 구조를 잠그는 메타데이터 락, 그리고 사용자의 필요에 맞게 사용할 수 있는 네임드 락(Named Lock)을 제공한다.

## 3. InnoDB 스토리지 엔진 잠금
- InnoDB 스토리지 엔진은 스토리지 엔진 내부에서 **레코드 기반의 잠금 방식**을 탑재하고 있다. 
- InnoDB는 레코드 기반의 잠금 방식 때문에 MyISAM 보다는 훨씬 뛰어난 동시성 처리를 제공할 수 있다.
- 하지만 이원화된 잠금 처리 탓에 InnoDB 스토리지 엔진에서 사용되는 잠금에 대한 정보는 MySQL 명령을 이용해 접근하기가 상당히 까다롭다. 
- 최근 버전에서는 InnoDB의 트랜잭션과 잠금, 그리고 잠금 대기 중인 트랜잭션의 목록을 조회할 수 있는 방법이 도입됐다. MySQL 서버의 `information_schema` 데이터베이스에 존재하는 `INNODB`, `INNODB_LOCKS`, `INNODB_LOCK_WAITS` 라는 테이블을 조인해서 조회하면 
	- 현재 어떤 트랜잭션이 어떤 잠금을 대기하고 있고 
	- 해당 잠금을 어느 트랜잭션이 가지고 있는지 확인할 수 있으며, 
	- 또한 장시간 잠금을 갖고 있는 클라이언트를 찾아서 종료시킬 수도 있다.
- InnoDB 잠금에 대한 모니터링이 더 강화되면서 `Performance Schema` 를 이용해 InnoDB 스토리지 엔진의 내부 잠금(세마포어)에 대한 모니터링 방법도 추가됐다.

## 4. MySQL의 격리 수준
- 트랜잭션의 격리 수준(isolation level)이란 **여러 트랜잭션**이 **동시에 처리될 때** 특정 트랜잭션이 **다른 트랜잭션에서 변경하거나 조회하는 데이터**를 볼 수 있게 허용할지 말지를 결정하는 것이다.
- 격리 수준은 크게 4가지로 나뉜다.
	- READ UNCOMMITTED
	- READ COMMITTED
	- REPEATABLE READ
	- SERIALIZABLE
- `DIRTY READ`라고도 하는 `READ UNCOMMITTED`는 일반적인 데이터베이스에서는 거의 사용하지 않고, 
- `SERIALIZABLE` 또한 동시성이 중요한 데이터베이스에서는 거의 사용되지 않는다.
- 4개의 격리 수준에서 순서대로 뒤로 갈수록 각 트랜잭션 간의 데이터 격리(고립) 정도가 높아지며, 동시 처리 성능도 떨어지는 것이 일반적이라고 볼 수 있다.
- 격리 수준이 높아질수록 MySQL 서버의 처리 성능이 많이 떨어질 것으로 생각하는 사용자가 많은데, 사실 `SERIALIZABLE` 격리 수준이 아니라면 크게 성능의 개선이나 저하는 발생하지 않는다.

데이터베이스의 격리 수준을 이야기하면 항상 언급되는 세 가지 부정합의 문제점이 있다. 이 세가지 부정합의 문제는 격리 수준의 레벨에 따라 발생할 수도 있고 발생하지 않을 수도 있다.

| |DIRTY READ|NON-REPEATABLE READ|PHANTOM READ|
|--|--|--|--|
|READ UNCOMMITTED|발생|발생|발생|
|READ COMMITTED|없음|발생|발생|
|REPEATABLE READ|없음|없음|발생 <BR>(InnoDB는 없음)|
|SERIALIZABLE|없음|발생|없음|

- SQL-92 또는 SQL-99 표준에 따르면 `REPEATABLE READ` 격리 수준에서는 `PHANTOM READ`가 발생할 수 있지만, InnoDB에서는 독특한 특성 때문에 발생하지 않는다. 
- 일반적인 온라인 서비스 용도의 데이터베이스에는 `READ COMMITTED`와  `REPEATABLE READ` 중 하나를 사용한다. 
	- 오라클 같은 DBMS에서는 주로 `READ COMMITTED` 수준을 많이 사용.
	- MySQL에서는 `REPEATABLE READ` 를 주로 사용.

### InnoDB에서는 왜 PHANTOM READ 문제가 발생하지 않는가?
 InnoDB 스토리지 엔진은 **레코드 락과 갭 락을 합친 넥스트 키 락**을 사용한다. 

t 테이블에 c1 = 13 , c = 17 인 두 레코드가 있다고 가정하자. 이때 SELECT c1 FROM t WHERE c1 BETWEEN 10 AND 20 FOR UPDATE 쿼리를 수행하면, 10 <= c1 <= 12, 14 <= c1 <= 16, 18 <= c1 <= 20 인 영역은 전부 갭 락에 의해 락이 걸려서 해당 영역에 레코드를 삽입할 수 없다. 또한 c = 13, c = 17인 영역도 레코드 락에 의해 해당 영역에 레코드를 삽입할 수 없다. 참고로 INSERT 외에 UPDATE, DELETE 쿼리도 마찬가지이다.

이러한 방식으로 InnoDB 스토리지 엔진은 PHANTOM READ 문제를 해결한다.

<HR>

# 트랜잭션의 격리 수준
## 1. READ UNCOMMITTED
- 각 트랜잭션의 변경 내용이 `COMMIT` 이거나 `ROLLBACK` 여부에 상관없이 다른 트랜잭션에서 보인다.

#### 시나리오
1. 사용자 A는 `emp_no`가 50000 이고 `first_name`이 "Lara"인 새로운 사원을 `INSERT`한다.
2. 사용자 B가 변경된 내용을 커밋하기도 전에 사용자 B는 `emp_no=50000`인 사원을 검색하고 있다. 

하지만 사용자 B는 사용자 A가 `INSERT`한 사원의 정보를 커밋되지 않은 상태에서도 조회할 수 있다. 그런데 문제는 사용자 A가 처리 도중 알 수 없는 문제가 발생해 `INSERT`된 내용을 롤백한다고 하더라도 여전히 사용자 B는 "Lara"가 정상적인 사원이라고 생각하고 계속 처리할 것이라는 점이다.

### 더티리드(Dirty read)
- 이처럼 어떤 트랜잭션에서 처리한 작업이 완료되지 않았는데도 다른 트랜잭션에서 볼 수 있는 현상을 더티 리드(Dirty read)라고 하고, 더티 리드가 허용되는 격리 수준이 `READ UNCOMMITTED`다. 
- 더티 리드를 유발하는 `READ UNCOMMITTED`는 RDBMS 표준에서는 트랜잭션의 격리 수준으로 인정하지 않을 정도로 정합성에 문제가 많은 격리 수준이다.
- MySQL을 사용한다면 최소한 `READ COMMITTED` 이상의 격리 수준을 사용할 것을 권장한다.

## 2. READ COMMITTED
- 오라클 DBMS에서 기본으로 사용되는 격리 수준이며, 온라인 서비스에서 가장 많이 선택되는 격리 수준이다.
- 이 레벨에서는 더티 리드 현상은 발생하지 않는다.
- 어떤 트랜잭션에서 데이터를 변경하더라도 `COMMIT`이 완료된 데이터만 다른 트랜잭션에서 조회할 수 있기 때문이다. 

#### 시나리오
  1. 사용자 A는 `emp_no=50000`인 사원의 `first_name`을 "Lara"에서 "Toto"로 변경했는데, 이때 새로운 값인 "Toto"는 `employees` 테이블에 즉시 기록되고 **이전 값**인 "Lara"는 **UNDO 영역**으로 **백업**된다. 
  2. 사용자 A가 커밋을 수행하기 전에 사용자 B가 `emp_no=50000`인 사원을 `SELECT`하면 조회된 결과의 `first_name`칼럼의 값은 "Lara"로 조회된다. 여기서 사용자 B의 `SELECT` 쿼리 결과는 `employees` 테이블이 아니라 UNDO 영역에 백업된 레코드에서 가져온 것이다.

READ COMMITTED 격리 수준에서는 어떤 트랜잭션에서 변경한 내용이 **커밋되기 전까지는** 다른 트랜잭션에서 그러한 변경 내역을 조회할 수 없기 때문이다. 최종적으로 사용자 A가 변경된 내용을 커밋하면 그때부터는 다른 트랜잭션에서도 백업된 UNDO 레코드("Lara")가 아니라 새롭게 변경된 "Toto"라는 값을 참조할 수 있게 된다.

### NON-REPEATABLE READ 
- READ COMMITTED 격리 수준에도 'REPEATABLE READ가 불가능하다' 라는 부정합의 문제가 있다. 

#### 시나리오
1. 사용자 B가 BEGIN 명령으로 트랜잭션을 시작하고 `first_name`이 "Toto"인 사용자를 검색했는데, 일치하는 결과가 없었다.
2. 하지만 사용자 A가 `emp_no=50000`인 사원의 이름을 "Toto"로 변경하고 커밋을 실행한 후, 사용자 B가 똑같은 SELECT 쿼리로 다시 조회하면 이번에는 결과가 1건이 조회된다.

이는 별다른 문제가 없어보이지만, 사실 사용자 B가 하나의 트랜잭션 내에서 똑같은 SELECT 쿼리를 실행했을 때는 항상 같은 결과를 가져와야한다는 **REPEATABLE READ** 정합성에 어긋나는 것이다. 이러한 부정합 현상은 일반적인 웹 프로그램에서는 크게 문제되지 않을 수 있지만 하나의 트랜잭션에서 동일 데이터를 여러 번 읽고 변경하는 작업이 **금전적인 처리**와 연결되면 문제가 될 수도 있다. 예를 들어, 특정 트랜잭션에서 입금과 출금 처리가 계속 진행될 때 다른 트랜잭션에서 오늘 입금된 금액의 총합을 조회한다고 가정해보자. 그런데 **REPEATABLE READ**가 보장되지 않기 때문에 총합을 계산하는 `SELECT` 쿼리는 실행될 때마다 다른 결과를 가져올 것이다. 

> 중요한 것은 사용 중인 트랜잭션의 격리 수준에 의해 실행하는 SQL 문장이 어떤 결과를 가져오게 되는지를 정확히 예측할 수 있어야 한다는 것이다. 그리고 당연히 이를 위해 각 트랜잭션이 어떻게 작동하는지 알아야 한다. 

## 3. REPEATABLE READ
- MySQL의 InnoDB 스토리지 엔진에서 기본으로 사용되는 격리 수준이다. 바이너리 로그를 가진 MySQL 서버에서는 최소 REPEATABLE READ 격리 수준 이상을 사용해야 한다.
- 이 격리 수준에서는 READ COMMITTED 수준에서 발생하는 **NON-REPEATABLE READ** 부정합이 발생하지 않는다.
- InnoDB 스토리지 엔진은 트랜잭션이 `Rollback`될 가능성에 대비해 변경되기 전 레코드를 Undo 공간에 백업해두고 실제 레코드 값을 변경한다. 이러한 변경 방식을 **MVCC(Multi Version Concurrency Control)** 라고 한다. 
- REPEATABLE READ는 MVCC를 위해 Undo 영역에 백업된 이전 데이터를 이용해 동일 트랜잭션 내에서는 동일한 결과를 보여줄 수 있게 보장한다. 
- 사실 READ COMMITTED도 MVCC를 이용해 `COMMIT` 되기 전의 데이터를 보여준다. REPEATABLE READ와 READ COMMITTED의 차이는 **Undo 영역에 백업된 레코드의 여러 버전 가운데 몇 번째 이전 버전**까지 찾아 들어가야 하느냐에 있다. 
- **모든 InnoDB의 트랜잭션은 고유한 트랜잭션 번호(순차적으로 증가하는 값)를 가지며**, Undo 영역에 백업된 모든 레코드는 변경을 발생시킨 트랜잭션의 번호가 포함돼 있다. 그리고 Undo 영역의 백업된 데이터는 InnoDB 스토리지 엔진이 불필요하다고 판단하는 시점에 주기적으로 삭제한다. 
- REPEATABLE READ 격리 수준에서는 MVCC를 보장하기 위해 실행 중인 트랜잭션 가운데 가장 오래된 트랜잭션 번호보다 트랜잭션 번호가 앞선 Undo 영역의 데이터는 삭제할 수 없다.
- 그렇다고 가장 오래된 트랜잭션 번호 이전의 트랜잭션에 의해 변경된 모든 Undo 데이터가 필요한 것은 아니다. 더 정확하게는 특정 트랜잭션 번호의 구간 내에서 백업된 Undo 데이터가 보존돼야 한다.

#### 시나리오
0. 현재 `employees` 테이블에는 두 개의 레코드가 있고, 모두 TRX-ID(트랜잭션 번호)가 6이다. 
1. 사용자 B의 트랜잭션(TRX-ID: 10) `BEGIN` : `emp_no=50000`인 사원의 이름을 `SELECT` 해서 "Lara"라는 결과를 얻었다.
2. 사용자 A의 트랜잭션(TRX-ID: 12) `BEGIN` :  `emp_no=50000`인 사원의 이름을 "Lara"에서 "Toto"로 변경했다. (TRX-ID: 6 → 12) 
   - 이때 변경 전 데이터를 Undo 로그로 복사한다. (TRX-ID: 6)
   - 그리고 `COMMIT`.
  
사용자 B의 트랜잭션(TRX-ID: 10)안에서 실행되는 모든 `SELECT` 쿼리는 TRX-ID가 10보다 작은 트랜잭션 번호에서 변경한 것만 보게 된다. 

Undo 영역은 하나의 레코드에 대해 백업이 하나 이상 얼마든지 존재할 수 있다. 한 사용자가 BEGIN으로 트랜잭션을 시작하고 장시간 트랜잭션을 종료하지 않으면 Undo 영역이 백업된 데이터로 무한정 커질 수도 있다. 이렇게 Undo에서 백업된 레코드가 많아지면 MySQL 서버의 처리 성능이 떨어질 수 있다.

### PHANTOM READ(PHANTOM ROW) 
- 다른 트랜잭션에서 수행한 변경 작업에 의해 레코드가 보였다 안 보였다 하는 현상을 PHANTOM READ(PHANTOM ROW) 라고 한다.
- `SELECT ... FOR UPDATE` 쿼리는 SELECT하는 레코드에 쓰기 잠금을 걸어야 하는데, Undo 레코드에는 잠금을 걸 수 없다. 
- 그래서 `SELECT ... FOR UPDATE` 나 `SELECT ... LOCK IN SHARE MODE`로 조회하는 레코드는 Undo 영역의 변경 전 데이터를 가져오는 것이 아니라 현재 레코드의 값을 가져오게 되는 것이다.

## 4. SERIALIZABLE
- 가장 단순한 격리 수준이면서 동시에 가장 엄격한 격리 수준이다.
- 그만큼 동시 처리 성능도 다른 트랜잭션 격리 수준보다 떨어진다.
- InnoDB에서 기본적으로 순수한 `SELECT` 작업은 아무런 레코드 잠금도 설정하지 않고 실행된다
	- 순수한 SELECT: `INSERT ... SELECT ...` 또는 `CREATE TABLE ... AS SELECT ...` 가 아닌 SELECT
- InnoDB 매뉴얼에서 자주 나타나는 "**Non-locking consistenct read(잠금이 필요 없는 일관된 읽기)**"라는 말이 이를 의미하는 것이다.
- 하지만 트랜잭션의 격리 수준이 SERIALIZABLE로 설정되면 읽기 작업도 공유 잠금(읽기 잠금)을 획득해야 하며, 동시에 다른 트랜잭션은 그러한 레코드를 변경하지 못하게 된다.
- 즉, 한 트랜잭션에서 읽고 쓰는 레코드를 다른 트랜잭션에서는 절대 접근할 수 없는 것이다.
- SERIALIZABLE 격리 수준에서는 일반적인 DBMS에서 일어나는 PHANTOM READ 문제가 발생하지 않는다.
- 하지만 InnoDB 스토리지 엔진에서는 갭 락과 넥스트 키 덕분에 REPEATABLE READ 격리 수준에서도 이미 PHANTOM READ가 발생하지 않기 때문에 굳이 SERIALIZABLE을 사용할 필요성은 없어 보인다. 
	> 엄밀하게는 `SELECT ... FOR UPDATE` 나 `SELECT ... FOR SHARE` 쿼리의 경우 REPEATABLE READ 격리 수준에서 PHANTOM READ 현상이 발생할 수 있다. 하지만 레코드 변경 이력(Undo 레코드)에 잠금을 걸 수는 없기 때문에, 이러한 잠금을 동반한 SELECT 쿼리는 예외적인 상황으로 볼 수 있다. 

### `SELECT ... FOR UPDATE`
- 동시성 제어를 위해 특정 데이터(ROW)에 대해 배타적 락을 건다.


### `SELECT ... FOR SHARE` 


<HR>

출처
- Real MySQL 8.0 책

<HR>


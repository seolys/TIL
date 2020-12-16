# 수십억건에서 QUERYDSL 사용하기

- https://www.youtube.com/watch?v=zMAX7g6rO_Y&feature=emb_title

## 적재된 데이터가 1000만건에서 10억건까지 되는 과정에서 얻은 Querydsl-JPA 개선 TIP

### 1. 워밍업

- extends / implements 사용하지 않기

  - 꼭 무언가를 상속/구현 받지 않더라도, 꼭 특정 Entity를 지정하지 않더라도 Querydsl을 사용할 수 있는 방법.
  - JPAQueryFactory만 있으면 Querydsl을 사용할 수 있다.

    - extends/implements 다 제거해도 된다고 한다.

    ```java
    @RequiredArgsConstructor
    @Repository
    public class AcademyQueryRepository {
        private final JPAQueryFactory queryFactory;

        public List<Acadymy> findByName(String name) {
            return queryFactory
                .selectFrom(acadymy)
                .where(academy.name.eq(name))
                .fetch();
        }
    }
    ```

- 동적쿼리
  - BooleanBuilder은 사용하기 쉽지만, 어떤 쿼리인지 예상하기 어렵다.
    - 조건이 많아질수록 점점 더..
  - BooleanExpression을 사용하자. null 반환 시 자동으로 조건절에서 제거된다.
    - 모든 조건이 null이 발생하는 경우는 장애가 발생하니 주의!
    ```java
    public List<Acadymy> findByName(String name) {
        return queryFactory
            .selectFrom(acadymy)
            .where(
                nameEq(name)
                // 추가 조건 생략. 모든조건이 null이 되지않아야 한다.
            )
            .fetch();
    }
    private BooleanExpression nameEq(String name) {
        if (StringUtils.isEmpty(name)) {
            return null;
        }
        return acadymy.name.eq(name);
    }
    ```

### 2. 성능개선 - Select

- exists 금지
  - 2500만건 기준으로 exists는 약 2.57초, count(1) > 0은 약 5.2초가 소요된다.
    - exists는 첫번째 값이 발견되는 순간 쿼리가 종료  
      count는 끝까지 모든 조건을 체크하기때문에 쿼리가 느림.
    - 스캔 대상조건이 앞에 있을 수록 더 심한 성능 차이가 발생한다.
  - JPQL은 from절이 없으면 쿼리를 내보내지 못함. 때문에 아래와 같은 쿼리는 불가.
    ```sql
    select exists(
        select 1
        from ad_item_sum
        where created_date > '2020-01-01'
    )
    ```
  - 아쉽게도 Querydsl의 exists는 실제로 count() > 0 으로 실행된다고함.
    ```java
    // QuerydslJpaPredicateExecutor.class
    @Override
    public boolean exists(Predicate predicate) {
        return createQuery(predicate).fetchCount() > 0;
    }
    ```
  - exists가 빠른 이유는 조건에 해당하는 row 1개만 찾으면 바로 쿼리를 종료하기 때문이다.  
     이를 직접 구현하자
    - limit 1로 조회 제한.
    ```java
    @Transactional(readOnly = true)
    public Boolean exist(Long bookId) {
        Integer fetchOne = queryFactory
            .selectOne()
            .from(book)
            .where(book.id.eq(bookId))
            .fetchFirst(); // fetchFirst == limit(1).fetchOne()과 동일함.
        return fetchOne != null; // 조회결과가 없으면 null을 리턴하기때문에, null여부를 체크해야 한다.
    }
    ```
    -
  - 정리
    - exists와 limit 1은 성능이 비슷하니, limit 1을 활용하자.
      ```sql
      select exists(
          select 1
          from ad_item_sum
          where created_date > '2020-01-01'
      )
      -- execution: 2s 56ms.
      ```
      ```sql
      select 1
      from ad_item_sum
      where created_date > '2020-01-01'
      limit 1
      -- execution: 2s 5ms.
      ```
- Cross Join 회피

  - 묵시적 Join으로 Cross Join 발생(일부의 DB는 이에 대해 어느정도 최적화가 지원됨.)
    ```java
    public List<Customer> crossJoin() {
        return queryFactory
            .selectFrom(customer)
            .where(customer.customerNo.gt(customer.shop.shopNo))
            .fetch();
    }
    ```
  - Hibernate 이슈라서, Spring Data JPA도 동일함.(JPQL)
  - 명시적 조인을 사용하자.
    ```java
    public List<Customer> crossJoin() {
        return queryFactory
            .selectFrom(customer)
            .innerJoin(customer.shop, shop)
            .where(customer.customerNo.gt(customer.shop.shopNo))
            .fetch();
    }
    ```

- Entity 보다는 Dto를 우선.
  - Entity 조회시 단점
    - Hibernate 캐시
    - 불필요한 컬럼 조회
    - OneToOne N+1 쿼리
    - 단순 조회 기능에서는 성능 이슈 요소가 많음.
  - 경우에 따른 처리
    - Entity 조회
      - 실시간으로 Entity 변경이 필요한 경우.
    - Dto 조회
      - 고강도 성능 개선 or 대량의 데이터 조회가 필요한 경우.
  - 조회컬럼 최소화
    ```java
    public List<BookPageDto> getBooks(int bookNo, int pageNo) {
        return queryFactory
            .select(
                Projections.feilds(BookPageDto.class
                    book.name,
                    // book.bookNo, // 파라메터에 있는 이미 알고있는 값.
                    Expressions.asNumber(bookNo).as("bookNo"), // as 표현식으로 대체한다.
                    book.id
                )
            )
            .from(book)
            .where(book.bookNo.eq(bookNo))
            .fetch();
    }
    Hibernate:
        select
            name,
            -- as컬럼은 select에서 제외된다.
            id
        from
            book
        where
            book_no = ?
    ```
- Select 컬럼에 Entity 자제

  - Querydsl로 조회된 결과를 신규 Entity로 생성.
    ```java
    queryFactory
        .select(
            Projections.feilds(AbBond.class,
                adItem.customer // 연관관계 Entity
            )
        )
        .from(...)
    ```
  - Customer의 모든 컬럼이 조회됨.(사용하지도 않는 컬럼들이 포함됨.)
  - Customer와 @OneToOne 관계인 Shop이 매 건마다 조회된다.
    - oneToOne은 Lazy Loading이 안된다. 무조건 N+1 발생.
    - Shop에도 @OneToOne이 있다면 쿼리가 그만큼 발생하게 됨.
  - Entity간 연관관계를 맺으려면 반대 Entity가 있어야 하는가?

    - 없어도됨. 반대면 ID만 있으면 됨.

      ```java
      queryFactory
          .select(
              Projections.field(AdBondDto.class
                  adItem.amout,
                  // ...생략...
                  adItem.customer.id.as("customerId") // customerId만 조회
              )
          ).from(adItem)
          // ...생략

      public AdBond toEntity() {
          return AdBond.builder()
                      .amount(amout)
                      // ...생략...
                      .customer(new Customer(customerId)) // AdBond가 저장되는 시점에는 Customer의 ID외에는 필요없다.
      }               .build();
      ```

    - disinct 문제
      - Select에 선언된 Entity의 컬럼 전체가 distinct 대상이 된다.

- Group By 최적화

  - MySQL에서 Group By를 실행하면 Filesort 비용이 필수로 발생함.(Index가 아닌 경우.)

    - order by null을 사용하면 Filesort가 제거된다.
    - 하지만 Querydsl에서는 order by null 문법을 지원하지 않는다.
    - 해결방법

      - 정렬이 없는경우, OrderByNull 조건 클래스를 생성하여 적용한다.

        ```java
        public class OrderByNull extends OrderSepecifier {
            public static final OrderByNull DEFAULT = new OrderByNull();

            private OrderByNull() {
                super(Order.ASC, NullExpression.DEFAULT, Default);
            }
        }

        queryFactory
            .selectFrom(...)
            .groupBy(txAddition.type, txAddition.code)
            .orderBy(OrderByNull.DEFAULT)
            .fetch();
        ```

      - 정렬이 필요하더라도, 조회 결과가 100건 이하라면 애플리케이션에서 정렬한다.
        ```java
        result.sort(comparingLong(PointCalculateAmount::getPointAmount));
        ```
        - 단, 페이징일 경우, order by null을 사용하지 못한다.

- 커버링 인덱스

  - 쿼리를 충족시키는데 필요한 모든 컬럼을 갖고 있는 인덱스
    - select / where / order by/ group by 등에서 사용되는 모든 컬럼이 인덱스에 포함된 상태
    - NoOffset 방식과 더불어 페이징 조회 성능을 향상시키는 가장 보편적인 방법
      ```sql
      select *
        from academy a
        join (select id
                from acadymy
               order by id
               limit 10000, 10) as temp
          on temp.id = a.id;
      ```
  - 하지만, JPQL은 from절의 서브쿼리를 지원하지 않는다.
  - 쿼리를 나눠서 진행.

    - Cluster Key(PK)를 커버링 인덱스로 빠르게 조회하고, 조회된 Key로 SELECT 컬럼들을 후속조회한다.

    ```java
    // 커버링 인덱스로 빠르게 키값만 조회.
    List<Long> ids = queryFactory
        .select(book.id)
        .from(book)
        // ...생략...
        .fetch()
    if(ids.isEmpty()){
        return emptyList();
    }

    // 후속 조회
    return queryFactory
        .select(
            Projections.fields(BookPaginationDto.class
                book.id.as("bookId"),
                book.name,
                book.bookNo,
                //...생략...
            )
        )
        .from(book)
        .where(book.id.in(ids)) // 후속 조회 조건.
        //...생략...
        .fetch();

    ```

### 3. 성능개선 - Update/Insert

- 일괄 Update 최적화

  - 객체지향을 핑계로 성능을 버리진 않는지, 무분별은 DirtyChecking을 꼭 확인해봐야 한다.
    - Querydsl update문 작성을 고려하라
      - 한방의 SQL로 update.
    - 1만건 단일 컬럼 기준으로 2000배 차이.
      - DirtyChecking: 9m 19s
      - UpdateQuery: 272ms
  - 일괄 Update의 단점
    - 하이버네이트 캐시가 갱신이 안됨.
      - 업데이트 대상들에 Cache Eviction이 필요하다.
      - Entity.flush(); Entity.clear();
  - 정리
    - DirtyChecking
      - 실시간 비즈니스 처리, 실시간 단건 처리시 사용할것.
    - 일괄 update
      - 대량의 데이터를 일괄로 Update 처리 시 사용할것.
  - 진짜 Entity가 필요한 게 아니라면, Querydsl과 Dto를 통해 딱 필요한 항목들만 조회하고, 업데이트 하자.

- Bulk Insert

  - JPA로 BultInsert는 자제한다.
  - rewriteBatchedStatements으로 Insert 합치기 옵션을 넣어도, JPA는 auto_increment일때 Insert 합치기가 적용되지 않는다.
  - JdbcTemplate로 Bulk Insert는 처리되나, 컴파일체크, 코드-테이블간의 불일치 체크등 Type-Safe개발이 어렵다.
  - 배민에서 적용하고있는 방식.

    - TypeSafe한 방식으로 Bulk Insert를 처리할 순 없을까?

      - Querydsl != Querydsl-JPA
        - Querydsl
          - Querydsl-JPA -> JPQL
          - Querydsl-SQL -> Native SQL
          - Querydsl-MongoDB -> Mongo Query
          - QueryDsl-ElasticSearch -> ES Query
      - QClass 기반으로 Native SQL을 사용할 수 있는 Querydsl-SQL

        - 여태 왜 안썼을까?
          - 테이블을 Scan해서 QClass를 만드는 방식임.
            - 로컬 PC에 DB를 설치하고 실행한 뒤,  
              Gradle/Maven에 로컬 DB정보를 등록해서 flyway로 테이블을 생성하고,  
              Querydsl-SQL 플러그인으로 테이블 Scan하면 QClass가 생성된다.
            - 너무 번거로움. 신규컬럼이 있을경우,,,,,,,
        - JPA 어노테이션으로 Querydsl-SQL QClass를 생성할순 없을까?

          - EntityQL이라는 프로젝트가 존재함.

            - JPA Entity를 기반으로 Querydsl-SQL QClass를 생성해준다.
            - EntityQL로 만들어진 Querydsl-SQL의 QClass를 이용하면 BulkInsert가 가능하다고 함.

              - 단일 Entity
                ```java
                SQLInsertClause insert = sqlQueryFactory.insert(qAcademy);
                for (int j = 1; j <= 1000; j++) {
                    insert.populate(new Academy("address", "name"), EntityMapper.DEFAULT).addBatch();
                }
                insert.execute();
                ```
              - OneToMany

                ```java
                SQLInsertClause insert = sqlQueryFactory.insert(qStudent);
                for (int j = 1; j <= 1000; j++) {
                    Academy academy = academyRepository.save(new Academy("address", "name"));

                    insert.populate(new Student("student", 1, academy), EntityMapper.DEFAULT).addBatch();

                    insert.populate(new Student("student", 2, academy), EntityMapper.DEFAULT).addBatch();
                }
                insert.execute();
                ```

            - EntityQL의 단점
              - Gradle 5이상 필요.
              - 어노테이션에 (name="명칭") 필수
                - @Column의 name 값으로 QClass필드가 선언된다.
                - @Table의 name값으로 테이블을 찾을 수 있다.
              - primitive type 사용 X
                - int, double, boolean 등을 사용할 수 없다.
                - 모든 컬럼은 Wrapper Class로 선언해야 한다.
              - 복잡한 설정
                - querydsl-sql이 개선되지 못해 불편한 설정이 많다고함.
              - @Embedded 미지원
                - @Embedded 어노테이션을 통한 컬럼인식을 못한다.
                - @JoinColumn은 지원한다.
              - 어플리케이션에서는 camelCase, DB는 unserScore인 경우 Mapper클래스가 별도로 필요함.

### 4. 마무리

- 상황에 따라 ORM / 전통적 Query 방식을 골라 사용할 것.
- JPA / Querydsl로 발생하는 쿼리 한번 더 확인하기.

- 책추천
  - JPA
  - Real MySQL
    - http://www.yes24.com/Product/Goods/6960931
  - DDD, 애플리케이션 성능 최적화도 중요하지만 DB를 등한시 하지말자

# Spring Data JPA: 신규 Entity 판단 전략 분석

JPA를 사용할 때, 단순히 `@Id`와 `@GeneratedValue`를 함께 `save()` 메서드에 사용하곤 합니다. 하지만 `save()` 메서드가 항상 우리가 기대하는 대로 동작하는 것은 아닙니다. 문득 "이 방식이 최선일까?"라는 의문이 들어, 이번 기회에 Spring Data JPA가 새로운 엔티티(Entity)를 판단하는 내부 동작 원리를 깊이 있게 정리해 보기로 했습니다.

Spring Data JPA의 `save()` 메서드는 내부적으로 **새로운 엔티티인지, 아니면 이미 존재하는 엔티티인지**를 판단하여 `EntityManager`의 `persist()`(신규 저장) 또는 `merge()`(기존 엔티티 병합)를 호출합니다. 이 판단 로직을 정확히 이해하는 것은 성능 최적화와 예상치 못한 동작을 방지하는 데 매우 중요합니다.

---

## `save()` 메서드의 핵심 동작 원리

`SimpleJpaRepository`에 구현된 `save()` 메서드의 코드를 살펴보면 로직을 명확히 알 수 있습니다.

```java
@Repository
@Transactional(readOnly = true)
public class SimpleJpaRepository<T, ID> implements JpaRepositoryImplementation<T, ID> {

    // ...

    @Override
    @Transactional
    public <S extends T> S save(S entity) {

        Assert.notNull(entity, "Entity must not be null");

        if (entityInformation.isNew(entity)) {
            entityManager.persist(entity); // ◀ 새로운 엔티티일 경우
            return entity;
        } else {
            return entityManager.merge(entity); // ◀ 이미 존재하는 엔티티일 경우
        }
    }
}
```

### 코드 핵심 분석

-   `@Transactional`: `save()` 메서드는 하나의 트랜잭션 내에서 동작합니다.
-   `entityInformation.isNew(entity)`: **새로운 엔티티인지를 판단하는 핵심 로직**입니다.
    -   기본적으로 엔티티의 `@Id` 프로퍼티가 `null`인지 확인합니다.
    -   만약 엔티티가 `Persistable` 인터페이스를 구현했다면, 해당 엔티티의 `isNew()` 메서드를 직접 호출하여 판단합니다.
-   `entityManager.persist(entity)`: `isNew()`가 `true`를 반환하면, JPA의 `persist()`를 호출하여 엔티티를 영속성 컨텍스트에 새로 저장합니다. (SQL `INSERT` 준비)
-   `entityManager.merge(entity)`: `isNew()`가 `false`를 반환하면, `merge()`를 호출하여 엔티티를 영속성 컨텍스트에 병합(수정)합니다. (SQL `UPDATE` 또는 `INSERT` 준비)

### 기본 전략: 식별자(@Id) 필드 확인

`isNew()`의 기본 판단 기준은 다음과 같습니다.

1.  **식별자 값이 `null`인 경우 → 새로운 엔티티(New)**
    -   `save()` 메서드는 `EntityManager.persist()`를 호출합니다.
    -   이는 가장 효율적인 저장 방식으로, 영속성 컨텍스트에 새 엔티티를 등록하고 트랜잭션 커밋 시 `INSERT` SQL을 실행합니다.

2.  **식별자 값이 `null`이 아닌 경우 → 이미 존재하는 엔티티(Not New)**
    -   `save()` 메서드는 `EntityManager.merge()`를 호출합니다.
    -   `merge()`는 먼저 해당 식별자로 DB에 엔티티가 있는지 확인하기 위해 `SELECT` SQL을 실행합니다.
        -   **DB에 엔티티가 있으면**: 조회한 엔티티에 현재 객체의 값을 덮어씌운 후, 트랜잭션 커밋 시 `UPDATE` SQL을 실행합니다.
        -   **DB에 엔티티가 없으면**: `INSERT` SQL을 실행합니다.

---

## 두 가지 궁금증

이 기본 전략을 바탕으로 두 가지 의문점이 생겼습니다.

> 1.  **`@Id` 값이 있지만, 실제 DB에는 데이터가 없는 상태**에서 `save()`를 호출하면 어떻게 동작할까?
> 2.  `@Id` 필드의 타입이 **Wrapper 타입(예: `Long`)인지, Primitive 타입(예: `long`)인지**에 따라 동작이 달라질까?

하나씩 확인해 보겠습니다.

### 1. `@Id` 값은 있지만 DB에 데이터가 없는 경우

결론부터 말하면, `merge()`가 호출되어 **불필요한 `SELECT` 쿼리가 발생**합니다.

#### 동작 순서
1.  **ID 값 존재 확인**: `save()` 메서드는 엔티티의 ID가 `null`이 아님을 확인하고, 이 엔티티를 '새로운 엔티티'가 아니라고 판단합니다.
2.  **`merge()` 호출**: 새로운 엔티티가 아니므로 `EntityManager.merge()`를 호출합니다.
3.  **`SELECT` 쿼리 실행**: `merge()`는 영속성 컨텍스트에 병합하기 전, DB에 데이터가 실제로 존재하는지 확인하기 위해 ID를 조건으로 `SELECT` 쿼리를 먼저 실행합니다.
    ```sql
    SELECT * FROM a_table WHERE id = ?;
    ```
4.  **`INSERT` 쿼리 실행**: `SELECT` 결과, DB에 데이터가 없음을 확인합니다. `merge()`는 업데이트할 대상이 없으므로, 이 엔티티를 새로운 데이터로 취급하여 `INSERT` 쿼리를 실행합니다.

#### 문제점: 비효율 발생
`save()` 호출 → `merge()` 호출 → **불필요한 `SELECT` 실행** → `INSERT` 실행

결과적으로 데이터는 정상 저장되지만, 새로운 데이터를 저장함에도 불구하고 불필요한 `SELECT` 쿼리가 한 번 더 실행되어 성능상 손해를 보게 됩니다.

### 2. `@Id` 타입: Wrapper vs. Primitive

`@Id` 필드가 원시 타입(primitive)이냐, 래퍼 타입(wrapper)이냐에 따라 `isNew()`의 동작이 달라지며, 심각한 문제를 일으킬 수 있습니다.

> **결론: JPA 엔티티의 식별자 필드는 반드시 래퍼(Wrapper) 타입을 사용해야 합니다.**

#### Wrapper 타입 (권장)
-   **특징**: 객체이므로 `null` 값을 가질 수 있습니다.
-   **동작**: 새로운 엔티티 객체를 생성하면 ID 필드는 기본적으로 `null` 상태입니다. Spring Data JPA의 `isNew()`는 ID가 `null`이므로 새로운 엔티티로 **정확하게 판단**합니다.
-   **결과**: `EntityManager.persist()`가 호출되어 불필요한 `SELECT` 없이 바로 `INSERT`가 실행됩니다.

```java
@Entity
public class Board {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id; // Wrapper 타입 (O)
    // ...
}

// 새로운 객체 생성
Board board = new Board(...); // 이때 board.id는 null 입니다.
boardRepository.save(board); // id가 null이므로 isNew()는 true -> persist() 호출
```

#### Primitive 타입 (사용 금지)
-   **특징**: `null`을 가질 수 없으며, 값이 할당되지 않으면 **기본값(default value)**을 가집니다. (예: `long`, `int`의 경우 `0`)
-   **동작**: 새로운 엔티티 객체를 생성하면 ID 필드 값은 `null`이 아니라 `0`이 됩니다. `isNew()`는 ID가 `0`이므로 `null`이 아니라고 판단하여, 이 엔티티를 새로운 엔티티가 아니라고 **잘못 판단**합니다.
-   **결과**: `EntityManager.merge()`가 호출됩니다. `merge()`는 DB에서 ID가 `0`인 데이터를 찾기 위해 불필요한 `SELECT` 쿼리를 실행하고, 없으면 그제야 `INSERT`를 실행합니다.

```java
@Entity
public class Board {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private long id; // primitive 타입 (X)
    // ...
}

// 새로운 객체 생성
Board board = new Board(...); // 이때 board.id는 0 입니다.
boardRepository.save(board); // id가 0이므로 isNew()는 false -> merge() 호출 -> 불필요한 SELECT 발생!
```

| 구분 | Wrapper 타입 (`Long`) | Primitive 타입 (`long`) |
| :--- | :--- | :--- |
| **기본값** | `null` | `0` |
| **`isNew()` 기본 동작** | `id == null` 체크 → **정상 동작 (true)** | `id == 0`은 `null`이 아님 → **오동작 (false)** |
| **`save()` 시 호출 메서드** | `persist()` | `merge()` |

---

## 해결 방안: `Persistable` 인터페이스 활용

앞서 제기된 1번 문제(불필요한 `SELECT`)를 해결하고 코드의 의도를 명확하게 만들기 위해 `Persistable` 인터페이스를 사용할 수 있습니다.

### `Persistable`을 사용해야 하는 이유

가장 큰 이유는 **불필요한 `SELECT` 쿼리를 막아 성능을 향상**시키기 위함입니다.

`@Id` 필드에 값을 직접 할당하는 전략(예: UUID, 비즈니스 키)을 사용할 때 이 문제는 두드러집니다. 객체 생성 시점에 이미 ID 값이 존재하므로, `save()` 메서드는 ID가 `null`이 아니라는 이유만으로 `merge()`를 호출하여 비효율적인 `SELECT`를 실행하게 됩니다.

`Persistable` 인터페이스의 `isNew()` 메서드를 직접 구현하면, ID 값의 존재 여부가 아닌 **개발자가 정의한 로직으로 새로운 엔티티인지를 판단**하게 할 수 있습니다. 이를 통해 `save()`가 불필요한 `SELECT` 없이 바로 `persist()`를 호출하도록 제어할 수 있습니다.

### 구현 예제

JPA Auditing의 `@CreatedDate` 필드를 활용하여 "생성일자가 없으면 새로운 엔티티"라고 판단하는 로직을 구현해 보겠습니다.

#### 1. JPA Auditing 활성화
`@CreatedDate`를 사용하기 위해 프로젝트에 JPA Auditing 기능을 활성화합니다.

```java
@Configuration
@EnableJpaAuditing
public class JpaConfig {
}
```

#### 2. `Persistable`을 구현한 엔티티

```java
@Getter
@NoArgsConstructor(access = AccessLevel.PROTECTED)
@Entity
@Table(name = "board")
@EntityListeners(AuditingEntityListener.class)
public class Board implements Persistable<String> {

    @Id
    private String borderId; // 직접 할당하는 ID

    @Column(name = "subject")
    private String subject;

    @Column(name = "contents")
    private String contents;

    @CreatedDate
    @Column(updatable = false) // 생성일자는 수정되지 않도록 설정
    private LocalDateTime createdDate;

    @Builder
    public Board(String borderId, String subject, String contents) {
        this.borderId = borderId;
        this.subject = subject;
        this.contents = contents;
    }

    @Override
    public String getId() {
        return borderId;
    }

    @Override
    public boolean isNew() {
        // createdDate가 null이면 새로운 엔티티로 판단
        return createdDate == null;
    }
}
```

#### 3. 서비스 코드

```java
@Service
@RequiredArgsConstructor
@Transactional(readOnly = true)
public class BoardService {

    private final BoardRepository boardRepository;

    @Transactional
    public void saveBoard(BoardDto dto) {
        Board board = Board.builder()
                .borderId(dto.getBorderId())
                .subject(dto.getSubject())
                .contents(dto.getContents())
                .build();

        boardRepository.save(board);
    }
}
```

#### 4. 실행 결과
`save()`를 호출하면, `isNew()`는 `createdDate`가 `null`이므로 `true`를 반환합니다. 따라서 `persist()`가 호출되어 `SELECT` 쿼리 없이 바로 `INSERT`가 실행됩니다.

```sql
/* insert for com.test.spring.domain.Board */
insert
into
    board
    (created_date, contents, subject, border_id)
values
    (?, ?, ?, ?)
```

---

제가 정리한 내용에 잘못된 점이나 다른 의견이 있다면 댓글로 공유해 주시면 확인 후 수정하겠습니다. <br>
감사합니다. 🙇

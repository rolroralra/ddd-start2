# Aggregate 와 Transaction
- 운영자와 고객이 동시에 주문 Aggregate를 수정한다고 생각해보자.
  - 운영자는 현재 배송지를 조회하고, 주문 Aggregate를 배송상태로 변경
  - 고객은 배송지를 변경
  
![동시성 문제](https://3553248446-files.gitbook.io/~/files/v0/b/gitbook-legacy-files/o/assets%2F-M5HOStxvx-Jr0fqZhyW%2F-MCvkgI26jt9I2m6y0y3%2F-MCvlY-3sW0JbdZcdfEu%2F8.1.png?alt=media&token=5d50c822-880c-40e2-8b2e-a9c3c05fea3e)

## 동시성 문제
- 운영자가 배송지 정보를 조회하고 상태를 변경하는 동안
  - 고객이 Aggregate를 수정하지 못하게 Blocking 해야한다.
- 운영자가 배송지 정보를 조회한 이후
  - 고객이 주문 Aggregate를 수정하면,
    - 운영자가 주문 Aggregate를 다시 조회한 뒤, 수정하도록 해야한다.

# Pessimistic Lock (선점 잠금)
- Pessimistic Lock은 먼저 Aggregate를 조회한 트랜잭션이 다른 트랜잭션에 의해 수정되는 것을 막는다.
  - 조회한 Aggregate에 대한 Lock을 걸고, 트랜잭션이 종료될 때까지 유지한다.
- DBMS에서 제공하는 Lock 기능을 사용한다.
  - `select for update`를 사용한다.
  - 특정 레코드에 한 Connection만 접근할 수 있다.

![PessimisticLock](https://nesoy.github.io/assets/posts/20180628/1.png)

## 코드 예시1
```java
Order order = entityManager.find(Order.class, orderNo, LockModeType.PESSIMISTIC_WRITE);
```

## 코드 예시2
```java
public interface MemberRepository extends JpaRepository<Member, MemberId> {
    @Lock(LockModeType.PESSIMISTIC_WRITE)
    @Query("select m from Member m where m.id = :id")
    Optional<Member> findByIdWithPessimisticLock(MemberId id);
}
```

## Dead Lock (교착상태)
- 두 개 이상의 트랜잭션이 서로 상대방이 가지고 있는 Lock을 기다리는 상태
  - 트랜잭션 A가 Aggregate A에 대한 Lock을 획득하고, Aggregate B에 대한 Lock을 획득하기 위해 대기
  - 트랜잭션 B가 Aggregate B에 대한 Lock을 획득하고, Aggregate A에 대한 Lock을 획득하기 위해 대기

## Dead Lock 해결 방법
- `javax.persistence.lock.timeout` QueryHints 설정을 해준다.

### 코드 예시1
```java
Map<String, Object> queryHints = new HashMap<>();
queryHints.put("javax.persistence.lock.timeout", 3000);
Order order = entityManager.find(Order.class, orderNo, LockModeType.PESSIMISTIC_WRITE, queryHInts);
```

### 코드 예시2
```java
public interface MemberRepository extends JpaRepository<Member, MemberId> {
    @Lock(LockModeType.PESSIMISTIC_WRITE)
    @QueryHints(value = @QueryHint(name = "javax.persistence.lock.timeout", value = "3000"))
    @Query("select m from Member m where m.id = :id")
    Optional<Member> findByIdWithPessimisticLock(MemberId id);
}
```

# Optimisitic Lock (비선점 잠금)
- Optimistic Lock은 트랜잭션이 Aggregate를 수정할 때, 다른 트랜잭션에 의해 Aggregate가 수정되지 않았는지 확인한다.
  - Aggregate의 버전을 체크해서, 버전이 다르면 예외를 발생시킨다.
```sql
UPDATE  ORDERS
SET     STATUS = 'SHIPPED',
VERSION = VERSION + 1
WHERE   ORDER_NO = '2021010100001'
AND VERSION = ${CURRENT_VERSION}
```

![OptimisticLock](https://3553248446-files.gitbook.io/~/files/v0/b/gitbook-legacy-files/o/assets%2F-M5HOStxvx-Jr0fqZhyW%2F-MCvkgI26jt9I2m6y0y3%2F-MCvlfob687ys1zV8nxr%2F8.5.png?alt=media&token=a9e2eca1-0ff0-4668-8554-c410b0adc13e)

## 코드 예시1
```java
Order order = entityManager.find(Order.class, orderNo, LockModeType.OPTIMISTIC);
```

## 코드 예시2
- JPA는 `@Version` Annotation을 활용하여 Optimistic Lock 기능을 제공한다.
    - 매핑되는 테이블에 `VERSION` 컬럼을 추가한다.

```java
@Entity
@Table(name = "ORDERS")
public class Order {
    @EmbeddedId
    private OrderId id;
    
    @Version
    private Long version;
}

public interface OrderRepository extends JpaRepository<Order, OrderId> {
    @Lock(LockModeType.OPTIMISTIC)
    @Query("select o from Order o where o.id = :id")
    Optional<Stock> findByIdWithOptimisticLock(@Param("id") OrderId id);
}
```

## Optimistic Lock Force Increment
- Aggregate Root와 연관된 Aggregate가 수정되었을 경우
  - Root Entity는 변경이 없지만
  - 도메인 관점에서는 변경이 발생했다고 판단할 수 있다.
  - 이런 경우, Root Entity의 버전을 강제로 증가시킨다.
- LockModeType.OPTIMISTIC_FORCE_INCREMENT
  - 조회한 Aggregate의 버전을 강제로 증가시킨다. 

# 오프라인 잠금
- DB를 통한 Lock 인터페이스
  - Custom Table을 통한 구현
  - MySQL에서 제공하는 Lock 함수를 사용한 구현
- Redis를 활용한 Lock 인터페이스
  - Lettuce를 활용한 구현
  - Redisson을 활용한 구현

```java
public interface LockManager {
    LockId tryLock(String type, String id) throws LockException;
    
    void checkLock(LockId lockId) throws LockException;
    
    void releaseLock(LockId lockId) throws LockException;
    
    void extendLockExpiration(Lockid lockId, long expirationTime) throws LockException;
}
```

## 코드 예시 (DB)
```java
public interface NamedLockRepository extends JpaRepository<Stock, Long> {

    @Query(value = "SELECT GET_LOCK(:key, 3000)", nativeQuery = true)
    void lock(@Param("key") String key);

    @Query(value = "SELECT RELEASE_LOCK(:key)", nativeQuery = true)
    void unlock(@Param("key") String key);
}
```

## 코드 예시 (Redis)
### Lettuce
```java
@Component
public class RedisLockRepository {
    private final RedisTemplate<String, String> redisTemplate;

    public RedisLockRepository(RedisTemplate<String, String> redisTemplate) {
        this.redisTemplate = redisTemplate;
    }

    /**
     * Lock product by ID with Redis Lock (Lettuce)
     * @param id Product ID
     * @return true if lock is acquired, false otherwise
     */
    public Boolean lock(Long id) {
        return redisTemplate
            .opsForValue()
            .setIfAbsent(id.toString(), "lock", Duration.ofSeconds(3));
    }

    /**
     * Unlock product by ID with Redis Lock (Lettuce)
     * @param id Product ID
     * @return true if lock is released, false otherwise
     */
    public Boolean unlock(Long id) {
        return redisTemplate.delete(id.toString());
    }
}
```

```java
@Component
public class StockLettuceLockFacade implements StockCommand {
    private final RedisLockRepository redisLockRepository;

    private final StockService stockService;

    public StockLettuceLockFacade(
        RedisLockRepository redisLockRepository,
        StockService stockService) {
        this.redisLockRepository = redisLockRepository;
        this.stockService = stockService;
    }

    /**
     * Decrease stock quantity with Redis Lock (Lettuce)
     * @param id Product ID
     * @param quantity Quantity to decrease
     */
    @Override
    public void decreaseStockQuantity(Long id, Long quantity) {
        try {
            while (Boolean.FALSE.equals(redisLockRepository.lock(id))) {
                sleep(100);  // Spin Lock
            }

            stockService.decreaseStockQuantity(id, quantity);
        } finally {
            redisLockRepository.unlock(id);
        }
    }

    private void sleep(long millis) {
        try {
            Thread.sleep(millis);
        } catch (InterruptedException e) {
            Thread.currentThread().interrupt();
        }
    }
}
```

### Redisson
```java
@Component
public class StockRedissonLockFacade implements StockCommand {
    private final RedissonClient redissonClient;

    private final StockService stockService;

    public StockRedissonLockFacade(RedissonClient redissonClient, StockService stockService) {
        this.redissonClient = redissonClient;
        this.stockService = stockService;
    }

    /**
     * Decrease stock quantity with Redisson Lock
     * @param id Product ID
     * @param quantity Quantity to decrease
     */
    @Override
    public void decreaseStockQuantity(Long id, Long quantity) {
        RLock lock = redissonClient.getLock(id.toString());

        try {
            // pub-sub Lock
            boolean isLocked = lock.tryLock(10, 1, TimeUnit.SECONDS);

            if (!isLocked) {
                throw new IllegalStateException("Failed to acquire lock");
            }

            stockService.decreaseStockQuantity(id, quantity);
        } catch (InterruptedException e) {
            Thread.currentThread().interrupt();
        } finally {
            lock.unlock();
        }
    }
}
```

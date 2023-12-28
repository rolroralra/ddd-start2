# 도메인 서비스
- 여러 Aggregate가 필요한 기능의 경우, 따로 도메인 서비스를 만들어서 구현하는 것이 좋다.

## 예시
- 주문 실제 결제 금액 계산 로직
- `상품`
  - 실제 상품 가격
- `주문`
  - 상품별 주문 개수
- `할인`
  - 적용된 할인
- `회원`
  - 회원 등급에 따른 추가 할인

> 실제 결제 금액을 계산해야하는 주체를 `주문` Aggregate로 할당하는 것보다,
> 
> 따로 실제 결제 금액을 계산하는 별도의 `도메인 서비스`를 구현하는 것이 좋다.

## 주로 사용되는 사례
- **계산 로직**
  - 여러 Aggregate가 필요한 계산 로직
  - 한 Aggregate가 책임지기에는 다소 복잡한 계산 로직
- **외부 시스템 연동이 필요한 도메인 로직**
  - 구현하기 위해 타 시스템을 사용해야 하는 도메인 로직

### 계산 로직과 도메인 서비스

- 주문 금액 계산
    - 도메인 서비스를 구현하고, 도메인 객체에 이를 파라미터로 전달하여 활용
```java
public class DiscountCalculationService {

    public Money calculateDiscountAmounts(
        List<Coupon> coupons,
        MemberGrade grade
    ) {
        Money couponDiscount = calculateDiscount(coupons);

        Money membershipDiscount = calculateDiscount(grade);

        return couponDiscount + membershipDiscount;
    }

    public Money calculateDiscount(List<Coupon> coupons) {
        // ...
    }

    public Money calculateDiscount(MemberGrade grade) {
        // ...
    }
}

public class Order {

    public void calculateAmounts(
        DiscountCalculationService discountCalculationService,
        MemberGrade grade
    ) {
        Money totalAmounts = getTotalAmounts();

        Money discountAmounts = discountCalculationService.calculateDiscountAmounts(this.coupons,
            grade);

        this.paymentAmounts = totalAmounts - discountAmounts;
    }
}

public class OrderService {

    private final DiscountCalculationService discountCalculationService;
    
    @Transactional
    public OrderNo takeOrder(OrderRequest req) {
        Order order = createOrder(orderRequest);

        orderRepository.save(order);
        
        return order.getOrderNo();
    }

    private Order createOrder(OrderRequest req) {
        Member member = memberRepository.findById(req.getOrdererId());
        Order order = new Order(/* ... */);
        order.calculateAmounts(this.discountCalculationService, member.getGrade());
        return order;
    }
}
```

- 계좌 이체 서비스
  - 도메인 서비스를 구현하고, 이 기능에 직접 Aggregate를 파라미터로 전달
```java
public class TransferService {
    public void transfer(Account fromAccount, Account toAccount, Money amounts) {
        fromAccount.withDraw(amounts);
        toAccount.credit(amounts);
    }
}
```

### 외부 시스템 연동과 도메인 서비스

# 도메인 서비스 패키지 위치
![image](https://velog.velcdn.com/images/aoqlsdl/post/de707213-c02a-4571-a18e-afea03f51f32/image.png)

![image](https://velog.velcdn.com/images/aoqlsdl/post/bcab4a5d-851d-492b-b32c-ea0b62b75764/image.png)

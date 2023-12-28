# Presentation, Application

![image](https://velog.velcdn.com/images/andy230/post/a7db3441-474c-436a-9e52-fe8ba50bd91b/image.png)

## Presentation
- 사용자의 요청을 해석한다.
- 사용자에게 알맞은 형식으로 응답한다.

## Application
- 사용자에게 원하는 기능을 제공한다.

# Application Layer
## 사용자의 요청을 처리
- `Repository`에서 도메인 객체(Aggregate) 조회
- Aggregate 메서드 실행 (도메인 로직 실행)
- 결과를 리턴

```java
public class ApplicationLayerExample {
  public DomainResult doSomeFunc(SomeRequest req) {
      // 1. Repository에서 Aggregate 조회
      SomeAgg agg = someAggRepository.findById(req.getId());
    
      // 2. Aggregate 도메인 기능 실행
      agg.doMethod(req.getValue());
    
      // 3. 결과를 리턴
      return createSuccessResult(agg);
  }
  
  public DomainResult doCreation(CreateRequest req) {
      // 1. 파라미터 검증
      validate(req);
      
      // 2. Aggregate 생성
      SomeAgg newAgg = createAggregate(req);
      
      // 3. Repository를 통해 Aggregate 저장
      somAggRepository.save(newAgg);
      
      // 4. 결과를 리턴
      return createSuccessResult(newAgg);
  }
}
```
## 트랜잭션 처리도 담당
- Spring에서 제공하는 `@Transactional`

## 도메인 로직을 Application Layer에 넣지 않기!
- 도메인 로직은 `Domain` 영역에 위치한다.
- `Application` 서비스는 도메인 로직을 구현하지 않는다!

```java
import org.springframework.security.crypto.password.PasswordEncoder;

public class ChangePasswordService {

  public void changePassword(MemberId memberId, String oldPassword, String newPassword) {
    Member member = memberRepository.findById(memberId)
            .orElseThrow(NotFoundMemberException::new);

    member.changePassword(oldPassword, newPassword, passwordPolicy, passwordEncoder);
  }
}

public class Member {

  public void changePassword(String oldPassword, String newPassword, PasswordPolicy passwordPolicy,
          PasswordEncoder passwordEncoder) {
    if (!matchPassword(oldPassword, passwordEncoder)) {
      throw new IllegalArgumentException("Password is not matched");
    }

    passwordPolicy.checkPassword(newPassword);

    setPassword(encodedNewPassword, passwordEncoder);
  }
  
  private boolean matchPassword(String password, PasswordEncoder passwordEncoder) {
      return passwordEncoder.matches(password, this.password);
  }

  private void setPassword(String password, PasswordEncoder passwordEncoder) {
    this.password = passwordEncoder.encode(password);
  }
}
```

### 도메인 로직을 도메인 영역과 응용 서비스에 분산해서 구현시, 발생하는 문제점
- **코드의 응집성이 떨어진다.**
  - 도메인 데이터를 변경하는 영역이 여러 개가 존재하게 된다.
  - 도메인 로직을 파악하기 위해 여러 영역을 분석해야 한다.
- **코드 중복이 발생한다.**
  - 여러 응용 서비스에서 동일한 도메인 로직을 구현할 가능성이 높아진다.

## 응용 서비스의 크기
- 한 응용 서비스 클래스에 1개의 도메인의 모든 기능 구현하기
  - 중복되는 코드를 private 메서드로 공통화 시켜, 코드 중복을 줄일 수 있다.
  - 하지만, 클래스 코드가 길어진다.
    - 또한 주입된 컴포넌트를 사용하지 않는 메서드도 존재하게 되어, 코드 품질이 떨어진다.
- 구분되는 기능별로 응용 서비스 클래스를 따로 구현하기
  - 같은 도메인 기능에서 사용하는 중복 코드는 따로 클래스를 선언하여, 코드 중복을 줄이자!
    - `MemberServiceHelper`

# Presentation Layer
## 표현 영역의 책임
- 사용자에게 도메인 서비스를 제공한다.
- 사용자의 요청을 알맞은 응용 서비스에 전달하고, 그 결과를 사용자에게 제공한다.
- 사용자의 세션, 인증, 권한을 관리한다.

## 값 검증
- 표현 영역
  - 필수 값, 값의 형식, 범위 등을 검증한다.
- 응용 서비스
  - 데이터의 존재 유무와 같은 논리적 오류를 검증한다.

### Spring에서 제공하는 기능
- BindingResult
- Errors
- Validator 인터페이스

## 인증, 권한 필터
- Spring Security

![image](https://velog.velcdn.com/images/andy230/post/6d7f6eea-3a2a-4ba7-9f33-b50ba047dccb/image.png)

## 조회 전용 기능 (CQRS)
![image](https://user-images.githubusercontent.com/43809168/100110332-2aac7880-2eb0-11eb-8df9-2e9a77dc9cc4.png)

---
title: "TDD라는 안전벨트 없이 운전하시겠습니까?"
date: 2026-02-06
categories:
  - Development
tags:
  - TDD
  - Spring Boot
  - 테스트
  - 아키텍처
toc: true
toc_sticky: true
---

## 들어가며

"테스트 코드? 시간 없어서 나중에 짜야지."

솔직히 말하면 나도 그랬다. 기능 구현이 급하니까 일단 코드부터 짜고 테스트는 여유 있을 때 하자고 생각했다. 그 "여유 있을 때"는 영원히 오지 않았다. 결국 버그가 터질 때가 되서야 디버깅하고 코드 수정하는 나를 발견했다.

이번에 회원 기능을 TDD로 구현해보면서 왜 테스트를 먼저 짜야 하는지 몸소 깨달았다. 테스트 코드는 **안전벨트**와 같다. 사고가 나기 전엔 불편하게만 느껴지지만 한 번 경험하고 나면 없이는 못 다닌다.

> Spring Boot + TDD로 회원가입, 내 정보 조회, 비밀번호 수정 기능을 구현하면서 배운 내용을 정리합니다.

---

## TDD 기본 개념

### Red-Green-Refactor 사이클

TDD의 핵심은 간단하다.

```
Red    → 실패하는 테스트 먼저 작성
Green  → 테스트를 통과하는 최소한의 코드 작성
Refactor → 중복 제거, 코드 개선
```

중요한 건 **"테스트 없이 프로덕션 코드를 작성하지 않는다"**는 원칙이다. 그리고 **오버엔지니어링 금지**. 현재 테스트를 통과하기 위한 최소 코드만 작성한다.

처음엔 답답했다. "그냥 한 번에 다 짜면 되는데 왜 이렇게 돌아가지?" 하지만 막상 해보니까, 이 사이클이 코드의 방향을 잡아주는 **네비게이션** 역할을 했다.

### 3A 패턴 (Arrange-Act-Assert)

테스트 코드를 작성할 때 가장 많이 쓰는 패턴이다. Given-When-Then이라고도 부른다.

```java
@Test
void throwsBadRequest_whenLoginIdAlreadyExists() {
    // Arrange (Given) - 테스트 준비
    fakeMemberReader.addExistingLoginId("existingUser");

    // Act & Assert (When & Then) - 실행 및 검증
    assertThatThrownBy(() ->
        memberService.register("existingUser", "password123!", "홍길동",
            LocalDate.of(1990, 1, 15), "test@email.com"))
        .isInstanceOf(CoreException.class);
}
```

- **Arrange**: 테스트에 필요한 데이터나 상태를 준비
- **Act**: 테스트 대상 메서드 실행
- **Assert**: 결과가 예상과 일치하는지 검증

이 구조를 지키면 테스트가 읽기 쉬워진다. 누가 봐도 "아, 이미 존재하는 로그인 ID로 가입하면 예외가 터지는구나" 하고 이해할 수 있다.

### Inside-Out vs Outside-In

TDD에는 두 가지 접근법이 있다.

| 방식 | 설명 | 순서 |
|------|------|------|
| **Outside-In** | 바깥(API)부터 안쪽으로 | E2E → Controller → Service → Domain |
| **Inside-Out** | 안쪽(Domain)부터 바깥으로 | Domain → Service → Controller → E2E |

나는 **Inside-Out** 방식을 선택했다.

```
1. MemberTest        → 도메인 생성/검증 로직 단위 테스트
2. MemberServiceTest → 중복체크 + 저장 조율 테스트
3. MemberV1ApiE2ETest → 전체 API 흐름 테스트
```

**왜 Inside-Out인가?**

핵심은 **"의존성이 없는 것부터 만든다"**는 점이다.

```
Domain (의존성 없음) → Service (Domain에만 의존) → Controller (Service에 의존)
```

도메인 레이어는 외부 의존성이 없다. 순수 Java 코드라서 테스트할 때 Mock이 거의 필요 없다. `Member.create()`를 테스트할 때 DB도 필요 없고, Spring Context도 필요 없다. 그냥 `new`해서 검증하면 끝이다.

반면 Outside-In으로 E2E부터 시작하면? 아직 Service도 없고, Domain도 없는데 전체 흐름을 테스트해야 한다. 결국 모든 걸 Mock으로 채워야 하고, Mock이 실제 구현과 다르게 동작하면 나중에 통합할 때 버그가 터진다.

**Inside-Out의 장점:**
- 의존성이 없는 도메인부터 **견고하게** 만들 수 있다
- Mock/Stub 사용이 최소화된다 (도메인 테스트는 거의 필요 없음)
- 도메인이 확립되면 상위 레이어는 **도메인을 믿고** 구현할 수 있다

도메인이 탄탄해야 그 위에 쌓는 것들도 견고해진다.

---

## 테스트 더블 (Test Doubles)

테스트를 작성하다 보면 외부 의존성(DB, 외부 API, 암호화 모듈 등)이 발목을 잡는다. 이때 **테스트 더블**을 사용한다.

> 테스트 대상이 의존하는 외부 객체의 동작을 **빠르고 안전하게 흉내 내는 대역 객체**

### 종류와 차이점

| 역할 | 설명 | 사용 시점 |
|------|------|----------|
| **Stub** | 미리 정해진 값 반환 | 외부 의존성의 결과값이 필요할 때 |
| **Mock** | 호출 여부/횟수 검증 | 행위 검증이 필요할 때 |
| **Spy** | 실제 객체 + 일부 대체 | 실제 동작 + 특정 메서드만 변경 |
| **Fake** | 간단한 실제 구현체 | InMemoryRepository 등 |

**중요한 포인트:** `mock()`, `spy()`는 **도구**이고, Stub/Mock/Spy/Fake는 **역할**이다. Mockito의 `mock()` 함수로 Stub 역할을 할 수도 있고, Mock 역할을 할 수도 있다.

### Stub 예시 - PasswordEncoder

도메인 테스트에서 실제 BCrypt를 쓰면 느리고 결과값 예측도 어렵다. 그래서 Stub을 만들었다.

```java
private final PasswordEncoder stubEncoder = new PasswordEncoder() {
    @Override
    public String encode(String rawPassword) {
        return "encoded_" + rawPassword;  // 예측 가능한 값
    }

    @Override
    public boolean matches(String rawPassword, String encodedPassword) {
        return encodedPassword.equals("encoded_" + rawPassword);
    }
};
```

이렇게 하면 `encode("password123")`의 결과가 항상 `"encoded_password123"`이다. 예측 가능하고 빠르다.

### Fake 예시 - MemberReader

서비스 테스트에서 실제 DB를 쓰면 느리고 환경 세팅이 번거롭다. Fake로 인메모리 구현체를 만들었다.

```java
static class FakeMemberReader implements MemberReader {
    private final Map<String, Boolean> existingLoginIds = new HashMap<>();

    void addExistingLoginId(String loginId) {
        existingLoginIds.put(loginId, true);
    }

    @Override
    public boolean existsByLoginId(String loginId) {
        return existingLoginIds.containsKey(loginId);
    }

    @Override
    public Optional<Member> findByLoginId(String loginId) {
        return Optional.empty();  // 필요하면 구현
    }
}
```

테스트에서 `fakeMemberReader.addExistingLoginId("existingUser")`로 상태를 셋업하고 서비스 로직이 중복 체크를 제대로 하는지 검증할 수 있다.

---

## 아키텍처 설계

### Layered Architecture

```
Interfaces (Controller, DTO)
    ↓
Application (Facade, Info)
    ↓
Domain (Entity, Service, Reader/Repository 인터페이스)
    ↓
Infrastructure (JPA Repository, 구현체들)
```

| 레이어 | 책임 | 예시 |
|--------|------|------|
| **Interfaces** | HTTP 요청/응답 처리 | MemberV1Controller, MemberV1Dto |
| **Application** | 유스케이스 조합 | MemberFacade (Service 호출 + Info 변환) |
| **Domain** | 비즈니스 로직 | Member 엔티티, MemberService |
| **Infrastructure** | 외부 시스템 연동 | MemberJpaRepository, PasswordEncoderImpl |

### Facade 패턴 - 정말 필요한가?

솔직히 처음엔 의문이 들었다. Facade가 하는 일이 이거뿐인데?

```java
public MemberInfo register(String loginId, String rawPassword,
                          String name, LocalDate birthDate, String email) {
    Member member = memberService.register(loginId, rawPassword, name,
        birthDate, email);
    return MemberInfo.from(member);  // 이게 전부?
}
```

단순히 Service 호출하고 DTO 변환하는 게 끝이다. 오버엔지니어링 아닌가?

하지만 생각해보면, **복잡한 Use Case가 생기면 빛을 발한다.**

```java
// 주문 생성 시 여러 서비스 조합이 필요한 경우
public OrderInfo createOrder(OrderCommand command) {
    inventoryService.checkStock(command.getItems());      // 1. 재고 확인
    Payment payment = paymentService.process(command);    // 2. 결제 처리
    Order order = orderService.create(command, payment);  // 3. 주문 생성
    notificationService.sendConfirmation(order);          // 4. 알림 발송
    return OrderInfo.from(order);
}
```

현재는 단순하지만 **일관된 아키텍처**를 위해 유지하기로 했다. 처음부터 Controller가 여러 Service를 직접 호출하면 나중에 리팩토링 비용이 커진다.

### Reader/Repository 분리 (CQRS 스타일)

보통 JPA를 쓰면 하나의 Repository에서 조회와 저장을 모두 처리한다.

```java
// 일반적인 방식 - 하나의 인터페이스에 모든 것
public interface MemberRepository extends JpaRepository<Member, Long> {
    Optional<Member> findByLoginId(String loginId);
    boolean existsByLoginId(String loginId);
    // save()는 JpaRepository에서 상속
}
```

나는 이걸 **조회(Reader)와 저장(Repository)으로 분리**했다.

```java
// 조회 전용 인터페이스
public interface MemberReader {
    Optional<Member> findByLoginId(String loginId);
    boolean existsByLoginId(String loginId);
}

// 저장/수정 전용 인터페이스
public interface MemberRepository {
    Member save(Member member);
}
```

**왜 굳이 분리했을까?**

사실 이 프로젝트 규모에서는 오버엔지니어링처럼 보일 수 있다. 하지만 몇 가지 이유가 있었다.

**1. 테스트할 때 필요한 것만 Fake로 만들 수 있다**

```java
// MemberService 테스트 시
class MemberServiceTest {
    private FakeMemberReader fakeMemberReader;      // 조회만 Fake
    private FakeMemberRepository fakeMemberRepository;  // 저장만 Fake

    @Test
    void 중복_로그인ID_체크() {
        // Reader만 셋업하면 됨, Repository는 신경 안 써도 됨
        fakeMemberReader.addExistingLoginId("existingUser");
        // ...
    }
}
```

하나의 Repository에 모든 메서드가 있으면 Fake를 만들 때 사용하지 않는 메서드도 다 구현해야 한다.

**2. 의존성이 명확해진다**

```java
// 이 서비스는 조회만 한다는 게 시그니처에서 보임
public class MemberQueryService {
    private final MemberReader memberReader;  // 조회만 의존
}

// 이 서비스는 저장도 한다
public class MemberCommandService {
    private final MemberReader memberReader;
    private final MemberRepository memberRepository;  // 저장도 의존
}
```

클래스가 어떤 역할을 하는지 생성자만 봐도 알 수 있다.

**3. 나중에 CQRS로 확장 가능**

지금은 둘 다 같은 DB를 바라보지만 트래픽이 많아지면 읽기 DB와 쓰기 DB를 분리할 수 있다. 그때 Reader와 Repository 구현체만 바꾸면 된다.

| 장점 | 단점 |
|------|------|
| 읽기/쓰기 책임 분리 (SRP) | 인터페이스가 늘어남 |
| 테스트 시 필요한 것만 Fake 가능 | 작은 프로젝트에선 오버엔지니어링 |
| 의존성이 명시적으로 드러남 | 러닝 커브 (팀원 설득 필요) |
| CQRS 확장 용이 | 당장은 이점이 안 보일 수 있음 |

**결론:** 현재 규모에서는 과할 수 있지만 테스트 작성이 편해지고 의존성이 명확해지는 장점 때문에 채택했다. "지금 당장"보다 "앞으로의 확장성"을 고려한 선택이다.

---

## 고민했던 결정들

### User vs Member 네이밍

엔티티 이름을 뭘로 할까 고민했다.

| 이름 | 장점 | 단점 |
|------|------|------|
| `User` | 일반적, 익숙함 | Spring Security `User`와 충돌, 너무 포괄적 |
| `Member` | 도메인 용어와 일치 | 덜 익숙할 수 있음 |

**결정: `Member` 사용**

- **도메인 용어**: "회원가입", "회원 정보" 등 비즈니스 용어와 일치
- **충돌 방지**: `org.springframework.security.core.userdetails.User`와 import 충돌 없음
- **명확한 의미**: User는 "사용자" (시스템 관점), Member는 "회원" (비즈니스 관점)

### 도메인 로직 위치 - Rich Domain Model

비밀번호 변경 로직을 어디에 둘지 고민했다.

**옵션 1: 엔티티에 위치**
```java
// Member.java
public void changePassword(String currentPassword, String newRawPassword,
                          PasswordEncoder encoder) {
    if (!encoder.matches(currentPassword, this.password)) {
        throw new CoreException(ErrorType.BAD_REQUEST, "현재 비밀번호가 일치하지 않습니다.");
    }
    validatePassword(newRawPassword, this.birthDate);
    this.password = encoder.encode(newRawPassword);
}
```

**옵션 2: 서비스에 위치**
```java
// MemberService.java
public void changePassword(Member member, String currentPassword, String newPassword) {
    if (!passwordEncoder.matches(currentPassword, member.getPassword())) {
        throw new CoreException(...);
    }
    member.setPassword(passwordEncoder.encode(newPassword));  // setter 노출 필요
}
```

| 기준 | 엔티티에 둘 때 | 서비스에 둘 때 |
|------|---------------|---------------|
| 캡슐화 | ✅ 내부 상태 보호 | ❌ setter 노출 필요 |
| 테스트 | ✅ 엔티티만 테스트 가능 | 서비스 + Mock 필요 |
| 의존성 | ❌ encoder를 전달해야 함 | ✅ 서비스가 직접 주입 |
| 일관성 | `Member.create()`와 동일 패턴 | 다른 패턴 |

**결정: 엔티티에 위치 (Rich Domain Model)**

단순히 "캡슐화가 좋다"는 이유만으로 결정한 건 아니다. 더 본질적인 이유들이 있다.

**1. 불변식(Invariant) 보장**

비밀번호에는 규칙이 있다. "8~16자", "생년월일 포함 불가" 같은 것들. 이 규칙은 **항상** 지켜져야 한다.

```java
// 엔티티에 로직이 있으면
member.changePassword(current, newPw, encoder);  // 무조건 검증됨

// 서비스에 로직이 있으면
memberService.changePassword(member, current, newPw);  // 이 서비스 통해야만 검증
member.setPassword("1234");  // 이렇게 우회하면? 검증 안 됨!
```

엔티티가 자기 자신의 상태를 항상 유효하게 유지할 책임을 갖는다. 어디서 호출하든 누가 호출하든 검증은 반드시 수행된다.

**2. 응집도 - 관련 데이터와 로직이 한 곳에**

비밀번호 검증 규칙 중 하나가 "생년월일을 포함할 수 없다"이다.

```java
validatePassword(newRawPassword, this.birthDate);  // birthDate가 필요함
```

`birthDate`는 Member의 속성이다. 이 검증 로직이 서비스에 있으면?

```java
// 서비스에서
validatePassword(newPassword, member.getBirthDate());  // getter로 꺼내야 함
```

데이터는 엔티티에 있고 로직은 서비스에 흩어진다. 엔티티에 로직이 있으면 **관련 있는 것들이 한 곳에 모인다**.

**3. 중복 방지**

나중에 "관리자가 회원 비밀번호를 강제로 변경하는 기능"이 추가된다면?

```java
// 서비스에 로직이 있을 때
class MemberService {
    void changePassword(...) { /* 검증 로직 */ }
}
class AdminService {
    void forceChangePassword(...) { /* 같은 검증 로직 복사? 아니면 MemberService 호출? */ }
}

// 엔티티에 로직이 있을 때
member.changePassword(...);  // 어디서든 이것만 호출하면 됨
```

**4. 코드가 문서가 된다**

```java
public class Member {
    public void changePassword(String currentPassword, String newRawPassword,
                              PasswordEncoder encoder) {
        // 현재 비밀번호 확인 필수
        // 새 비밀번호는 생년월일 포함 불가
        // ...
    }
}
```

Member 클래스만 읽으면 "비밀번호 변경 시 어떤 규칙이 적용되는지" 바로 알 수 있다. 서비스 여기저기 뒤질 필요 없다.

---

물론 트레이드오프는 있다. encoder를 파라미터로 넘겨야 하는 게 어색할 수 있다. 하지만 `Member.create()`에서 이미 이 패턴을 쓰고 있어서 일관성을 유지하기로 했다.

### PasswordEncoder 인터페이스 분리

Spring Security의 `PasswordEncoder`를 직접 쓰면 안 될까?

```
❌ Domain → Spring Security (외부 의존)
✅ Domain ← Infrastructure (의존성 역전)
```

도메인 레이어에 커스텀 인터페이스를 정의하고, Infrastructure에서 구현했다.

```java
// domain/member/PasswordEncoder.java (인터페이스)
public interface PasswordEncoder {
    String encode(String rawPassword);
    boolean matches(String rawPassword, String encodedPassword);
}

// infrastructure/member/PasswordEncoderImpl.java (구현)
@Component
public class PasswordEncoderImpl implements PasswordEncoder {
    private final BCryptPasswordEncoder bCryptPasswordEncoder = new BCryptPasswordEncoder();
    // ...
}
```

**장점:**
- 도메인을 순수 Java로 유지 → Stub으로 쉽게 테스트 가능
- Spring Security 버전 업그레이드해도 도메인은 영향 없음

---

## 테스트 전략과 실수

### E2E vs 단위 테스트의 역할

| 구분 | E2E 테스트 | 단위 테스트 |
|------|-----------|------------|
| 범위 | 전체 흐름 (Controller → DB) | 특정 클래스/메서드 |
| 속도 | 느림 | 빠름 |
| 신뢰도 | 높음 (실제 환경과 유사) | Mock 의존 |
| 디버깅 | 어려움 | 쉬움 |
| 용도 | 통합 검증, 회귀 테스트 | 로직 검증, 빠른 피드백 |

**원칙: 각 레이어의 테스트는 해당 레이어의 책임만 검증한다.**

```java
// MemberTest (단위 테스트) - 도메인 규칙 검증
@Test
void throwsBadRequest_whenPasswordIsTooShort() {
    // 비밀번호 8자 미만 → 여기서 상세 검증
}

// MemberV1ApiE2ETest (E2E 테스트) - API 흐름만 검증
@Test
void returnsBadRequest_whenNewPasswordInvalid() {
    // "short" 비밀번호로 API 호출 → 400 확인
    // 세부 규칙(8자 미만인지, 특수문자 문제인지)은 테스트 안 함
}
```

같은 검증을 여러 곳에서 하면 규칙이 바뀔 때 전부 고쳐야 한다. **도메인 규칙은 도메인 테스트에서 보장**하면 된다.

### 실수: E2E만 믿다가 단위 테스트 누락

이번에 실제로 실수한 부분이다.

E2E 테스트가 통과하니까 "끝났다!"고 생각했다. 하지만 `Member.changePassword()` 단위 테스트를 빠뜨렸다.

E2E 테스트에서 비밀번호 변경이 잘 되는 건 확인했지만, **도메인 로직 자체가 엣지 케이스를 다 커버하는지**는 검증하지 못했다.

교훈: **E2E 테스트는 "전체가 돌아간다"를 확인하고, 단위 테스트는 "로직이 정확하다"를 확인한다.** 둘 다 필요하다.

### 테스트 피라미드

```
        /\
       /  \      E2E 테스트 (적게, 핵심 시나리오만)
      /----\
     /      \    통합 테스트 (Repository + DB)
    /--------\
   /          \  단위 테스트 (많이, 빠르게)
  --------------
```

단위 테스트를 촘촘히 깔고 통합 테스트로 연결을 확인하고 E2E로 전체를 검증한다. 이 구조가 유지보수에 좋다.

---

## 마무리

### 배운 점

| 주제 | 핵심 |
|------|------|
| TDD | Red-Green-Refactor, 테스트가 설계를 이끈다 |
| 테스트 더블 | Stub/Mock/Spy/Fake는 역할, mock()/spy()는 도구 |
| 아키텍처 | 레이어를 나누면 테스트하기 쉬워진다 |
| 도메인 로직 | 가능한 엔티티에 (Rich Domain Model) |
| 테스트 전략 | E2E + 단위 테스트 병행, 각자 역할이 다르다 |

### 더 고민해볼 주제들

- **@Transactional**: Facade vs Service 어디에 붙여야 할까?
- **예외 처리**: 도메인 예외와 HTTP 상태 코드 분리?
- **동시성**: 비밀번호 동시 변경 요청이 들어오면?
- **Request 검증**: Bean Validation vs 도메인 검증 중복?

이 주제들은 다음 글에서 다룰 수 있으면 다뤄보기로ㅎㅎ

---

TDD를 처음 시작할 때 "테스트 먼저 짜는 게 뭐가 좋은데?"라는 의문이 있었다. 지금은 안다. 테스트는 코드의 **안전벨트**다. 없어도 운전은 되지만, 사고 나면 후회한다.

안전벨트 매고 출발하자. 테스트를 먼저 짜자.

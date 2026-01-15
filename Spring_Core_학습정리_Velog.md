# 🍃 Spring Core 원리 - 객체지향과 DI 완벽 이해하기

> Spring 강의를 들으며 회원 관리 시스템을 만들면서 배운 핵심 개념들을 정리했습니다.

## 🎯 들어가며

Spring을 제대로 이해하려면 **"왜 Spring을 사용하는가?"**에 대한 답을 알아야 합니다. 
단순히 프레임워크 사용법만 익히는 것이 아니라, **객체지향 설계 원칙**과 **의존성 주입(DI)**의 필요성을 코드로 직접 겪어보는 것이 중요합니다.

---

## 📌 핵심 학습 내용

### 1. 계층형 아키텍처 설계

실무에서 사용하는 3-Layer Architecture를 구현했습니다.

```
Controller (Presentation Layer)
    ↓
Service (Business Logic Layer)  
    ↓
Repository (Data Access Layer)
```

**왜 계층을 나누나요?**
- ✅ 각 계층을 독립적으로 수정 가능 (유지보수성 향상)
- ✅ 단위 테스트가 쉬워짐
- ✅ 역할과 책임이 명확해짐

---

## 💡 1. 인터페이스 기반 설계의 중요성

### 문제 상황: 구현체에 직접 의존하면?

처음에는 이렇게 코드를 작성했습니다:

```java
public class MemberServiceImpl implements MemberService {
    // ❌ 문제: 구체 클래스에 직접 의존
    private final MemberRepository memberRepository = new MemoryMemberRepository();
    
    @Override
    public void join(Member member) {
        memberRepository.save(member);
    }
}
```

**문제점:**
- 나중에 JPA나 JDBC로 변경하려면? → 코드 수정 필요
- **OCP(개방-폐쇄 원칙) 위반**: 확장에는 열려있어야 하지만 변경에는 닫혀있어야 함
- **DIP(의존관계 역전 원칙) 위반**: 추상화가 아닌 구체화에 의존

### 해결책: 인터페이스 + 생성자 주입

```java
// 1️⃣ 인터페이스 정의 (추상화)
public interface MemberRepository {
    void save(Member member);
    Member findById(Long memberId);
}

// 2️⃣ 구현체 (언제든 교체 가능)
public class MemoryMemberRepository implements MemberRepository {
    private static final Map<Long, Member> store = new HashMap<>();
    
    @Override
    public void save(Member member) {
        store.put(member.getId(), member);
    }
    
    @Override
    public Member findById(Long memberId) {
        return store.get(memberId);
    }
}

// 3️⃣ 생성자 주입으로 의존성 주입받기
public class MemberServiceImpl implements MemberService {
    private final MemberRepository memberRepository;
    
    // ✅ 외부에서 주입받음 (DI)
    public MemberServiceImpl(MemberRepository memberRepository) {
        this.memberRepository = memberRepository;
    }
    
    @Override
    public void join(Member member) {
        memberRepository.save(member);
    }
}
```

**이점:**
- ✅ MemoryMemberRepository → JpaMemberRepository 변경 시 서비스 코드는 수정 불필요
- ✅ 테스트 시 Mock 객체로 쉽게 대체 가능
- ✅ OCP, DIP 원칙 준수

---

## 💡 2. 불변 객체(Immutable Object) 설계

```java
public class Member {
    private final Long id;        // final로 불변성 보장
    private final String name;
    private final Grade grade;
    
    public Member(Long id, String name, Grade grade) {
        this.id = id;
        this.name = name;
        this.grade = grade;
    }
    
    // Getter만 제공, Setter는 없음
    public Long getId() { return id; }
    public String getName() { return name; }
    public Grade getGrade() { return grade; }
}
```

**왜 불변 객체를 사용하나요?**
- ✅ **멀티스레드 환경에서 안전**: 여러 스레드가 동시에 접근해도 안전
- ✅ **의도치 않은 변경 방지**: Setter가 없어서 실수로 값을 바꿀 수 없음
- ✅ **예측 가능한 코드**: 생성 시점의 값이 유지됨

---

## 💡 3. Enum을 활용한 타입 안전성

```java
public enum Grade {
    BASIC,  // 일반 회원
    VIP     // VIP 회원
}
```

**Enum의 장점:**
```java
// ❌ 문자열 사용 시 문제
String grade = "vip";  // 소문자로 오타
String grade = "VIIP"; // 철자 오류 - 컴파일 에러 안 남!

// ✅ Enum 사용 시
Grade grade = Grade.VIP;  // 컴파일 시점에 오류 체크
```

- ✅ 타입 안전성 보장
- ✅ IDE 자동완성 지원
- ✅ Switch 문에서 모든 케이스 처리 여부 체크 가능

---

## 💡 4. Spring의 핵심: AppConfig로 DI 구현

### 문제: 서비스가 직접 구현체를 선택하면?

```java
public class OrderServiceImpl implements OrderService {
    // ❌ 서비스가 구현체를 직접 선택
    private final DiscountPolicy discountPolicy = new RateDiscountPolicy();
}
```

**문제점:**
- 할인 정책 변경 시 OrderService 코드 수정 필요
- 서비스가 "사용"과 "선택" 두 가지 책임을 모두 가짐 (SRP 위반)

### 해결: AppConfig로 책임 분리

```java
@Configuration
public class AppConfig {
    
    // 의존성 주입 설정을 한 곳에서 관리
    @Bean
    public MemberService memberService() {
        return new MemberServiceImpl(memberRepository());
    }
    
    @Bean
    public OrderService orderService() {
        return new OrderServiceImpl(
            memberRepository(), 
            discountPolicy()  // 여기서 정책 선택!
        );
    }
    
    @Bean
    public MemberRepository memberRepository() {
        return new MemoryMemberRepository();
    }
    
    @Bean
    public DiscountPolicy discountPolicy() {
        // 정액 → 정률 할인으로 변경 시 여기만 수정!
        return new RateDiscountPolicy();
    }
}
```

**AppConfig의 역할:**
- ✅ **구성(Configuration) 영역**: 어떤 구현체를 사용할지 결정
- ✅ **실행(Service) 영역**: 비즈니스 로직에만 집중
- ✅ **SRP 준수**: 책임이 명확하게 분리됨

---

## 💡 5. 전략 패턴으로 할인 정책 구현

```java
// 전략 인터페이스
public interface DiscountPolicy {
    int discount(Member member, int price);
}

// 전략 1: 정률 할인 (10%)
public class RateDiscountPolicy implements DiscountPolicy {
    private int discountPercent = 10;
    
    @Override
    public int discount(Member member, int price) {
        if (member.getGrade() == Grade.VIP) {
            return price * discountPercent / 100;
        }
        return 0;
    }
}

// 전략 2: 정액 할인 (1000원)
public class FixDiscountPolicy implements DiscountPolicy {
    private int discountFixAmount = 1000;
    
    @Override
    public int discount(Member member, int price) {
        if (member.getGrade() == Grade.VIP) {
            return discountFixAmount;
        }
        return 0;
    }
}
```

**서비스에서 사용:**
```java
public class OrderServiceImpl implements OrderService {
    private final DiscountPolicy discountPolicy;
    
    public OrderServiceImpl(DiscountPolicy discountPolicy) {
        this.discountPolicy = discountPolicy;  // 어떤 정책이든 OK!
    }
    
    @Override
    public Order createOrder(Long memberId, String itemName, int itemPrice) {
        Member member = memberRepository.findById(memberId);
        int discountPrice = discountPolicy.discount(member, itemPrice);
        return new Order(memberId, itemName, itemPrice, discountPrice);
    }
}
```

**전략 패턴의 장점:**
- ✅ 런타임에 전략(정책) 변경 가능
- ✅ 새로운 할인 정책 추가 시 기존 코드 수정 불필요 (OCP)
- ✅ 각 전략이 독립적으로 테스트 가능

---

## 💡 6. Given-When-Then 테스트 패턴

```java
public class MemberServiceTest {
    MemberService memberService = new MemberServiceImpl();
    
    @Test
    void join() {
        // Given: 테스트 데이터 준비
        Member member = new Member(1L, "memberA", Grade.VIP);
        
        // When: 테스트할 동작 실행
        memberService.join(member);
        Member findMember = memberService.findMember(1L);
        
        // Then: 결과 검증
        Assertions.assertThat(member).isEqualTo(findMember);
    }
}
```

**테스트 3단계:**
1. **Given (준비)**: 테스트에 필요한 데이터를 준비
2. **When (실행)**: 실제 테스트할 동작을 실행
3. **Then (검증)**: 예상한 결과가 맞는지 검증

**사용 도구:**
- **JUnit 5**: Java 표준 테스트 프레임워크 (@Test 어노테이션)
- **AssertJ**: 가독성 좋은 assertion 라이브러리

---

## 🎓 배운 SOLID 원칙 정리

| 원칙 | 설명 | 적용 예시 |
|------|------|-----------|
| **SRP**<br>(단일 책임) | 한 클래스는 하나의 책임만 | `MemberService`는 회원 로직만,<br>`MemberRepository`는 데이터 접근만 |
| **OCP**<br>(개방-폐쇄) | 확장에는 열려있고<br>변경에는 닫혀있어야 함 | 인터페이스로 구현체 교체 시<br>클라이언트 코드 수정 불필요 |
| **LSP**<br>(리스코프 치환) | 하위 타입은 상위 타입을<br>대체 가능해야 함 | `MemberRepository` 인터페이스로<br>어떤 구현체든 사용 가능 |
| **ISP**<br>(인터페이스 분리) | 클라이언트는 사용하지 않는<br>메서드에 의존하지 않아야 함 | 필요한 메서드만 정의<br>(save, findById) |
| **DIP**<br>(의존관계 역전) | 구체화가 아닌<br>추상화에 의존해야 함 | `new MemoryRepository()` 대신<br>생성자로 주입받기 |

---

## 🤔 자주 하는 질문

### Q1. 인터페이스 없이 구현체만 사용하면 안 되나요?

```java
// ❌ 나쁜 예: 구체 클래스에 직접 의존
MemberServiceImpl service = new MemberServiceImpl();

// ✅ 좋은 예: 인터페이스에 의존
MemberService service = new MemberServiceImpl();
```

**A:** 
- 테스트가 어려워집니다 (Mock 객체로 대체 불가)
- 구현체 변경 시 모든 클라이언트 코드를 수정해야 함
- 다형성 활용 불가

### Q2. static final Map은 왜 사용하나요?

```java
private static final Map<Long, Member> store = new HashMap<>();
```

**A:**
- `static`: 모든 인스턴스가 같은 저장소를 공유
- `final`: HashMap 객체 자체를 다른 객체로 교체 불가
- ⚠️ 주의: Map 내부의 값은 변경 가능 (put, remove 등)

### Q3. 생성자가 2개인 이유는?

```java
public class MemberServiceImpl implements MemberService {
    private final MemberRepository memberRepository;
    
    // 기본 생성자
    public MemberServiceImpl() {
        this.memberRepository = new MemoryMemberRepository();
    }
    
    // DI용 생성자
    public MemberServiceImpl(MemberRepository memberRepository) {
        this.memberRepository = memberRepository;
    }
}
```

**A:**
- 기본 생성자: 간단한 테스트나 독립 실행 시 사용
- DI 생성자: Spring 컨테이너나 AppConfig에서 의존성 주입 시 사용
- 실무에서는 DI 생성자만 사용하는 것이 권장됨

---

## 📊 핵심 정리

### Spring을 사용하는 진짜 이유

```
1️⃣ 객체지향 설계 원칙(SOLID)을 쉽게 지킬 수 있게 도와줌
   → DI 컨테이너가 의존성을 자동으로 주입

2️⃣ 다형성만으로는 OCP, DIP를 지키기 어려움
   → AppConfig나 @Configuration으로 해결

3️⃣ 역할과 구현을 분리하면 유연한 설계 가능
   → 인터페이스 + 생성자 주입 패턴
```

### 설계 체크리스트

- [ ] 인터페이스와 구현체를 분리했나요?
- [ ] 구체 클래스가 아닌 인터페이스에 의존하나요?
- [ ] 생성자 주입으로 DI를 구현했나요?
- [ ] 각 클래스가 단일 책임을 가지나요?
- [ ] 테스트 코드를 작성했나요?

---

## 🚀 다음 학습 계획

- [ ] `@Autowired`를 사용한 자동 의존성 주입
- [ ] `@ComponentScan`으로 자동 빈 등록
- [ ] 생성자 주입 vs 필드 주입 vs Setter 주입 비교
- [ ] Bean Scope와 생명주기
- [ ] AOP (Aspect Oriented Programming)

---

## 💭 학습 후기

처음에는 "왜 이렇게 복잡하게 인터페이스를 만들어야 하지?"라고 생각했는데, 
실제로 할인 정책을 변경하면서 **AppConfig의 한 줄만 수정**하면 되는 것을 보고 감탄했습니다.

> "객체지향의 핵심은 다형성이다. 하지만 다형성만으로는 OCP, DIP를 지킬 수 없다.  
> Spring이 제공하는 DI 컨테이너가 이 문제를 해결한다!"

---

## 📚 참고 자료

- [Spring 공식 문서](https://spring.io/projects/spring-framework)
- [JUnit 5 Guide](https://junit.org/junit5/)
- [AssertJ Documentation](https://assertj.github.io/doc/)
- [객체지향의 사실과 오해 - 조영호](http://www.yes24.com/Product/Goods/18249021)

---

**태그:** `#Spring` `#객체지향` `#DI` `#SOLID` `#Java` `#SpringBoot`

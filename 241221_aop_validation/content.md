# Spring AOP를 이용하여 사용자 검증하기

## 1. 문제 상황

유저의 상세 정보를 조회하는 API 가 있다고 가정 해보자.  
endpoint는 GET /users/{id} 가 될 것이고, 로그인 검증이 필요할 것이다.  


**이 때, userA가 로그인한 상태에서 userB의 유저 정보를 요청하게 되면 어떻게 될까?** API를 잘못 설계하면 로그인 검증이 통과되었기 때문에 userB의 회원 정보를 그대로 보여줄 수도 있을 것이다.  


이런 상황을 방지 하기 위해 **'로그인한 사용자'** 와 **'요청을 보낸 사용자'** 가 같은지 한번 더 검증하는 로직이 추가 된다.  


## 2. 변경 전

앞서 얘기한 문제를 해결 하기 위해 기존 프로젝트에서는 다음과 같은 코드를 사용하고 있었다.  

```java
@GetMapping("/users/{userId}")
public ResponseEntity<FindUserResponse> findUserById(@PathVariable("userId") String userId) {
    loginIdVerification.validateRequestUserId(userId); //이와 같이 검증하는 코드가 추가됨.
    //...  사용자 정보 조회
    return ResponseEntity.ok(response);
}
```
아래는 검증하는 함수의 구현부이다.
```java
//로그인한 사용자와 요청을 시도한 사용자가 같은지 검증하는 함수
public void validateRequestUserId(String userId) {
    UserDetails principal = (UserDetails) SecurityContextHolder.getContext().getAuthentication().getPrincipal();
    if (!principal.getUsername().equals(userId)) {
        throw new IllegalArgumentException("잘못된 접근 입니다.");
    }
}
```

위의 코드는 기능적으로 아무 문제가 없다. 다만 이러한 아쉬움이 있었다.
- 횡단관심사가 비즈니스 로직과 혼재되어 있다. 
- 검증이 필요한 모든 Controller에서 검증 객체(위에서는 loginIdVerification)를 의존해야 한다.
- 검증이 필요한 곳에 거추장스런 중복 코드가 삽입되어야 한다.
- 멋이 없다.(중요)

## 3. Spring AOP 적용

횡단 관심사를 분리하기 위해 먼저 Spring AOP를 적용한다.

```java

@Aspect
@Component
public class RequestLoginIdValidationAspect {

    @Before("@annotation(xxx.xxx.RequestLoginIdValidation) && args(userId, ..)")
    public void beforeRequestLoginIdValidation(String userId) {
        UserDetails principal = (UserDetails) SecurityContextHolder.getContext().getAuthentication().getPrincipal();
        if (!principal.getUsername().equals(userId)) {
            throw new IllegalArgumentException("잘못된 접근 입니다.");
        }
    }
}

```
그리고 커스텀 어노테이션을 적용한다. 이 어노테이션은 단순히 **어디에 프록시를 적용**할지 결정하는 역할을 한다.

```java
@Target(ElementType.METHOD)
@Retention(RetentionPolicy.RUNTIME)
public @interface RequestLoginIdValidation {
}
```

## 4. 변경 후
기존의 Conroller는 다음과 같이 바뀌었다.
```java
@RequestLoginIdValidation //추가된 어노테이션
@GetMapping("/users/{userId}")
public ResponseEntity<FindUserResponse> findUserById(@PathVariable("userId") String userId) {
    //...  사용자 정보 조회
    return ResponseEntity.ok(response);
}
```
어노테이션 하나가 추가 되고, 더이상 검증 로직과 비즈니스 로직이 섞이지 않게 되었다.


## 5. 하지만

이 방법은 간편하고 코드가 깔끔하다. 하지만 **성능적으로 좋지 않다.**  
그렇다고 대단히 느려서 사용하면 안되는가? 그건 아니다.  
실제로 프로덕션에서 사용했을 때 크게 문제가 있을거라 생각하진 않는다.  

하지만 더 나은 방법이 있기에 [다음 글]()에서 소개해보도록 하겠다.

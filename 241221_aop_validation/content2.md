# 스프링 AOP 포인트컷 구현 시 성능 차이


## 1. 포인트컷 구현시 성능 차이
- 포인트컷이 조인포인트에 매칭되는지 판단하는 작업은 비용이 많이 든다.
- AspectJ는 매칭 퍼포먼스를 위해 최적화를 진행한다.
- 하지만 개발자가 작성한 포인트컷 표현식을 기반으로 최적화를 하기 때문에 근본적으로 포인트컷을 잘 작성하는 것이 중요하다


## 2. [Writing Good PointCuts(스프링 공식문서)](https://docs.spring.io/spring-framework/reference/core/aop/ataspectj/pointcuts.html#writing-good-pointcuts)
문서를 간단히 요약 하자면
- 포인트컷은 다음과 같이 3가지 종류가 있고 이들을 잘 조합하여 가능한 범위를 좁혀야 한다.
- Kinded 지정자와 Scoped 지정자를 함께 지정해 주는 것이 가장 좋다.

| 제목             | 설명                                  | 예시                |
|----------------|-------------------------------------|-------------------|
| Kinded 지정자     | 조인포인트의 동작을 지정(주로 메서드 실행)            | execution         |
| Scoped 지정자     | 조인포인트의 범위를 지정(패키지 범위, 메서드 이름 등)     | within            |
| Contextual 지정자 | 조인포인트의 동적 조건을 지정(메서드 파라미터, 어노테이션 등) | args, @annotation |

그런데 여기서 **문제가 되는 것은 마지막 Contextual 지정자** 이다.  
이 친구는 동적 상태를 담고 있는 경우가 많아서 메서드 호출 시점에야 매칭 판단이 가능한 경우가 있다.  
이에 대해 자세히 알아 보자.

## 3. 포인트컷 매칭 시점
포인트컷 매칭 시점은 2가지로 나뉜다. 스프링 초기화 시점, 메서드 호출 시점 이다.

### 3-1. 스프링 초기화 시점
대부분의 포인트컷 매칭은 스프링 초기화 시점에 이뤄 진다.  
예를 들어 `execution(* MyService.methodB(..))`과 같은 포인트 컷 표현식은  
MyService 클래스의 methodB에 대해 매칭한다는 뜻이다. 이는 **클래스, 메서드 선언 등의 정적인 정보만으로 판단이 가능하다.**  
따라서 스프링이 초기화되고 프록시가 생성되는 시점에 포인트컷 매칭이 이루어 진다.(즉, 어드바이스가 적용된다.)

### 3-2. 메서드 호출 시점
하지만 일부 포인트컷 매칭은 스프링 메서드 호출 시점에야 이루어 진다. 스프링 초기화 시점에는 포인트컷 적용에 대한 정보가 부족하기 때문이다.  
대표적인 것인 메서드의 파라미터 정보이다. 예를 들어 메서드의 파라미터가 A일 때와 B일 때 판단이 달라진다고 해보자.  
**파라미터가 무엇이 넘어 올지는 초기화 시점에는 알 수 없기 때문에** 호출이 되고 나서야 판단할 수 있는 것이다.

> 참고 - 메서드 호출 시점에만 판단 가능한 포인트컷 종류
- args : 메서드 파라미터는 호출 시점에 매칭 가능하다.
- @annotation : 어노테이션은 경우에 따라 다르다.
  - 어노테이션의 속성 값이 없으면 초기화 시점에 매칭이 가능하다.
  - 속성 값이 있는 경우에는 어떤 값이 넘어 올지 모르기 때문에 호출 시점에 매칭된다.
  - 몇 가지 실험하다 알게 된 것은 메서드 파라미터에 어노테이션이 붙을 경우에는 속성값 유무에 상관 없이 호출 시점에 매칭된다.

## 4. 호출 시점에 포인트컷 매칭이 이루어 진다는 것의 의미

### 4-1. 매칭 판단이 한번 더 일어 난다.

```java
public interface MethodMatcher {

	boolean matches(Method m, Class<?> targetClass);

	boolean isRuntime(); //isRuntime() 이 true를 리턴할 경우

	boolean matches(Method m, Class<?> targetClass, Object... args); //이 메서드가 메서드 호출 시마다 호출 된다.
}
```
[matches 동작에 대한 공식문서 설명](https://docs.spring.vmware.com/spring-framework/reference/core/aop-api/pointcuts.html?)  
위의 함수는 포인트컷이 매칭되는지 판단할 때 사용되는 함수들이다.  
첫 번째 matches 함수는 초기화 시점에 호출된다.  
하지만 첫 번째 matches 함수가 true이고, isRuntime()이 true일 때 세번 째 matches 함수가 호출 시마다 호출된다.  
즉, 포인트컷 매칭을 판단하는 함수가 2번 호출되는 것이다.


### 4-2. 리플렉션으로 인한 오버헤드가 발생한다.
args와 같은 포인트컷으로 런타임에 파라미터 정보를 가져 온다는 것은 리플렉션이 사용된다는 것이다.  
리플렉션을 사용하게 되면 일반적인 메서드 호출에 비해 오버헤드가 발생한다.


## 5. 결론
포인트컷 표현식을 사용할 때는 매칭 오버헤드가 적은 kinded, scoped 지정자를 이용하자.  

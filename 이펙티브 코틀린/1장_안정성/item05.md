# item5 예외를 활용해 코드에 제한을 걸어라

많이들 사용해서 코틀린에서 DSL로 만들어준 블록들이 소개된다.
- require (인자검사)
- check (상태검사)
- assert (테스트 이건 왜들어갔는지 모르겠음 ???)
- ?: return/throw

해당 구문들을 사용하는건 뭐 당연지사고 숨쉬듯이 사용할것이다.  
그리고 해당 구문들은 DSL일뿐 그냥 조건검사후 return이든, throw든 그 실행흐름을 탈출할 수 있으니   
스마트 캐스트를 보장할뿐이다. 즉 읽기전용과 조합해서 사용하는것을 고려해야한다.  
결론적으로 이것 또한 이름이 중요하다(이름에 맞는 행동을 하자)  

이번 아이템에서 진정으로 다시 생각해봐야할 문제는  
fast-fail / fail-safe인거 같다.  

클라 개발자 입장에서 크리티컬하게 문제가 될 로직이 과연 많은가?(당연히 크리티컬 하다면 throw 해야겠지만)  
오히려 크래시가 더 크리티컬 할것이다.  
고로 예외를 활용하는것보다 함수 분리와 함께하는 return 문을 사용하는 편이 더 고려되어야하지 않을까 싶다.    
(옛날에는 무지성으로 require 좋아? 그럼 쓴다 하고 여기저기 똥칠했던 추억이...)  

그리고 스마트 캐스팅을 진짜 스마트하게 이용해야 좋은 코틀린 개발자가 될수있을것이다.  
스터디 라이브 코딩떄 시연했던 요소들이 스마트 캐스팅을 잘이용할수 있게 해주는 키가될것이다.  
- 단위별 함수분리
- early return
- val(읽기전용)변수 이용

### assert??
왜 초심자 헷갈리게 넣어놨는지 모르겠다.  
프로덕션에는 안쓰이고 왠만해서 그리고 심지어 jvm기반에서만(Kotlin/js에서 못씀) 쓸수있는데다가   
딱히 명확히 용도를 가지고 동작도 안하는데 나는 이거 일회독 시점에 뭔소리인지 못알아들었다.  

그리고 안드로이드 junit4만 계속 지원해서 열받아서 junit 못쓰겠다.  
안드개발자라면 지원도 안해주는 junit쓰지말고 유행따라 적어도 assertion이라도 kotest-assertion쓰는게 편하더라  

심지어 kotest-assertion은 dsl이라 assert이런 네이밍도 아니다.  
쓸데없으니 그냥 무시하고 넘어가자  


### contract
갑자기 예시에 띡하고 나오는데 스마트 캐스팅을 직접 다루기위해서 사용하는 키워드이다.  

생각보다 이런것들 적용해서 공통코드를 구성해놓으면 사내에서 거부감이 많았는데  
우리 회사가 워낙 보수적이라 그런건가 싶기도 하고 다른 회사들은 어떤지 모르겠다.  
(유틸코드가 반대인거라면 나도 동의한다. SRP에 위배되기 쉬워서 개인적으로 유틸은 엄청난 고민과 번뇌속에서 만들어져야한다고 생각한다.)  

근데 어쩃든 이거 맨날쓰는게 아니라 너무 맨날 헷갈리고 귀찮아서 클로드한테 또 학습자료 만들어와 시켰다 ㅎㅎ  
그냥 스마트 캐스팅 직접만들어야겠다 불편하다 싶으면 와서 보고 만들면된다.  

<details>
<summary>contract 설명</summary>

# Kotlin Contract 설명

Contract는 컴파일러에게 "이 함수가 이렇게 동작한다"는 추가 정보를 제공해서, 컴파일러가 더 똑똑하게 스마트 캐스팅이나 변수 초기화 분석을 할 수 있게 해주는 기능입니다.

## 1. Returns Contract (조건부 반환)

스마트 캐스팅이 함수 경계를 넘어서도 작동하게 해줍니다.

```kotlin
import kotlin.contracts.*

@OptIn(ExperimentalContracts::class)
fun String?.isNotNullOrEmpty(): Boolean {
    contract {
        returns(true) implies (this@isNotNullOrEmpty != null)
    }
    return this != null && this.isNotEmpty()
}

fun main() {
    val name: String? = getName()

    if (name.isNotNullOrEmpty()) {
        // contract 덕분에 여기서 name이 String으로 스마트 캐스팅됨
        println(name.length)  // 안전하게 호출 가능
    }
}
```

**Contract 없이는?** 컴파일러가 `isNotNullOrEmpty()` 내부 로직을 모르기 때문에 스마트 캐스팅이 안 됩니다.

## 2. 타입 체크 함수

```kotlin
@OptIn(ExperimentalContracts::class)
fun Any?.isString(): Boolean {
    contract {
        returns(true) implies (this@isString is String)
    }
    return this is String
}

fun process(value: Any?) {
    if (value.isString()) {
        // String으로 스마트 캐스팅
        println(value.uppercase())
    }
}
```

## 3. CallsInPlace Contract (람다 호출 보장)

람다가 정확히 몇 번 호출되는지 컴파일러에게 알려줍니다.

```kotlin
@OptIn(ExperimentalContracts::class)
inline fun <R> myRun(block: () -> R): R {
    contract {
        callsInPlace(block, InvocationKind.EXACTLY_ONCE)
    }
    return block()
}

fun example() {
    val x: Int
    myRun {
        x = 10  // contract 덕분에 초기화 보장됨
    }
    println(x)  // 컴파일 OK
}
```

**InvocationKind 옵션:**

| 옵션            | 설명                |
| --------------- | ------------------- |
| `EXACTLY_ONCE`  | 정확히 1번          |
| `AT_LEAST_ONCE` | 1번 이상            |
| `AT_MOST_ONCE`  | 0번 또는 1번        |
| `UNKNOWN`       | 알 수 없음 (기본값) |

## 4. 실용적인 예시: require/check 패턴

```kotlin
@OptIn(ExperimentalContracts::class)
fun requireNotNull(value: Any?, message: String): Unit {
    contract {
        returns() implies (value != null)
    }
    if (value == null) throw IllegalArgumentException(message)
}

fun processUser(user: User?) {
    requireNotNull(user, "User must not be null")
    // 여기서부터 user는 non-null로 스마트 캐스팅
    println(user.name)
}
```

## 주의사항

1. **`@OptIn(ExperimentalContracts::class)`** 필요 (아직 실험적 기능)
2. Contract는 함수 본문의 **맨 처음**에 선언해야 함
3. 컴파일러는 contract를 **검증하지 않음** - 거짓 contract를 작성하면 런타임 오류 발생
4. 주로 **inline 함수**와 함께 사용

## 표준 라이브러리의 Contract 활용 함수들

`require()`, `check()`, `requireNotNull()`, `checkNotNull()`, `run()`, `let()`, `apply()` 등이 내부적으로 contract를 사용합니다.

</details>

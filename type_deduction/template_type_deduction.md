# Chapter 1. Deducting Types

## Item 1: Understand template type deduction

`auto` 키워드의 타입추론(type deduction)은 템플릿의 타입추론에 기반을 둔다.

```cpp
template<typename T>
void f(ParamType param);

f(expr);
```

컴파일 타임에 컴파일러는 `expr` 을 이용해 `T` 와 `ParamType` 의 타입을 추론한다.
이때`ParamType` 은 cv-qualifier(const, reference)와 같은 수식자(adornments)를 포함해 종종 두개의 타입 추론이 다르기도 하다.
```cpp
template<typename T>
void f(const T& Param);

int x = 0;
f(x);
```

위와 같은 코드에서 `T `는 int로, ParamType은 const int&로 추론이 된다.
함수에 넘겨지는 인자와 `T`의 타입은 항상같다고 생각할수 있지만 이는 항상 그런건 아니다. `T` 의 타입추론은 `expr` 뿐만 아니라 `ParamType` 의 형태와도 관련이 있다.

### Case1: ParamType is Reference / Pointer, not Universal(Forward)

1. `expr` 의 타입이 레퍼런스라면(reference), 레퍼런스 부분을 무시한다.
2. `expr` 과 `ParamType` 의 패턴을 비교하여 `T` 의 타입을 결정한다.

```cpp
template<typename T>
void f1(T& param);

template<typename T>
void f2(const T& param);

int x = 27;         
const int cx = x;  
const int& rx = x;
```

left: T type,  right: param type
|    |        x        |           cx          |           rx          |
|:--:|:---------------:|:---------------------:|:---------------------:|
| f1 |    int, int&    | const int, const int& | const int, const int& |
| f2 | int, const int& |    int, const int&    |    int, const int&    |

`f2` 에서 `param` 의 형태에 `const` 가 존재하기 때문에 템플릿 파라미터(template paramter) 타입 추론에 `const` 가 필요없다.

### Case2: ParamTyper is Universal(Forward) Reference

- `expr`이 lvalue인 경우, `T` 와 `ParamType` 은 lvalue reference로 추론된다.
- `expr`이 rvalue인 경우, Case1과 같은 룰이 적용된다.

```cpp
template<typename T>
void f(T&& param);

int x = 27;
const int cx = x;
const int& rx =x;
```

Effective Modern C++ [Item 24] 참고

### Case3: ParamType is Neither a Pointer nor a Reference

```cpp
template<typename T>
void f(T param);
```

위 함수의 `param` 은 무엇이 전달되든 복사된 완벽한 새로운 객체라는 것이다. 이는 Case3 에서의 타입추론에 매우 중요한 근거가 된다.

1. `expr` 이 레퍼런스라면, 레퍼런스를 무시한다.
2. 1번 이후 `expr` 이 const 라면 이 또한 무시한다.(volatile 또한 마찬가지이다.)

```cpp
int x = 27;
const int cx = x;
const int& rx = x;

f(x);
f(cx);
f(rx);
```

위 세 함수호출의 타입추론 결과는 모두다 `T` & `param` : int 이다. `param` 은 함수인자로 부터 복사된 완벽히 독립적인 새로운 객체이기 때문에 `param` 의 변경은 인자들과 관계가 없기때문에 const가 무시된다.

```cpp
template<typenmae T>
void f(T parma);

const char* const ptr = "Fun with pointers"

f(ptr);
```

`ptr` 이 함수 `f` 에 넘어갈때 포인터 자체가 값으로 넘어가는 것이기 때문에 포인터의 상수성은 무시되고 포인터가 가르키는 객체의 상수성은 무시되지 않는다. 즉 T는 `const char*` 로 추론이 된다.

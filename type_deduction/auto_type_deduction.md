## Item 2: Understand auto type deduction

`auto` 의 타입추론은 한가지를 제외하곤 [Effective Modern C++ - Item1] 에서 다룬 템플릿 타입추론과 매우 유사하다. 앞선 템플릿 타입추론은 템플릿과 함수, 매개변수의 관계로 추론이 되었지만, `auto` 키워드는 이것들을 취급하진 않는다.

```cpp
template<typename T>
void f(ParamType param);

f(expr);

auto x = 27;
```

위의 예에서 `auto` 는 함수 템플릿의 `T` 의, 변수의 타입 식별자(type specifer)는 `ParamType` 처럼 동작한다.

```cpp
auto x = 27;            // ts(type specifer): auto by itself

const auto cx = x;      // ts: const auto

const auto& rx = x;     // ts: const auto&
```

- Case 1: type specifier is a pointer or reference, but not a universal reference.
- Case 2: type specifer: universal reference.
- Case 3: type specifer: neither a pointer nor reference.

### Exception

C_++1_1 에서는 새로운 변수 초기화 문법(uniform initialization)이 등장한다.

```cpp
// C++98
int x1 = 27;
int x2(27);

// C++11, uniform initialization
int x3 = { 27 };
int x4{ 27 };
```

위 예제에서 `x1, x2, x3, x4` 는 모두 int형이며 27로 초기화가된다. 이때 int를 auto로 바꾸어 예제를 컴파일해보면 컴파일은 정상적으로 된다. 하지만 컴파일 결과는 위와는 조금 다른 모습이 나타난다.

```cpp
auto x1 = 27;           // int
auto x2(27);            // int
auto x3 = { 27 };       // std::initializer_list<int>
auto x4{ 27 };          // std::initializer_list<int>
```

이는 `auto` 타입의 변수가 `{}` 로 초기화가 되면 `std::initializer_list` 로 타입추론이 되는 특별한 규칙때문이다. 이를 정리하자면 템플릿과 auto의 타입추론에 다른점 하나라면 `auto` 는 `{}` 를 `std::initializer_list` 로 추론을 한다는 것이도 템플릿은 추론을 못한다는 것이다.

C++14 에선 함수 리턴 타입, 람다의 매개변수에 `auto` 타입을 사용할 수 있는데, 이 두가지의 경우에는 완벽히 템플릿 타입 추론의 법칙을 따른다.

```cpp
auto createInitList()
{
    return {1, 2, 3};       // error: can't deduce type
}

auto resetV = [&v](const auto& newValue) { v = newValue; };

resetV({1,2,3});            // error: can't deduce type
```

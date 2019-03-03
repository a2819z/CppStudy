### Array Arguments

```cpp
const char name[] = "J. P. Briggs";

const char* ptrToName = name;

template<typename T>
void f(T param);

f(name);
```

Cpp 에서 배열과 포인터는 서로 다른 타입이지만 배열이 포인터로 붕괴(decay)가 가능하다. 이때 배열이 템플릿 함수에 넘어가게 되면 어떻게 될까?

```cpp
void myFunc(int param[]);
void myFunc(int* parma);
```

위 배열 함수 파라미터는 아래의 포인터 파라미터를 가진 함수와 동일시 취급된다. 이와 같이 배열을 템플릿 함수에 넘길때 타입추론은 포인터 타입으로 된다. 즉 `T` 는 `const char*` 타입으로 추론된다.

```cpp
template<typename T>
void f(T& param);

f(name);
```

위와같은 경우에는 `T` 는 배열타입으로 추론이 된다. 이 타입은 배열의 크기를 포함하고 있으며, `T` 는 `const char [13]` 으로, `param` 은 `const char(&)[13]` 으로 추론이 된다. 이를 이용해 배열의 길이를 추론하는 템플릿 함수를 구현할 수 있다.

```cpp
template<typename T, std::size_t N>
constexpr std::size_t arraySize(T (&)[N]) noexcept
{
    return N;
}
```

### Function Arguments

배열과 같이 함수타입 또한 포인터로 붕괴(decay)가 가능하다.

```cpp
void someFunc(int, double);

template<typename T>
void f1(T param);

template<typename T>
void f2(T& param);

f1(someFunc);       // param: void (*)(int, double)

f2(someFunc);       // param: void (&)(int, double)
```

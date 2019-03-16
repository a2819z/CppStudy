## Item 8: Prefer nullptr to 0 and NULL

`nullptr` 을 다루는 이 장에서 우선 다뤄야 하는것은 0은 int 이지 포인터가 아니라는 것이다. 컴파일러는 0이 오직 포인터로만 해석이 가능할 때 포인터로 사용하는 것이지, 이 외 다른 해석여지가 있으면 int로 다룬다. 이는 `NULL` 또한 같다.

```cpp
void f(int);
void f(bool);
viod f(void*);

f(0);           // calls f(int), not f(void*)

f(NULL);        // never calls f(void*)
```

`f(NULL)` 의 불확실한 동작의 원인은 `NULL` 의 구현과 연관이 있다. 만약 `NULL` 을 `0L` 로 정의한다면, `long` 을 `int`, `bool` 또는 `void*` 의 변환이 모호하기 때문이다. 여기서 중요한 점은 컴파일러와 개발자의 입장에서의 해석(의도)가 다르다는 것이다. null pointer의 의미를 가진 NULL을 넘겼지만 컴파일러는 integer 타입의 함수를 호출한다.

`nullptr` 의 장점이 바로 integer 타입이 아니라는 것이다. 사실 `nullptr` 의 타입( 'std::nullptr_t` )은 포인터가 아니지만 모든 타입의 포인터라 생각해도 된다. 이는 암시적으로 포인터 타입으로 변환이 가능하여, `nullptr` 이 모든 포인터 타입에 작동하는 것의 이유이기도 하다.

```cpp
f(nullptr)      // calls f(void*) overload
```

0과 NULL 대신 `nullptr` 의 사용으로 우리는 C++11 이전에 겪었던 오버로드 문제를 해결할 수 있다. 더불어 `auto` 변수를 사용할 때, 우리는 더욱더 깔끔한 코드를 구현할 수 있다.

```cpp
auto result = findRecord( /* arguments */ );

if (result == 0) {
    ...
}
```

`findRecord` 함수가 무엇을 리턴하는지 모른다면, `result` 가 포인터 타입인지, 정수 타입인지 명확하지 않을 것이다. 조건문의 0은 두 경우 모두를 만족할 수 있기때문이다. 0 대신 `nullptr` 을 사용한다면 우리는 더욱더 명확한 의미를 가진 코드를 작성할 수 있을것이다.

```cpp
auto result = findRecord( /* arguments */ );

if (result == nullptr) {
    ...
}
```

또한 템플릿과 함께 사용할 때도 멋진 코드를 생산해낼 수 있다. 뮤텍스(mutex)가 잠겨(lock)있을 때만 호출할 수 있는 함수가 있다고 가정해보자. 각각의 함수는 다른 종류의 포인터를 가진다.

```cpp
int    f1(std::shared_ptr<Widget> spw);
double f2(std::unique_ptr<Widget> upw);
bool   f3(Widget* pw);

std::mutex f1m, f2m, f3m;

using MuxGuard =
    std::lock_guard<std::mutex>;
...

{
   MuxGuard g(f1m);
   auto result = f1(0);
}

{
   MuxGuard g(f2m);
   auto result = f2(NULL);
}

{
   MuxGuard g(f3m);
   auto result = f3(nullptr);
}
```

위의 반복적인 패턴을 줄이고자 템플릿 함수를 구성하였다.

```cpp
template<typename FuncType, typename MuxType, typename PtrType>
decltype(auto) lockAndCall(FuncType func, MuxType& mutex, PtrType ptr)
{
    MuxGuard g(mutex);
    return func(ptr);
}

auto result1 = lockAndCall(f1, f1m, 0);         // error

auto result2 = lockAndCall(f2, f2m, NULL);      // error

auto result3 = lockAndCall(f3, f3m, nullptr);   // fine
```

위 세개의 함수 호출에서 `PtrType` 의 타입추론에 대해 자세히 분석해보자. 0과 NULL의 경우에는 int 타입으로 추론이 된다. 이런 int 타입 ptr을 포인터 타입을 가지는 함수에 넘기면서 에러가 생기는 것이다.
이와 반대로, `nullptr` 을 넘기면 `ptr` 의 타입은 `std::nullptr_t` 로 추론이 되며, `ptr` 을 함수에 넘기면 `std::nullptr_t` 는 `Widget*` 는 암시적으로 변환이된다.



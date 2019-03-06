# Chapter 2. auto

## Item 5: Prefer auto to explicit type declarations.

이터레이터(iterator)의 역참조(dereference)를 통해 지역변수를 선언하는 예를 들어보자.
```cpp
template<typename It>
void dwim(It b, It e)
{
    while (b != e) {
        typename std::iterator_traits<It>::value_type
            currValue = *b;
        ...
    }
}
```

C++11에서 `auto` 키워드가 등장하며 이러한 수고(`typename std::iterator_traits<It>::value_type`)는 사라젔다. `auto` 변수는 초기화될 때 타입을 추론하므로 반드시 선언과 함께 초기화가 되야한다. 이는 초기화되지 않은 변수 문제에 벗어날 수 있다는 뜻이다.

```cpp
template<typename It>
void dwim(It b, It e)
{
    while( b != e) {
        auto currValue = *b;
        ...
    }
}
```

또한 `auto` 는 타입추론을 사용하므로, 컴파일러만 아는 타입을 변수로 선언할 수 있다.
```cpp
auto derefUPLess =                          // in C++11
    [](const std::unique_ptr<Widget>& p1,
       const std::unique_ptr<Widget>& p2)
    { return *p1 < *p2; };

auto derefLess =                            // in C++14
    [](const auto& p1,
       const auto& p2)
    { return *p1 < *p2; };
```

이때 굳이 auto를 사용하지 않고 `std::function` 을 사용해도 되지않냐는 의문이 생길수 있다. 이 의문을 풀기전 `std::function` 에 대해 먼저 알아보자.

### std::function

`std::function` 은 C++11 에서 도입된 함수 포인터의 일반화된 템플릿이다. 기존의 함수 포인터가 함수만 가르킬 수 있었다면 `std::function` 은 호출가능한 객체(callable object)를 참조할수 있다. 함수 포인터가 가르킬 함수의 타입을 명시하는 것처럼, `std::function` 또한 템플릿 매개변수를 통해 타입을 명시해줘야한다.

```cpp
bool(const std::unique_ptr<Widget>&, const std::unique_ptr<Widget>&);

std::function<bool(const std::unique_ptr<Widget>&,
                   const std::unique_ptr<Widget>&)> func;
```

람다 또한 호출가능한 객체를 만들어내기 때문에 `std::function` 객체에 저장이 가능하다. 이를 `auto` 대신에 `std::function` 으로 표현해보면

```cpp
std::function<bool(const std::unique_ptr<Widget>&,
                   const std::unique_ptr<Widget>&)>
    derefUPLess = [](const std::unique_ptr<Widget>& p1,
                     const std::unique_ptr<Widget>& p2)
                    { return *p1 < *p2; };
```

이는 똑같은 매개변수를 중복적으로 작성하는 것뿐만아니라, `std::function` 과 클로져(closure)는 서로 다른 타입이다. `auto` 는 클로져 자체로 타입이 추론되기때문에 불필요한 메모리 낭비가 없지만, `std::function`은 힙(heap)에 클로져를 할당하면서 불필요한 메모리 사용이 늘어난다. 또한 호출면에서도 `auto` 가 훨씬 빠르다.

또 하나의 장점은 type shortcuts과 관련된 문제를 피할수있다.

```cpp
std::vector<int> v;
...
unsigned sz = v.size();
```

`v.size()` 의 공식적인 반환타입은 `std::vector<int>::size_type` 이다. 이에 몇몇 개발자들은 unsigned를 통해 반환값을 받는다. 이는 32-bit 윈도우에서는 두개의 타입이 같지만, 64-bit 윈도우에선 unsigned와 `std::vector<int>::size_type` 의 크기는 다르다. 이는 32-bit 시스템에서 64-bit 시스템으로 포팅하며 에러가 발생할 수 있으며 이러한 이슈로 시간을 낭비하게 된다.

```cpp
auto sz = v.size(); // sz's type: std::vector<int>::size_type
```

또한 for 구문에서도 `auto` 의 진가가 발휘된다.

```cpp
std::unordered_map<std::string, int> m;
...

for (const std::pair<std::stinrg, int>& p : m)
{
    ...
}
```

위 코드에는 에러가 있다. `std::unordered_map` 의 키(key) 부분은 const 이므로 `std::pair` 의 타입은 `std::pair<std::string, int>` 가 아닌 `std::pair<const std::string, int>` 가 맞다. 이러한 차이를 컴파일러는 매꾸기 위해 임시 객체를 만들어 `p` 가 참조하게 만든다. 이후 루프가 끝나고 난 후에 모든 임시 객체들이 소멸하며 의미없는 반복을 하게된다. 이를 막기위해 컴파일러가 알아서 변수를 추론하게 만든다.

```cpp
for (const auto& p : m)
{
    ...
}
```

위 두가지 예(std::vector<int>::size_type, std::pair)는 명시적 타입이 개발자의 의도치않게 암시적으로 변환이 되는것을 보여주는 예이다. 만약 `auto` 를 사용한다면 이러한 실수로부터 벗어날 수 있으며 훌륭한 가독성 또한 얻을수 있다.

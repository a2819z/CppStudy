## Item 3: Understand decltype

`decltype` 은 주어진 변수명이나 표현식의 타입을 알려준다. 아래는 일반적인 상황의 예이다.

```cpp
const int i = 0;            // decltype(i): const int

bool f(const Widget& w);    // decltype(w): const Widget&
                            // decltype(f): bool(const Widget&)

Widget w;                   // decltype(w): Widget

if (f(w)) ...               // decltype(f(w)): bool
```

`decltype` 은 함수 템플릿의 리턴 타입이 매개변수의 타입에 따라 결정될때 주로 사용된다. () 연산자를 통해 컨테이너(container)의 요소를 반환하는 [] 연산자를 작성한다고 예를 들어보자. 이때 [] 연산자의 반환타입은 () 연산자와 똑같아야 한다.
객체의 타입이 `T` 인 컨테이너의 `operator[]` 는 `T&` 타입을 반환한다. 하지만 `std::vector<bool>` 과 같은 경우에는 `bool&` 이 아닌 새로운 객체를 반환한다. 이때 이렇게 다양한 컨테이너들의 반환타입에 맞게 함수를 작성하는데 `decltype` 이 유용하게 사용이 된다.

```cpp
template<typename Container, typename Index>
auto authAndAccess(Container& c, Index i)
    -> decltype(c[i])                       // c++11
{
    authenticateUser();
    return c[i];
}
```

C++11 에선 함수의 반환 타입에 단순히 `auto` 만을 적는다고 타입추론이 되진않는다. 이를 위해 C++11 에선 trailing return type(->) 문법을 사용한다. 이는 함수의 매개변수를 통해 리턴타입을 추론할 수 있는 이점이 있다. 위 예에서는 함수의 매개변수인 `c` 와 `i` 를 이용해 반환타입을 추론하고 있다. 이를 통해, 위 함수는 어떤 컨테이너를 인자로 받아도 우리가 원하는 반환타입을 반환할 수 있다.
C++11에서 람다의 리턴 타입이 추론되는게 가능했다면, C++14에선 이 기능을 함수에까지 확장했다. 즉 C++14부터는 trailing return type을 생략하고 함수를 작성할 수 있게 되었다.

```cpp
template<typename Container, typename Index>
auto authAndAccess(Container& c, Index i)
{
    authenicateUser();
    return c[i];
}
```

이때 함수의 반환타입은 앞선 Item 1, 2 에서 설명했듯이 cv-qualifer와 &를 다 무시한다. 이는 `T&` 타입을 반환할 시 `T` 가 반환된다는 말이다.

```cpp
std::deque<int> d;
...
authAndAccess(d, 5) = 10;       // authAndAccess return d[5], but its not int&, but int.
```

위 코드에서 `authAndAccess` 는 `int&` 가 아닌 `int` , rvalue를 반환하며 컴파일 에러가 나타난다. 이를 해결하기 위해 우리는 `decltype` 을 리턴타입에 사용한다.

```cpp
template<typename Container, typename Index>
decltype(auto)
authAndAccess(Container& c, Index i)
{
    authenticateUser();
    return c[i];
}
```

`decltype(auto)` 를 사용하게 되며 함수의 반환타입은 우리가 원하는 방식되로 추론되기 시작했다. 이러한 문법은 함수 반환타입 뿐만 아니라 초기화와 함께하는 변수 선언에 사용될 수 있다.

위 `authAndAccess` 함수에는 고칠것이 있는데 그 중 하나가 rvalue를 함수 인자로 받을수 없다는 것이다. rvalue인 컨테이너를 받을 일이 흔한 일은 아니지만 한번 알아보자. 사용자가 단순하게 임시 컨테이너의 한 요소의 복사본을 원한다고 예를 들어보자.

```cpp
std::deque<std::string> makeStringDeque();

auto s = authAndAccess(makeStringDeque(), 5);
```

위 처럼 lvalue와 rvalue를 받는 각각의 오버로딩을 구현할 수 있겠지만, universal(forward) reference를 사용해 보자.
아래 템플릿에서 우리는 어떤 컨테이너를 넘길지 모르고 이는 어떠한 타입의 인덱싱 객체가 나올지 모른다는 이야기다. 미정의 타입을 가진 객체를 pass-by-value로 넘긴다는 것은 불필요한 복사(unnecessary copying), 객체 슬라이싱(object slicing)에 문제가 있을 수 있다. 하지만 표준 라이브러리(standard library)를 따르는게 맞다는 생각에, pass-by-value를 고수한다.

```cpp
template<typename Container, typename Index>
decltype(auto) authAndAcess(Container&& c, Index i)
{
   authenitcateUser();
   return std::forward<Container>(c)[i];    // Item 25
}
```

두번째 문제는 매우 큰 라이브러리를 구현할때의 `decltype` 의 예외발생이다. 이는 `decltype` 의 완벽한 이해가 필요하며 특수한 몇가지 케이스에 익숙해야한다. 우리는 그 중 하나에 대해서만 알아보겠다.

`decltype` 을 변수 이름에 적용하면 변수가 선언된 타입으로 타입을 나타낸다. lvalue expression 은 변수이름 보다 더 복잡한데 `decltype` 은 lvalue expression 에 대해 항상 lvalue reference를 반환한다. lvalue expression 에 타입 `T` 를 가지고 있다면 `decltype` 은 `T&` 타입을 반환한다. 이때 변수 이름에 괄호() 로 감싸면 이는 단순한 이름이 아닌 표현식(expression) 이 되며 `decltype((x))` 는 레퍼런스(reference) 타입을 반환한다.

```cpp
decltype(auto) f1()
{
    int x = 0;
    ...
    return x;           // return int
}

decltype(auto) f2()
{
    int x = 0;
    ...
    return (x);         // return int&
}
```

이때 `f2` 함수가 `f1` 함수의 리턴타입과 다를뿐만 아니라, 지역변수(local variable)을 레퍼런스 타입으로 반환하고 있다는 것에 주목하라. 이는 미정의 동작(undefined behavior)에 빠질 수 있다.

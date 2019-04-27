## Item 15: Use constexpr whenever possible.

C++11 에서 도입된 `constexpr` 은 변수(variable)과 함수(function)에 적용할 수 있다. 이는 변수와 함수에 적용될때 각각 조금 다른 의미를 가지게 되며 이는 혼란을 일으킨다.

변수에 적용된 `constexpr` 은 컴파일타임에 변수가 정해지며 상수성을 가진다(사실 컴파일과 링킹타임이 아닌 번역(translation)에 변수의 값이 정해진다.) 그에 반해 함수에 적용된 `constexpr` 은 그저 컴파일타임에 계산이 되고 상수성을 가진다고 보장만을 해줄뿐이다.

변수가 컴파일타임에 정해지는 것은 정수형 상수값이 필요할 때 이용이된다. 이런 값은 배열의 크기, 템플릿 인자, enum의 값 등 많은곳에서 사용이 가능하다.

```cpp
constexpr auto arraySize1 = 10;

std::array<int, arraySize1> data1;      // fine
```

`constexpr` 함수의 경우 반환값이 컴파일타임 상수인지 아닌지는 함수 호출에 사용되는 인자에 따라 달라진다. 호출에 사용되는 인자가 컴파일타임 상수일 경우 반환값 또한 컴파일타임 상수이며, 런타임 변수로 호출을 하면 반환값 또한 런타임에서 계산이 된다.

실험의 결과를 자료구조에 저장해두는 코드를 작성한다고 생각해보자. 실험에는 `n` 개의 환경(변수)가 있으며 이 환경들은 각 3개의 상태(state)를 가진다. 이럴경우 실험결과는 총 <a href="https://www.codecogs.com/eqnedit.php?latex=3^n" target="_blank"><img src="https://latex.codecogs.com/gif.latex?3^n" title="3^n" /></a> 가지의 결과가 나온다. 이때 `n` 을 컴파일동안 안다면 우리는 `std::array` 가 실험결과값을 저장하기에 적절하다. 이때 배열의 길이를 정할때 `std::pow` 의 사용이 불가능하다. 첫째로 `std::pow` 는 `constexpr` 함수가 아니며, 두번째론 정수형이 아닌 부동소수점형(float)을 반환한다.

이에 우리는 `constexpr` pow 함수를 새로 구현을 해야한다. 이때 `constexpr` 함수는 컴파일타임 상수를 반환할때 모든 연산이 컴파일타임동안 실행되기때문에 몇가지 제약이 따른다.

- C++11: 오직 return 구문하나로만 구현을 해야한다. 이때 ?:(삼항연산자)와 재귀를 이용할 수 있다.

```cpp
constexpr int pow(int base, int exp) noexcept
{
    return (exp == 0 ? 1 : base * pow(base, exp - 1));
}
```

- C++14: C++11에 비해 제약이 비교적 약해젔다.

```cpp
constexpr int pow(int base, int exp) noexcept
{
    auto result = 1;
    for (int i = 0; i < exp; ++i) result *= base;

    return result;
}
```

함수구현에 사용되는 변수를 리터럴 타입으로만 제한이된다. C++11의 경우 void를 제외한 모든 빌트인 타입과, 사용자 정의 리터럴 타입(생성자와 멤버함수가 constexpr인)의 경우를 말한다.

```cpp
class Point {
public:
    constexpr Point() noexcept;
    ...

    constexpr double xValue() const noexcept;
    ...
};

constexpr Point p1(9.4, 27.7);
```

컴파일타임 상수로 위 `Point` 클래스를 생성한다면, 이 클래스 또한 `constexpr` 로 선언할 수도있다. `xValue()` 와같은 멤버함수 또한 생성된 객체가 `constexpr` 이라면 `constexpr` 로 호출이 된다. 이런 멤버변수는 `constexpr` 함수 바디내에서도 호출이 가능하다.

```cpp
constexpr
Point midpoint(const Point& p1, const Point& p2) noexcept
{
    return { (p1.xValue() + p2.xValue()) /2,
             (p1.yValue() + p2.yValue()) /2 };
}

constepxr auto mid = midpoint(p1, p2);
```

객체 `mid` 의 초기화에 생성자, 게터(getter), 비멤버 함수의 호출이 있었음에도 불구하고, 이 객체는 읽기전용 메모리에 생성이된다. 이는 `mid.xValue() * 10` 과 같은 표현식이 템플릿 인자나 enum의 값으로 사용될 수 있다는 말이다. 이는 컴파일타임과 런타임이 명확히 구분되던 예전과 다르게 둘사이의 경계가 모호해지며, 많은 코드들이 컴파일타임에 계산이 되며 더욱더 빠른 소프트웨어를 만들수 있게되었다.(물론 컴파일 시간이 더 오래걸리긴 한다.)

C++11 에선 `Point` 의 멤버함수로 세터가 `constexpr` 로 선언이 되는걸 제한한다. 이는 객체의 내부를 변경시키며(`constexpr` 함수는 암시적으로 `const` 이다), `void` 리턴 타입을 가져 C++11에선 이를 금지시킨다. 이 두개의 제한은 C++14에서 약화된다.

```cpp
class Point {
public:
    ...
    constexpr void setX(double newX) noexcept { x = newX; }
    ...
};

constexpr Point reflection(const Point& p) noexcept
{
    Point result;

    result.setX(-p.xValue());
    result.setY(-p.yValue());

    return result;
}

constexpr Point p1(9.4, 27.7);
constexpr Point p2(28.8, 5.3);
constexpr auto mid = midpoint(p1, p2);

constexpr auto reflectedMid = reflection(mid);
```

`constexpr` 객체와 함수는 비 `constepxr` 객체와 함수에 비해 더 많은 범위에서 사용이 가능하며, 이는 `constexpr` 을 가능한 채택하는것이 좋다는것을 의미한다.

또한 `constexpr` 은 객체나 함수의 인터페이스의 일부이다. 이는 사용자(client)에게 "상수 표현식이 필요한 상황에서 사용이 가능하다." 라는것을 말해준다.

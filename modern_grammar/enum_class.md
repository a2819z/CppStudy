## Item 10: Prefer scoped enum

C++은 기본적으로 {} 안에서 선언된 이름들은 해당 범위(scope) 내에서만 보인다(visiblity). 하지만 C++98 스타일의 enum은 이 규칙을 만족하지 않는다.

```cpp
enum Color { black, white, red };

auto white = false;                     // error
```

위 예제에 `black, white, red` 는 Color 범위에서 선언되었지만, 범위밖에서도 볼 수 있어 `white` 가 이미 선언된 변수라 인식을 한다. 이러한 열거자(enumerator)를 우린 `unscoped enum` 이라 한다. 반대로 C++11 에서는 `scoped enum` 이 등장하였다.

```cpp
enum class Color { black, white, red };

auto white = false;                     // fine

Color c = white;                        // error
                                        // type of white is bool

Color c = Coloro::white;                // fine

auto c = Color::white;                  // fine
```
이름공간(namespace) 오염을 줄여주는 이유로만도 `scoped enum` 을 채택할 이유는 충분하지만, 그 외 몇가지 더 장점이 있다. `unscoped enum` 은 암시적으로 정수형으로 바뀌는 반면에, `scoped enum` 은 그렇지 않다는 점에 비교적 강타입(strongly typed)이다.

```cpp
enum Color { black, white, red };

std::vector<std::size_t>
    primeFactors(std::size_t x);

Color c = red;
...

if ( c < 14.5 ) {                   // compare Color to double
    auto factors =
        primeFactors(c);            // compute prime facotrs of a Color
}
```

위의 경우 문법적으로 올바르지 않은 경우라도 암시적인 변환때문에 허용이된다. 이를 막기위해 우리는 `scoped enum` 을 사용하면 된다.

```cpp
enum class Color { black, white, red };

std::vector<std::size_t>
    primeFactors(std::size_t x);

Color c = Color::red;
...

if ( c < 14.5 ) {                   // error!
    auto factors =
        primeFactors(c);            // error!
}
```

만약 `Color` 타입을 다른 타입으로 변환을 원한다면 `static_cast<>` 을 사용하면 된다.

```cpp
if ( static_cast<double>(c) < 14.5 ) {                   // error!
    auto factors =
        primeFactors(static_cast<double>(c));            // error!
}
```

또한 `scoped enum` 은 전방선언(forward declaration) 이 가능하다.

```cpp
enum Color;         // error!

enum class Color;   // error!
```

사실 C++11 에서 조금의 추가작업을 한다면 `unscoped enum` 또한 전방선언이 가능하다. 이는 컴파일러가 모든 `enum` 의 underlying type을 정수형으로 결정하는 것에 기안한것이다.

```cpp
enum Color { black, white, red };
```

위의 enum은 세개의 변수만 있기때문에 enum의 underlying type은 char로 결정될 것이다. 하지만 범위가 더 넓은 `enum` 의 경우

```enum Status { good = 0,
                 failed = 1,
                 incomplete = 100,
                 corrupt = 200,
                 indeterminate = 0xFFFFFFFF
                };
```

위 `enum` 은 0부터 0xFFFFFFFF 의 범위를 가젔다. 특별한 경우를 제외하곤 컴파일러는 char 타입보다 더큰 정수형 타입을 채택할 것이다. 메모리 사용의 효율성을 위해 컴파일러는 `enum` 의 값을 표현할 수 있는 최소한의 타입을 선택한다. 또는 사이즈 보다 속도를 최적화하기를 위해 최소한의 값을 선택하지 않을 수 있다. 하지만 컴파일러는 주로 크기를 최적화하기를 원한다. 이 사이즈 최적화를 위해 C++98에서 `enum` 이 오직 정의되어야만 한다.

전방선언의 불가로 컴파일 의존성(compilation dependencies)의 증가가 생긴다.

```cpp
enum Status { good = 0,
              failed = 1,
              incomplete = 100,
              corrupt = 200,
              indeterminate = 0xFFFFFFFF
            };
```

위 `enum` 은 시스템 전체적으로 사용이되 시스템의 모든 부분에 헤더 파일로 인클루드(include)를 했다고 가정하자. 만약 이 `enum` 에 새로운 값이 입력된다고 생각해보자.


```cpp
enum Status { good = 0,
              failed = 1,
              incomplete = 100,
              corrupt = 200,
              audited = 500,
              indeterminate = 0xFFFFFFFF
            };
```

새로운 열거자의 변화로 모든 시스템은 새롭게 컴파일될 것이다. 많은 개발자들은 이를 피하고 싶을 것이고, C++11의 전방선언 열거자는 이러한 고민을 날려버린다.

```cpp
enum class Status;

void continueProcessing(Status s);
```

하지만 컴파일러가 `enum` 의 사이즈를 알아야 한다면 C++11의 `enum` 은 어떻게 전방선언으로 가능하게 할 수 있을까? 바로 underlying type을 명시적으로 선언해주는 것이다.

```cpp
enum class Status: std::uint32_t;   // underlying type for
                                    // Status is std::uint32_t

enum Color: std::uint8_t;

enum class Status: std::uint32_t { good = 0,
                                   failed = 1,
                                   incomplete = 100,
                                   corrupt = 200,
                                   audited = 500,
                                   indeterminate = 0xFFFFFFFF
                                 };
```

위의 장점에 불구하고 `unscoped enum` 은 C++11의 `std::tuple` 에 사용이 된다. 사용자의 이름, 이메일, 평판을 가지는 튜플을 만든다고 가정해보자.

```cpp
using UserInfo =
    std::tuple<std::string, std::string, std::size_t>;

UserInfo uInfo;
...

auto val = std::get<1>(uInfo);
```

개발자의 입장에서 field 1이 사용자의 이메일과 항상 기억할 순 없을것이다. 이를 `unscoped enum` 을 사용하면 쉽게 해결할 수 있다.

```cpp
using UserInfo =
    std::tuple<std::string, std::string, std::size_t>;

enum UserInfoFields { uiName, uiEmail, uiReputation };

UserInfo uInfo;
...

auto val = std::get<uiEmail>(uInfo);
```

이를 `scoped enum` 으로 바꾸게되면,

```cpp
using UserInfo =
    std::tuple<std::string, std::string, std::size_t>;

enum class UserInfoFields { uiName, uiEmail, uiReputation };

UserInfo uInfo;
...

auto val 
    = std::get<static_cast<std::size_t>(UserInfoFields::uiEmial)>
      (uInfo);
```

이런 장황한 길이의 문장을 열거자를 받고 std::size_t 를 반환하는 함수를 작성하면 길이를 줄일수 있지만, 꾀 복잡하다.

- `std::get` 은 템플릿이며 템플릿 인자가 필요하다 -> 컴파일타임 내 템플릿으로 넘어오는 인자의 타입이 정해저야 한다. -> `constexpr`

- 반환타입의 일반화가 필요함 -> `std::underlying_type`

- 절대로 예외를 생성할 일이 없음 -> `noexcept`

```cpp
template<typename E>
constexpr auto
toUType(E enumerator) noexcept
{
    return
        static_cast<typename
                    std::underlying_type<E>::type>(enumerator);
}

auto val = std::get<toUType(UserInfoFields::uiEmail)>(uInfo);
```

하지만 여전히 `unscoped enum` 보다 더 장황하다. 위와 같은 소수의 경우에는 이름공간 오염을 감수하고도 `unscoped enum` 을 채택할 필요가 있기도 하다.

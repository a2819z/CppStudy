## Item 7: () and {}

### Uniform Initialization

```cpp
int x(0);           // initializer is in parentheses

int y = 0;          // initializer follows "="

int z{ 0 };         // initializer is in braces

int z = { 0 };      // same as initializer in braces
```

C++ 입문자들은 `=` 연산자를 초기화에 사용하게 되면 할당(assignment) 또한 같이 일어난다고 착각한다. 내장된 타입(built-in type) 같은 경우는 실용적인 면에선 별차이가 없지만, 사용자 타입(user-defined type)의 경우 초기화와 할당을 구분해주는 것은 매우 중요하다.

```cpp
Widget w1;          // call default constructor

Widget w2 = w1;     // not an assignmnet, copy constructor

w1 = w2;            // an assignmnet, copy operator=
```

이런 여러가지의 초기화 문법의 혼동을 막기위해 C++11 에서는 __uniform initialization__ 문법을 도입한다.

```cpp
std::vector<int> v{ 1, 3, 5 };
```

중괄호는 또한 비정적(non-static) 데이터 멤버를 초기화할때 사용할 수 있다. 이때 중괄호 이외에도 "=" 문법을 허용한다(괄호는 제외).

```cpp
class Widget {
  ...
    
private:
  int x{ 0 };   // fine
  int y = 0;    // also fine
  int z(0);     // error
};
```

반대로 복사불가능(uncopyable)한 객체는 중괄호와 소괄호를 통해 초기화가 가능하다(= 불가능).

이렇기 때문에 중괄호를 통한 초기화가 왜 uniform 이라 불리는지 알 수 있다.

중괄호 초기화의 새로운 특징으로 내장타입의 암시적 축소 변환(implicit narrowing conversion)을 금지한다.

```cpp
double x, y, z;
...
int sum1{ x + y + z };  // error

int sum2(x + y + z);

int sum3 = x +y + z;
```

또 다른 특징은 C++의 제일 짜증 나는 분석(vexing parse)에 대한 면역성이다. C++의 모든 선언은 분석이 되는데, 개발자를 제일 짜증나게 하는 분석은 객체를 기본 생성자(default-construct)로 생성하고 싶은데, 함수 선언으로 인식하는 경우이다. 이를 해결하기 위해 우린 중괄호를 이용해 초기화하면 된다.

```cpp
Widget w1(10);  // call Widget ctor with argument 10

Widget w2();    // declare a function

Widget w3{};    // calls Widget ctor with no args
```

### uniform initialization with ctor with std::initializer_list

중괄호 초기화에 마냥 좋은점만 있는건 아니다. 생성자에 `std::initializer_list` 를 매개변수로 가지는 오버로드가 존재할때 우린 예상치 못한 결과를 맞이한다. 

생성자 호출에서, `std::initializer_list` 오버로드가 존재하지 않는다면 소괄호와 중괄호는 똑같은 의미를 가진다.

```cpp
class Widget {
public:
  Widget(int i, bool b);
  Widget(int i, double d);
  ...
};

Widget w1(10, true);

Widget w2{10, true};

Widget w3(10, 5.0);

Widget w4{10, 5.0};
```

만약 `std::initializer_list` 가 생성자 매개변수로 존재한다면, 중괄호 초기화 문법은 `std::initializer_list` 오버로드를 선택하게 될 것이다.

```cpp
class Widget {
public:
  Widget(int i, bool b);
  Widget(int i, double d);
  Widget(std::initializer_list<long double> il);    // added
  ...
};

Widget w1(10, true);

Widget w2{10, true};    // now calls std::initialzier_list ctor

Widget w3(10, 5.0);

Widget w4{10, 5.0};     // now calls std::initialzier_list ctor
```

또한 복사/이동 생성자를 호출하려던 의도 또한 `std::initializer_list` 생성자로 인해 무산이 될 수 있다.

```cpp
class Widget {
public:
  Widget(int i, bool b);
  Widget(int i, double d);
  Widget(std::initializer_list<long double> il);

  operator float() const;
  ...
};

Widget w5(w4);              // calls copy ctor

Widget w6{w4};              // calls std::initializer_list ctor
                            // w4 converts to float, and float
                            // converts to long double

Widget w7(std::move(w4));   // calls move ctor

Widget w8{std::move(w4)};   // calls std::initializer_list ctor
                            // for same as w6
```

```cpp
class Widget {
public:
  Widget(int i, bool b);
  Widget(int i, double d);
  
  Widget(std::initializer_list<bool> il);
  
  ...
};

Widget w{10, 5.0};  // error, narrowing conversion
```

앞의 두개의 생성자는 무시하고 컴파일러는 `std::initialzier_list` 를 매개변수로 갖는 생성자를 선택하였다. 이 생성자를 호출할려면 10과 5.0을 `bool` 타입으로 변환해야지만, 축소 변환의 금지로 에러가 난다.

위 예제들에서 봤듯이 컴파일러는 중괄호 초기화를 `std::initializer_lsts` 를 가진 생성자와 매칭을 거의 강제하다 싶이 한다.

위와 같은 상황에서 중괄호 초기화가 우리가 원한데로 작동할려면 `std::initializer_list` 의 템플릿 인자의 타입이 암시적 변환을 허용하지 않는 것이다. 예를들어 `std::string` 을 템플릿 인자로 사용한다면 `int` 와 `bool` 을 암시적으로 변환할 수 없으므로 컴파일러는 다른 생성자를 선택한다.

마지막으로 빈 중괄호 초기화를 사용하면 컴파일러는 기본 생성자와 `std::initializer_list` 를 매개변수로 가진 생성자 둘중 하나를 선택하게 될까? 답은 기본 생성자이다.

만약 빈 `std::initializer_list` 를 이용해 생성자를 호출하고 싶다면 아래와 같이 하면 된다.

```cpp
Widget w4({});

Widget w5{{}};
```

`std::initializer_list` 생성자가 없는 클래스가 있다. 여기에 새로운 오버로드를 추가했다고 해보자. 중괄호 초기화를 하면 컴파일러는 오버로드된 생성자들 중 하나를 선택할 것이다. 추가로 계속 다른 오버로드를 추가한다면 컴파일러는 새로운 오버로드들 중 가장 적합한 것은 선택할 것이다. `std::initializer_list` 오버로드와 다른 점은 이 오버로드는 다른 오버로드들을 다 감춰버린다. 이와같은 성격은 초기화 리스트를 오버로드에 추가하기에 꾀 많은 노력이 들어가게 한다.

두번째 교훈은 사용자가 소괄호와 중괄호의 선택에 신중해야한다는 것이다. 대부분의 개발자들은 하나의 방식을 기본으로 사용하고, 다른 하나는 꼭 필요할때만 사용한다. 중괄호를 기본으로 사용하는 것은 막강한 활용성, 축소 변환의 금지 그리고 짜증나는 파싱(vexing parse)를 막아준다. 반면에 소괄호를 기본으로 사용하는 것은 C++98 문법의 일관성 유지, `auto` 타입의 타입추론 오류를 그리고 초기화 리스트 생성자의 무차별한 호출을 막을 수 있다.

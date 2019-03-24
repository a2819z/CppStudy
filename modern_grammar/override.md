## Item 12: override

C++에서의 OOP(object oriented programming)은 클래스, 상속, 가상함수으로 이루어진다. 가장 중요한 개념중 하나는 부모클래스(base class)의 가상함수의 구현부를 재구현하여 오버라이드(override)하는 것이다.

이때 오버라이딩은 오버로딩과 햇갈려해 오류를 저지르는 경우가 많다. 이때 오버라이딩을 할 때 몇몇가지 지켜야할 조건이 있다.

- 부모 클래스의 함수는 무조건 virtual

- 상속된 함수의 이름은 같아야함.

- 매개변수의 타입은 같아야함.

- 상수성(constness) 또한 같아야함.

- 반환타입과 예외 지정자가 같아야함.

- 레퍼런스 제한자 또한 같아야함.

위 조건들을 지키지 못한다면 조금의 실수로 큰 차이점을 불러일으킨다. 오버라이딩 에러를 가진 코드는 작동할테지만, 우리가 원했던 방식으로 작동하는건 아닐것이다. 

```cpp
class Base {
public:
    virtual void mf1() const;
    virtual void mf2(int x);
    virtual void mf3() &;
    void mf4() const;
};

class Derived: public Base {
public:
    virtual void mf1();                 // const
    virtual void mf2(unsinged int x);   // parameter type
    virtual void mf3() &&;              // ref-qualifer
    void mf4() const;                   // non-virtual
};
```

위와 같은 경우 컴파일러가 경고를 줄수도 있고 안줄수도 있다. 이를위해 C++11에서는 자식클래스의 함수가 부모클래스의 오버라이드라는걸 명시먹으로 나타내는 키워드를 도입했다. `override`

```cpp
class Base {
public:
    virtual void mf1() const;
    virtual void mf2(int x);
    virtual void mf3() &;
    void mf4() const;
};

class Derived: public Base {
public:
    virtual void mf1() override;                 // error
    virtual void mf2(unsinged int x) override;   // error
    virtual void mf3() && override;              // error
    void mf4() const override;                   // error
};
```

이렇게 `overriding` 키워드를 명시해주면 컴파일러는 오버라이딩과 관련된 모든 문제를 검사한다. 이는 또한 부모클래스의 함수 시그네쳐가 변경되었을때, 자동으로 감지하여 자식클래스의 함수 시그네처 또한 변경해야함을 알려준다.

### 참고: ref-qualifier

각각 lvalue와 rvalue만을 받는 함수를 작성한다고 가정해보자.

```cpp
void doSomething(Widget& w);

void doSomething(Widget&& w);
```

멤버함수의 레퍼런스 제한자 또한 객체가 어떤 멤버함수를 호출할지 정하는데 도움을 준다. 이가 필요한 경우를 알아보자.

```cpp
class Widget {
public:
    using DataType = std::vector<double>;
    ...
    
    DataType& data() { return values; }
    ...
    
private:
    DataType values;
};

Widget w;

auto vals1 = w.data();
```

`Widget::data` 의 리턴타입은 `std::vector<double>&` 이며, lvalue refer는 lvalue로 부터 정의되어 우리는 `vals1` 는 복사생성자를 호출해 초기화를 한다.

이제 `Widget` 을 생성하는 팩토리 함수를 가지고 있다고 가정해보자.

```cpp
Widget makeWidget();

auto vals2 = makeWidget().data();
```

`makeWidget()` 은 임시객체를 반환하고, 임시객체의 data를 우리는 `vals2` 에 넘겨 복사생성자를 호출한다. 이때 임시객체의 데이터를 복사하며 시간낭비가 생긴다. 이 경우, move가 이상적일 것이다.

우리는 레퍼런스 한정자를 사용해 rvalue `Widget` 으로 부터 반환되는 `data` 가 rvalue라는 것을 명시적으로 나타내줄수 있다. 즉 복사생성자가 아닌 이동생성자를 호출할 수 있다.

```cpp
class Widget {
public:
    using DataType = std::vector<double>;
    ...
    
    DataType& data() & { return values; }

    DataType data() &&
    { return std::move(values); }
    ...
    
private:
    DataType values;
};

Widget w;

auto vals1 = w.data();

auto vals2 = makeWidget().data();
```

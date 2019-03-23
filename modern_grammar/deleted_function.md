## Item 11: Prefer deleted functions to private undefined ones.

라이브러리를 설계할때 사용자들이 몇몇 함수들을 호출하는 것을 막고 싶을때 그 함수를 선언하지 않을 수 있다. 하지만 몇몇 함수들은 컴파일러가 자동으로 생성해준다.

C++98 은 이러한 함수들을 private으로 선언하는 방식을 사용한다. C++의 표준 라이브러리에서 iostream의 베이스는 basic_ios 이다. istream과 ostream의 복사는 여러가지 이유로 올바르지 않다. istream과 ostream이 복사불가능하게 만들기 위해, basic_ios 는 C++98에서 이렇게 구현되있다.

```cpp
template<class charT, class traits = char_traits<charT>>
class basic_ios : public ios_base {
public:
...

pirvate:
    basic_ios(const basic_ios&);                // not defined
    basic_ios& operator=(const basic_ios&);     // not defined
};
```

정의를 하지 않은것은 멤버함수나 friend 클래스를 통한 해당 함수를 호출할 수 없게 만들기 위함이다.

C++11에선 `= delete` 키워드가 추가되었다.

```cpp
template<class charT, class traits = char_traits<charT>>
class basic_ios : public ios_base {
public:
    ...
    basic_ios(const basic_ios&) = delete;
    basic_ios& operator=(const basic_ios&) = delete;
};
```

위 두가지 방식은 겉으로 보기에는 방식의 차이라고 생각할 수 있지만, 생각보다 큰 차이점이 있다. 삭제된 함수(deleted function)은 어떤 방식으로도 호출이 되지 않으므로, 멤버 함수(or 프렌드 함수)로 해당 함수를 호출하는 것은 컴파일 에러를 일으킨다. 하지만 C++98은 링킹타임(link-time)까지 해당 오류를 알아차리지 못한다.

삭제된 함수는 public에 선언되는것이 규칙이다. 클라이언트가 함수를 호출할 때, C++은 삭제여부 확인 전 접근성(accessibility)를 확인한다. 클라이언트가 삭제된 private 함수를 호출한다면 컴파이러는 오직 private 함수라고만 말해줄수도 있다. 이는 에러메시지 추적에 어려움을 갖게 한다.

private은 오직 멤버함수에만 적용이 가능하지만, delete는 어떤 함수에나 가능하다는 것이다. int 타입을 받아 bool 타입을 리턴하는 비멤버 함수를 가정해보자.

C++이 C로부터 받은 세습은 수처럼 보이는 타입들은 암시적으로 int 타입으로 변환이 가능하다는 것이다.

```
bool isLucky(int number);

if (isLucky('a'))...

if (isLucky(true))...

if (isLucky(3.5))...
```

만약 무조건 정수형으로만 함수를 호출해야 한다면, 우리는 정수형을 제외한 다른 인자를 가지는 함수를 삭제하면 된다.

```cpp
bool isLucky(int number);

bool isLucky(char) = delete;

bool isLucky(bool) = delete;

bool isLucky(double) = delete;
```

삭제된 함수는 사용될순 없지만, 그들은 프로그램의 일부이다. 이는 오버로드 결정에서도 참여하게 된다는 뜻이다.

또한 delete는 템플릿이 특정 타입으로 인스턴스화(instantiation)가 되는것을 막을때 사용한다.

```cpp
template<typename T>
void processPointer(T* ptr);
```

포인터에서 두가지의 특별한 케이스가 있다.

- `void*` : 역참조와 증감연산이 불가능하다.

- `char*` : C-스타일 문자열

위 두가지의 특별한 케이스의 호출은 특별한 조치가 필요하며, 이 두가지 타입으로 호출하는 것을 금지하는게 조치라고 가정해보자.

```cpp
template<>
void processPointer<void>(void*) = delete;

template<>
void processPointer<char>(char*) = delete;
```

클래스 내 함수 템플릿이 있다고 가정해보자. 몇몇 인스턴스를 private에 선언해 해당 함수를 비활성화하겠다고 가정하면, 이것은 불가능하다. 이는 다른 서로 다른 접근 레벨에 특수화를 할 수 없기 때문이다.

```cpp
class Widget{
public:
    ...
    template<typename T>
    void processPointer(T* ptr)
    {...}

private:
    template<>                      // error!
    void processPointer<void>(void*);
};

class Widget{
public:
    ...
    template<typename T>
    void processPointer(T* ptr)
    {...}
};

template<>
void Widget::processPointer<void>(void*) = delete;
```

C++98 에서 사용하던 기법은 C++11의 `delete` 로 완벽하게 구현이 가능하며, `delete` 가 훨씬 훌륭하다.

## Item 14: Declare functions noexcpet if they won't emit exceptions.

C++98의 예외한정자는 까다로운점이 많다. 함수 구현이 바뀐다면 예외한정자 또한 같이 바꿔줘야 하는 경우가 많다. 이에 따라 예외한정자에 의존하는 클라이언트 코드에서 오류가 생길 수 있다.

C++11으로 넘어오며 어떤 종류의 에러가 생기는지 나타내기보다는 에러의 발생유무만을 나타내자는 의견들이 나오며 이 방식을 채택했다.(여전히 C++98 방식의 예외한정자는 유효하지만 권장되진 않는다.) `noexcept` 는 함수가 예외를 발생시키지 않는다고 보장하는 한정자이다.

`noexcept` 의 유무는 인터페이스 디자인에 매우 중요하다. 호출자의 입장에선 `noexcept` 함수는 예외안전성과 호출용이성에 영향을 끼친다. 이와 같이 noexcept이냐 아니냐는 멤버 함수가 const 이냐 아니냐라는 결정하는데 하나의 정보가 된다.

또한 이는 컴파일러가 더욱 좋은 오브젝트 코드(object code)를 생성한다. 이를 이해하기 위해선 C++98 / C++11의 예외한정자 차이를 알아야 한다.

```cpp
int f(int x) throw();       // C++98 style

int f(int x) noexcept;      // C++11 style
```

위 코드에서 런타임(runtime)에 예외가 발생한다고 가정해보자. C++98의 경우, 콜스택이 풀리며(unwind) 프로그램이 종료된다. C++11의 경우, 프로그램이 종료되기전 _아마(possibly)_ 스택이 풀릴것이다. 미묘한 두 차이는 코드 생성에 매우 큰 영향을 끼친다. `noexcept` 함수의 경우, 필히 스택을 추적해 풀필요가 없기때문에 최적화의 입장에선 융통성이 발휘된다. 하지만 `throw()` 의 경우에는 필히 스택을 추적해 풀어야 하므로 최적화의 제한이 따른다.

```cpp
RetType function(params) noexcept;      // most optimizable

RetType function(params) throw();       // less optimizable

RetType function(params)                // less optimizable
```

몇몇 케이스에는 더욱더 효과를 발휘하는데, 그중 하나가 move 연산자이다.

```
// C++98
std::vector<Widget> vw;
...
Widget w;
...
vw.push_back(w);
...
```

`std::vector` 가 `push_back` 으로 새로운 요소를 추가할때 `std::vector` 의 용량이 부족하면 더 큰 공간의 메모리를 잡아주며 이전의 데이터를 옮긴다. 이때 C++98은 각각 요소의 복사로 전달을 완료하며, 이전의 메모리를 해제한다. 이런 메커니즘은 `push_back` 이 매우강한 예외안전성을 보장해준다.(모든 요소가 새로운 메모리공간에 복사되어야만 이전 메모리가 해제됨을 이용.)

C++11의 경우 컴파일러가 자연스럽게 `std::vector` 의 요소 복사를 이동(move)로 바꿔준다. 이는 `push_back` 의 예외안전성의 보장에 악영향을 끼친다. n번째까지의 데이터들이 이동에 성공하고 n+1번째 이동에 예외가 발생한다면, `push_back` 연산은 완료되지 못하며 기존의 `std::vector` 의 데이터는 변경된다.(n번째까지의 데이터들이 이동됨)

이는 `push_back` 의 강력한 예외안전성 보장에 따른 레거시 코드의 동작에 매우 큰 문제가 된다. 그러므로 C++11에선 `push_back` 의 내부구현을 함부로 copy에서 move로 바꿀수 없다. 이러한 이유로 `std::vector::push_back` 은 가능하면 이동을 하되, 왠만하면 복사를 사용해라 라는 내부구현 전략을 취한다. 그러면 언제 이동이 가능하는지를 판단할 수 있을까? 그건 바로 `noexcept` 의 선언여부를 확인해보는 것이다.

`swap` 함수 또한 위와같이 `noexcept` 가 매우 중요한 케이스이다. 표준라이브러리의 `swap` 이 `noexcept` 이냐 아니냐는 사용자가 정의한 `swap` 이 `noexcept` 인지 아닌지에 따라 결정된다.

```cpp
template<class T, size_t N>
void swap(T (&a)[N], T(&b)[N]) noexcept(noexcept(swap(*a,*b)));

template<class T1, class T2>
struct pair {
    ...
    void swap(pair& p) noexcept(noexcept(swap(first, p.first)) &&
                                noexcpet(swap(second, p.second)));
    ...
}

위 두 함수는 잠정적으로 `noexcept` 이다.(표현식 내부가 `noexcept` 로 구현된다면 함수또한 `noexcept` 이다.) 첫번째 함수는 swap할 두 배열의 요소가 noexcept라면 함수 또한 `noexcept` 가 된다. `pair` 의 경우도 `pair` 가 가지고 있는 요소들이 `noexcept` 라면 `std::pair::swap` 또한 `noexcept` 가 된다.

몇몇 함수에서 `noexcept` 는 매우 중요하다. 메모리 해제 함수들(delete, 소멸자 등)은 암시적으로 `noexcept` 로 선언이 된다. 즉, 사용자가 직접적으로 `noexcept`를 선언해줄 필요는 없는것이다. 소멸자가 예외를 발생시킬 여지가 있는 멤버 데이터를 가지고 있다면,  소멸자가 암시적으로 `noexcept` 가 되지않는다. 이러한 경우는 흔치 않다.

라이브러리 개발자의 경우 함수를 __wide contracts__ 와 __narrow contracts__ 로 나누는 것을 주목할 필요가 있다. wide contracts 함수는 선행조건이 없다. 이런 함수는 프로그램의 상태와 인자의 제한에 상관없이 호출이 가능하다. 또한 이는 절대로 미정의 동작(undefined behavior)이 생기지 않는다.

함수를 작성할 때 이러한 관점은 `noexcept` 로 함수를 선언할지 말지에 대한 힌트를 준다. wide contract의 경우 `noexcept` 로 선언하는건 당연하지만 narrow contract의 경우 애매하다.

std::string` 의 매개변수는 절대로 32개의 문자를 넘어서면 안된다는 조건이 있다고 가정을 해보자. 32개의 문자를 넘는 매개변수를 넘긴다면 미정의 동작이 일어난다. 이 함수는 필요조건(length <= 32)을 체크할 의무는 없기때문에 예외체크를 안한다.(호출자가 이러한 조건을 맞출 의무가 있다.) 필요조건이 있음에도 불구하고, 함수를 `noexcept` 로 선언하는건 적절해 보인다.

하지만 필요조건을 함수 구현단에서 체크한다고 가정해보자. 이는 디버깅에 매우 효과적이다. 그럼 어떻게 필요조건의 위반을 알려주냐는게 문제다. 우리는 필요조건이 위반되었다는 예외를 던질 수 있을 것이다. 하지만 함수가 `noexcept` 로 선언되었다면 이러한 접근법은 불가능하다. 이러한 이유로 wide contract 함수의 경우에만 `noexcept` 로 선언해주는 경우가 많다.

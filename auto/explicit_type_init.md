## Item 6: Use the explicitly typed initializer idiom when auto deduces undesired types.

앞선 Item 5에서 `auto` 를 사용하면서의 여러가지 기술적 이득을 설명했으나, 때때로 타입추론이 우리가 생각한데로 되지 않을 때가 있다.

```cpp
std::vector<bool> features(const Widget& w);
...

Widget w;
...

bool highPriority = features(w)[5];
...

processWidget(w, highPriority);


auto highPriority = features(w)[5];
...

processWidget(w, highPriority);         // undefined behavior
```

위 코드에서 `highPriority` 의 타입선언을 `auto` 로 바꾸게 되면 미정의 동작(undefined behavior)를 유발한다. 그 이유는 `std::vector<bool>` 이 `bool` 타입이 아닌 `bool&` 타입과 유사하게 작동하는 새로운 객체(std::vector<bool>::reference)를 반환한다. 이 객체는 `bool` 타입으로 암시적(implicit)으로 변환이 된다. 

`highPriority` 를 `bool` 타입으로 명시적으로 선언할 경우 리턴되는 객체가 암시적으로 `bool` 타입으로 변환이 된다. 반대로 `auto` 타입으로 선언할 경우 `std::vector<bool>::reference` 가 어떻게 구현되었냐에 따라 다르다. 

한가지 구현으로는 참조하고 있는 비트의 machine word의 포인터를 가지고 있으며 인덱싱은 offset을 그 주소에 더하여 하는 방식이다. `features` 의 호출로 `std::vector<bool>` 의 임시객체를 반환한다. 이 임시객체에서 `operaotr[]` 연산자가 호출되며 `std::vector<bool>::reference` 를 반환한다. 즉 `highPriority` 또한 해당 bool값에 해당되는 포인터를 가지게 되는것이다. 구문(statement)가 끝나고 난 뒤, 이 임시객체는 소멸하며, 그 결과 `highPriority` 는 댕글링 포인터(dangling pointer)를 가지게 된다.

이와 같이 `std::vector<bool>::reference` 는 프록시(proxy)의 하나이다. 이러한 보이지 않는 프록시의 경우 `auto` 와 잘 어울리지 못한다. 이러한 프록시의 객체들은 주로 구문이 끝날때 같이 소멸되도록 설계되있다. 즉, `auto` 타입으로 타입을 추론하면 소멸된 임시객체를 가지게 되며 원치않는 동작을 맞이 할 수 있다.

```cpp
auto someVar = expression of "invisible" proxy class type;  // aviod code of this form.
```

### How can i recognize when proxy objects are in use?

1. 도큐먼트(document)
2. 함수 시그네쳐(function signature)
3. 유닛테스트(unit test)를 통한 디버깅(debug)

### Solution: the explicitly typed initializer idiom.

```cpp
auto highPriority = static_cast<bool>(features(w)[5]);
```

`features(w)[5]` 는 여전히 `std::vector<bool>::reference` 를 반환하지만 캐스팅으로 인해 `auto` 타입은 `bool` 타입으로 추론이된다. 런타임(run-time)동안에도 `std::vector<bool>::reference` 는 `bool` 타입으로 변환이 된다. 이 형식은 프록시 클래스 타입의 초기화 뿐만아니라, 표현식(expression)으로부터 생성된 값의 타입과 다른 타입의 변수를 만든다는 것을 강조할 수 있다.

```cpp
dobule calcEpsilon()

float ep = calcEpsilon();       // impliclitly convert

auto ep = static_cast<float>(calcEpsilon());
```

## Item 9: Prefer alias declarations to typedef

STL을 적극적으로 사용하는 코드를 보면 장황한 타입들을 볼 수 있다. (`std::unique_ptr<std::unordered_map<std::string, std::string>>`) 이를 C++11 이전에는 `typedef` 를 이용해 해결했다.

```cpp
// C++98
typedef 
    std::unique_ptr<std::unordered_map<std::string, std::string>>
    UPtrMapSS;

// C++11
using UPtrMapSS =
    std::unique_ptr<std::unordered_map<std::string, std::string>>
```

C++11 에서는 _alias declarations_ 가 도입되었다. 이는 `typedef` 와 동일하게 작동하지만 몇몇 다른점이 있다.

```cpp
typedef void (*FP)(int, const std::string&);

using FP = void (*)(int, const std::string&);   // easy
```

위와 같이 가독성의 이점이 있지만, 더 중요한 이유는 템플릿에서 나타난다. alias declaration은 템플릿에 이용이 가능하지만(alias templates), typedef는 불가능하다. `typedef` 로 구현이 가능하기는 하지만 조금의 속임수가 필요하다.

```cpp
template<typename T>
using MyAllocList = std::list<T, MyAlloc<T>>;

MyAllocList<Widget> lw;

template<typename T>
struct MyAllocList {
    typedef std::list<T, MyAlloc<T>> type;
};

MyAllocList<Widget>::type lw;
```

만약 템플릿 매개변수로부터 리스트가 가질 객체의 타입을 정할때 `typename` 이 같이 필요하다.

```cpp
template<typename T>
class Widget
{
private:
    typename MyAllocList<T>::type list;
    ...
};

`MyAllocList<T>::type` 은 템플릿 매개변수 `T` 의 타입을 참조한다. 즉 `MyAllocList<T>::type` 은 의존타입( _dependent type_ )이다. C++에서 의존타입에는 항상 `typename` 이 선행해야 한다.

만약 _alias template_ 을 사용한다면, `typename` 은 필요없다.

```cpp
template<typename T>
using MyAllocList = std::list<T, MyAlloc<T>>;

template<typename T>
class Widget
{
private:
    MyAllocList<T> list;
    ...
};
```

보기에는 `MyAllocList<T>` 또한 템플릿 매개변수에 의존적으로 보일지 몰라도 컴파일러는 다르다. 컴파일러가 `Widget` 템플릿을 컴파일할때, _alias template_ 을 사용한 `MyAllocList<T>` 는 타입으로 인식하기 때문에 `typename` 이 필요없다.( _alias template_ 을 사용하면 무조건 타입으로 인식한다.)

반면 `MyAllocList<T>::type` 은 `MyAllcoList` 의 특수화가 존재할지 모르기때문에 타입일지를 확실히 하지 못한다. 

```cpp
template<>
class MyAllocList<Wine>
{
private:
    enum class WineType
    { White, Red, Rose };
    
    WindType type;
    ...
};
```

위의 특수화에서 `MyAllocList<Wine>::type` 은 타입이 아니다. 만약 `Wine` 으로 인스턴스화가 된다면, `MyAllocList<T>::type` 은 데이터 멤버를 참조하지 타입이 아니다. 이러한 이유때문에 컴파일러는 `typename` 의 선행을 강제한다.

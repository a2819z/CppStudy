## Item 16: Make const member functions thread safe.

수학분야에서 일을 한다면, 다항식 클래스를 만들어 두는게 편한점이 많다. 이 클래스는 다항식의 해를 계산하는 함수가 존재한다. 다항식의 해를 구하면서 객체의 내부상태를 변환시키지 않으며 이는 `const` 함수로 선언되는게 당연하다.

```cpp
class Polynomial
{
public:
    using RootsType = std::vector<double>;

    RootsType roots() const;
    ...
};
```

이때 해를 계산하는 프로세스에는 많은 비용이 들수있다. 그러므로 우리는 단한번의 계산 후 결과값을 캐싱(caching)해 이후 다시 `roots` 함수를 호출할때 계산을 생략하고 캐싱된 값을 반환할려 한다.

```cpp
class Polynomial
{
public:
    using RootsType = std::vector<double>;

    RootsType roots() const
    {
        if (!rootsAreValid) {
            ...

            rootsAreValid = true;
        }

        return rootVals;
    }

private:
    mutable bool rootsAreValid{ false };
    mutable RootsType rootVals{};
};
```

원래라면 `roots` 함수는 객체내부를 변경시키지 않지만 캐싱을 위해 일부분을 변경시킨다. 이때 우리는 `mutable` 을 사용한다.

두개의 쓰레드(thread)가 동시에 하나의 객체에서 `Polynomial::roots` 를 호출한다고 가정해보자. `const` 멤버 함수는 읽기연산(read operation)을 뜻하며 다수의 쓰레드가 동기화없이 읽기연산을 사용하는것엔 위험성이 없다. 하지만 이경우엔 다르다. 하나 또는 두개의 쓰레드가 `rootsAreValid` 와 `rootVals` 변수를 동시에 수정할 가능성이 있기때문이다. 즉, 서로 다른 쓰레드가 동기화없이 같은 메모리에 읽기/쓰기를 한다는 것이며, 이는 데이터 레이스(data race)를 유발한다.

이를 해결하는 법은 `std::mutex` 를 사용하면 된다. 

```cpp
class Polynomial
{
public:
    using RootsType = std::vector<double>;

    RootsType roots() const
    {
        std::lock_guard<std::mutex> g(m);       // lock mutex

        if (!rootsAreValid) {
            ...

            rootsAreValid = true;
        }

        return rootVals;                        // unlock mutex
    }

private:
    mutable std::mutex m;
    mutable bool rootsAreValid{ false };
    mutable RootsType rootVals{};
}
```

`std::atomic` 을 사용하는 방법이 있지만 두개이상의 변수를 수정할때는 문제가 발생할 소지가 있다.

```cpp
class Polynomial
{
public:
    using RootsType = std::vector<double>;

    RootsType roots() const
    {
        if (!rootsAreValid) {
            ...

            rootVals = expensiveComputation();
            rootsAreValid = true;
        }

        return rootVals;                        // unlock mutex
    }

private:
    mutable std::atomic<bool> rootsAreValid{ false };
    mutable std::atomic<RootsType> rootVals{};
}
```

첫번째 쓰레드가 계산을 다마치기전 두번째 쓰레드가 접근을 한다면 아직 `rootsAreValid` 가 true로 바뀌지 않아 똑같은 연산을 실행할 수 있다. 반대로 `rootVals` 과 `rootsAreValid` 의 순서를 바꾼다 해도 문제가 있다. 먼저 true 바꾸고 계산을 실행하는 도중 두번째 쓰레드가 접근하면 아직 계산되지도 않은 `rootVals` 를 반환하려고 들것이다.

이를 통해 우리는 두곳 이상의 메모리에 접근할 시 `std::atomic` 보단 `std::mutex` 로 뮤텍스를 잠그고 푸는것이 더 효율적이고 안전하다는 것을 알 수 있다.

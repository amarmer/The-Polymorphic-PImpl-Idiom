# The Polymorphic PImpl Idiom

## Why the Polymorphic PImpl Idiom?

The PImpl (Pointer to Implementation) idiom creates compilation firewalls, but the Classic PImpl often comes with a "Proxy Tax" - 
the tedious task of manually forwarding every public method to the hidden implementation.</br>
The Polymorphic PImpl eliminates this boilerplate; it provides a zero-maintenance compilation firewall.</br>
This pattern simplifies maintenance and enables native mocking for unit tests.

## Applying the Idiom

**Calculator.h**
```cpp
#pragma once

#include "PImpl.h"

// The Public Interface
struct ICalculator {
    // Required by PImpl: checked at compile time.
    virtual ~ICalculator() = default;
    
    virtual void Add(int n) = 0;
    virtual int Sum() const = 0;
};

class Calculator: public PImpl<ICalculator> {
public:
    explicit Calculator(int sum);
};
```

**CalculatorImpl.cpp**
```cpp
#include "Calculator.h"

class CalculatorImpl: public ICalculator {
    int sum_;
public:
    CalculatorImpl(int sum): sum_(sum) {}

    void Add(int n) override { sum_ += n; }
    int Sum() const override { return sum_; }
};

Calculator::Calculator(int sum) 
    : PImpl(std::in_place_type<CalculatorImpl>, sum) 
{}
```

**main.cpp**
```cpp
#include "Calculator.h"
#include <iostream>

int main() {
    Calculator calc(10);

    calc->Add(5);

    std::cout << calc->Sum() << std::endl;
    
    return 0;
}
```

## The PImpl Base Class

**PImpl.h**
```cpp
#pragma once
#include <memory>
#include <utility>
#include <type_traits>

template <typename Interface>
class PImpl {
    static_assert(std::has_virtual_destructor_v<Interface>, "Interface must have a virtual destructor");

public:
    Interface* operator->() const { return ptr_.get(); }

protected:
    template <typename Implementation, typename... Args>
    PImpl(std::in_place_type_t<Implementation>, Args&&... args) {
        ptr_ = std::make_unique<Implementation>(std::forward<Args>(args)...);
    }

private:
    std::unique_ptr<Interface> ptr_;
};
```

*The complete source code is available at https://wandbox.org/permlink/4mkDPXssSC0etzSW*

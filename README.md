# An alternative to the classic PImpl idiom

## Introduction

This article describes a template-based pattern for hiding class implementations behind a pure virtual interface.</br>
It is an alternative to the classic PImpl idiom.

## Applying the Pattern

**Calculator.h**
```cpp
#pragma once

#include "PImpl.h"

// The Public Interface
struct ICalculator {
    // No virtual destructor needed.
    
    virtual void Add(int n) = 0;
    virtual int Sum() const = 0;
};

class Calculator: public PImplHandle<ICalculator> {
public:
    explicit Calculator(int factor);
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
    : PImplHandle(std::in_place_type<CalculatorImpl>, sum) 
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

template <typename Interface>
class PImplHandle {
public:
    Interface* operator->() const { return ptr_.get(); }

protected:
    template <typename Implementation, typename... Args>
    PImplHandle(std::in_place_type_t<Implementation>, Args&&... args) {

        // With 'final' and 'shared_ptr', no need for a virtual destructor in 'Interface'.
        struct FinalWrapper final: public Implementation {
            using Implementation::Implementation;
        };

        // shared_ptr handles the non-virtual destructor safety
        ptr_ = std::make_shared<FinalWrapper>(std::forward<Args>(args)...);
    }

private:
    std::shared_ptr<Interface> ptr_;
};
```

*The complete source code is available at https://wandbox.org/permlink/tbZxnTDCFKsifhQv*

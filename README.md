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
    Calculator();
    explicit Calculator(int sum);
};
```

**CalculatorImpl.cpp**
```cpp
#include "Calculator.h"

class CalculatorImpl: public ICalculator {
    int sum_;
public:
    CalculatorImpl(): sum_(0) {}
    CalculatorImpl(int sum): sum_(sum) {}

    void Add(int n) override { sum_ += n; }
    int Sum() const override { return sum_; }
};

Calculator::Calculator() 
    : PImpl(std::in_place_type<CalculatorImpl>) 
{}

Calculator::Calculator(int sum) 
    : PImpl(std::in_place_type<CalculatorImpl>, sum) 
{}
```

**main.cpp**
```cpp
#include "Calculator.h"
#include <iostream>

int main() {
    Calculator calc1(10);

    Calculator calc2;
    calc2 = calc1;
    calc2->Add(1);
    
    calc1->Add(5);

    Calculator calc3(calc1);
    calc3->Add(1);

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
class PImpl {
  static_assert(std::has_virtual_destructor_v<Interface>, "Interface needs a virtual destructor");

   /*
   * IClone and CloneT provide "Virtual Copying":
   * 1. Deep Copy: unique_ptr<Interface> can't copy itself because it's abstract.
   * 2. Type Erasure: CloneT "remembers" the concrete type (e.g., CalculatorImpl)
   *    so the base class can duplicate it without the header knowing its definition.
   */
  struct IClone {
    virtual std::unique_ptr<Interface> Clone(const Interface* p) const = 0;
  };

  template <typename T>
  struct CloneT: public IClone {
    std::unique_ptr<Interface> Clone(const Interface* p) const override {
      return std::make_unique<T>(*static_cast<const T*>(p));
    }
  };

  const IClone* pClone_ = nullptr;
  std::unique_ptr<Interface> pInterface_;

protected:
  ~PImpl() = default;

  // Constructor
  template <typename Implementation, typename... Args>
  PImpl(std::in_place_type_t<Implementation>, Args&&... args)
    : pInterface_(std::make_unique<Implementation>(std::forward<Args>(args)...)) {
    static const CloneT<Implementation> globalCloner;
    pClone_ = &globalCloner;
  }

  // Copy/Move Operations
  PImpl(const PImpl& other) {
    CopyFrom(other);
  }

  PImpl& operator=(const PImpl& other) {
    if (this != &other) {
        CopyFrom(other);
    }
    return *this;
  }
    
  PImpl(PImpl&& other) noexcept {
    MoveFrom(other);
  }

  PImpl& operator=(PImpl&& other) noexcept {
    if (this != &other) {
        MoveFrom(other);
    }
    return *this;
  }
    
  void CopyFrom(const PImpl& other) {
    pClone_ = other.pClone_;
    pInterface_ = other.pClone_->Clone(other.pInterface_.get());
  }

  void MoveFrom(PImpl& other) noexcept {
    pClone_ = other.pClone_;
    pInterface_ = std::move(other.pInterface_);
    other.pClone_ = nullptr;
  }
    
public:
  Interface* operator->() const { return pInterface_.get(); }
};
```

*The complete source code is available at https://wandbox.org/permlink/p627TDfHhMXFdnZn*

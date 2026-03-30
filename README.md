# The Polymorphic PImpl Idiom

## Why the Polymorphic PImpl Idiom?

The PImpl (Pointer to Implementation) idiom creates compilation firewalls, but the Classic PImpl often comes with a "Proxy Tax" — 
the tedious task of manually forwarding every public method to the hidden implementation.

The Polymorphic PImpl eliminates this boilerplate and provides four key advantages over the Classic PImpl:

1. **No Proxy Tax** — no manual forwarding wrappers needed; adding methods requires no changes to the public class
2. **Native mocking** — create `MockCalculator: public ICalculator` for unit tests without touching `PImpl`
3. **Multiple implementations** — swap `CalculatorImpl` for a different implementation without changing the header
4. **Value semantics** — copy and move operations are automatically provided; copying performs a deep copy

## Applying the Idiom

**Calculator.h**
```cpp
#pragma once

#include "PImpl.h"

// The Public Interface
struct ICalculator {
    virtual ~ICalculator() = default; // Enforced in PImpl.
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

public:
  Interface* operator->() const { return pInterface_.get(); }
    
protected:
  ~PImpl() = default;

  // Constructor
  template <typename Implementation, typename... Args>
  PImpl(std::in_place_type_t<Implementation>, Args&&... args)
    : pInterface_(std::make_unique<Implementation>(std::forward<Args>(args)...)), 
      pClone_(&CloneImpl<Implementation>) {}

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
    
private:
  template <typename T>
  static std::unique_ptr<Interface> CloneImpl(const Interface* p) {
    return std::make_unique<T>(*static_cast<const T*>(p));
  }    
    
  void CopyFrom(const PImpl& other) {
    pClone_ = other.pClone_;
    pInterface_ = other.pClone_(other.pInterface_.get());
  }

  void MoveFrom(PImpl& other) noexcept {
    pClone_ = other.pClone_;
    pInterface_ = std::move(other.pInterface_);
    other.pClone_ = nullptr;
  }

  std::unique_ptr<Interface> pInterface_;
  std::unique_ptr<Interface>(*pClone_)(const Interface*);
};
```

*The complete source code is available at https://wandbox.org/permlink/RgWQZZbfO9zwydac*

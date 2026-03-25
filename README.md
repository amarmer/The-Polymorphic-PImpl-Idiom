# An alternative to the classic PImpl idiom

## Introduction

This article describes a template-based pattern for hiding class implementations behind a pure virtual interface.</br>
It is an alternative to the classic PImpl idiom.

## Applying the Pattern

**Counter.h**
```cpp
#pragma once

#include "PImpl.h"
#include <memory>

struct ICounter {
    // Required by PImpl: checked at compile time
    virtual ~ICounter() = default;
    virtual void Inc() = 0;
    virtual int Count() const = 0;
};

std::unique_ptr<ICounter> ConstructCounter(int initialCount);

class Counter: public PImpl<ConstructCounter> {
    using PImpl::PImpl;
};
```

**CounterImpl.cpp**
```cpp
#include "Counter.h"

class CounterImpl : public ICounter {
public:
    CounterImpl(int initialCount) : count_(initialCount) {}
    void Inc() override        { count_++; }
    int Count() const override { return count_; }
private:
    int count_ = 0;
};

std::unique_ptr<ICounter> ConstructCounter(int initialCount) {
    return std::make_unique<CounterImpl>(initialCount);
}
```

**main.cpp**
```cpp
#include "Counter.h"
#include <iostream>

int main() {
    Counter counter(10);

    counter->Inc();
    
    int count = counter->Count();

    std::cout << "count: " << count << std::endl;
    
    return 0;
}
```

## Multiple Constructors

When a class needs multiple constructors, pass multiple factory functions to `PImpl`.</br>
The first factory whose signature matches the arguments is called at compile time.</br>
Use `PImplConstructor` to resolve overloaded factory names:

```cpp
std::unique_ptr<ICounter> ConstructCounter1();
std::unique_ptr<ICounter> ConstructCounter2(int initialCount);

class Counter : public PImpl<ConstructCounter1, ConstructCounter2> {
    using PImpl::PImpl;
};

Counter c1;     // calls ConstructCounter()
Counter c2(5);  // calls ConstructCounter(int)
```

## The PImpl Base Class

**PImpl.h**
```cpp
#pragma once

#include <type_traits>
#include <functional>
#include <memory>

// Implementation details.
namespace PImplDetail {

template<typename T>
struct is_unique_ptr : std::false_type {};

template<typename T, typename D>
struct is_unique_ptr<std::unique_ptr<T, D>> : std::true_type {};

template<auto F, typename Void, typename... Args>
struct is_callable_impl : std::false_type {};

template<auto F, typename... Args>
struct is_callable_impl<F, std::void_t<decltype(F(std::declval<Args>()...))>, Args...>
    : std::true_type {};

template<auto F, typename... Args>
constexpr bool is_callable_v = is_callable_impl<F, void, Args...>::value;

template<auto First, auto... Rest>
struct selector {
    template<typename... Args>
    static auto select(Args&&... args) {
        if constexpr (is_callable_v<First, Args...>) {
            return First(std::forward<Args>(args)...);
        } else {
            static_assert(sizeof...(Rest) > 0, "No matching constructor found");
            return selector<Rest...>::select(std::forward<Args>(args)...);
        }
    }
};

template<auto F>
struct return_type;

template<typename Ret, typename... Args, Ret(*F)(Args...)>
struct return_type<F> {
    using type = Ret;
};

template<auto F>
using return_type_t = typename return_type<F>::type;

} // namespace PImplDetail

// Base class for PImpl pattern.
template <auto... Constructors>
class PImpl {
    using unique_ptr_t = PImplDetail::return_type_t<std::get<0>(std::make_tuple(Constructors...))>;

    static_assert(PImplDetail::is_unique_ptr<unique_ptr_t>::value,
                  "Constructors must return a std::unique_ptr");

    static_assert(std::has_virtual_destructor_v<typename unique_ptr_t::element_type>,
                  "Interface must have a virtual destructor");

public:
    template<typename... Args>
    PImpl(Args&&... args)
        : impl_(PImplDetail::selector<Constructors...>::select(std::forward<Args>(args)...)) {}

    using interface_t = typename unique_ptr_t::element_type;
    
    interface_t* operator->() {
        return impl_.get();
    }

    const interface_t* operator->() const {
        return impl_.get();
    }

private:
    PImpl(const PImpl&) = delete;
    PImpl& operator=(const PImpl&) = delete;

protected:
    unique_ptr_t impl_;
};
```



*The complete source code is available at https://wandbox.org/permlink/6EKqG3EqYtjxzzhP*

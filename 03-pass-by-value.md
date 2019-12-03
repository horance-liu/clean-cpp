## Clean C++：为"按值传递"正名

在`C++98`中，按值传递(pass-by-value)意味着低效的、不必要的拷贝，被广大程序员嗤之以鼻。按照惯例，对于自定义类型，如果用于入参则应使用`pass-by-const-reference`；如果用于出参则应`pass-by-reference`。

在缺失移动语义`(move semantics)`下的`C++98`标准，这样的结论无可厚非。但是，在增加移动语义`(move semantics)`下的`C++11`标准，按值传递在某些特殊场景具有优异的表现力，是时候为"按值传递"正名了。

### 一个例子

存在两个使用`std::function`定义的「仿函数」。其中，`std::function`的泛型参数使用函数的原型表达。例如`std::function<bool(int)>`的泛型参数是一个一元的谓词。

```cpp
#include <string>
#include <functional>

using Matcher = std::function<bool(int)>;
using Action = std::function<std::string(int)>;
```

### C++98风格

存在一个`Atom`的函数对象。按照既有的`C++98`的习惯，使用`pass-by-const-reference`传递参数，然后将它们拷贝至私有数据区。此时必然调用`std::function`的「拷贝构造函数」。

```cpp
struct Atom {
  Atom(const Matcher& matcher, const Action& action)
    : matcher(matcher), action(action) {
  }
  
  std::string operator()(int m) const {
    return matcher(m) ? action(m) : "";
  }
  
private:
  Matcher matcher;
  Action action;
};
```

可是，当传递给`Atom`构造函数的是「右值」时，我们期望调用「移动构造函数」，而非「拷贝构造函数」；因为对于`std::function`，移动构造相对拷贝构造更加低廉。一般地，针对这个问题，存在3种解决方案，我们逐一分析。

### 重载函数

使用重载的构造函数，可以准确的根据左值或右值实现函数指派。但是，为了支持两个参数的重载，便要实现4个重载的构造函数，可谓得不偿失。组合爆炸是一种典型的设计缺陷，为了提升程序性能，该方案所付出的成本相当昂贵。

```cpp
struct Atom {
  Atom(const Matcher& matcher, const Action& action)
    : matcher(matcher), action(action) {
  }
  
  Atom(const Matcher& matcher, Action&& action)
    : matcher(matcher), action(std::move(action)) {
  }
  
  Atom(Matcher&& matcher, const Action& action)
    : matcher(std::move(matcher)), action(action) {
  }

  Atom(Matcher&& matcher, Action&& action)
    : matcher(std::move(matcher)), action(std::move(action)) {
  }
  
  std::string operator()(int m) const {
    return matcher(m) ? action(m) : "";
  }
  
private:
  Matcher matcher;
  Action action;
};
``` 

#### 成本分析

当传递左值，存在一次「拷贝构造」；当传递右值，存在一次「移动构造」。但是，存在不可接受的组合爆炸的问题。

### 透传引用

「透传引用(Forward Reference)」是`C++11`实现「完美转换(Perfect Forward)」机制而引入的一个概念。使用「透传引用」可以合并上述4个构造函数，实现左值和右值的透明传递和分发。但是，使用「透传引用」引入了泛型设计，其实现必须放到头文件，增加了编译时依赖的成本，极大地增加了实现的复杂度。而且，因为模板而导致代码膨胀的问题，相对上述重载方案其目标代码规模并没有得到改善，甚至更糟。此外，鉴于「完美转换」并非100%的完美，当客户传递不正确的类型时，编译器的错误信息相当冗长。

```cpp
struct Atom {
  template <typename Matcher, typename Action>
  Atom(Matcher&& matcher, Action&& action)
    : matcher(std::forward<Matcher>(matcher))
    , action(std::forward<Action>(action)) {
  }
  
  std::string operator()(int m) const {
    return matcher(m) ? action(m) : "";
  }
  
private:
  Matcher matcher;
  Action action;
};
``` 

#### 成本分析

当传递左值，存在一次「拷贝构造」；当传递右值，存在一次「移动构造」。避免了组合爆炸的问题，但引入了模板的复杂度，及其代码膨胀的问题。但是，相对重载方法，代码实现更加简洁，更加紧凑。

### 按值传递

考虑使用`pass-by-value`的方式传递`Matcher`和`Action`，不仅实现简单，天然支持左值或右值，而且成本在接受的范围之内。

```cpp
struct Atom {
  Atom(Matcher matcher, Action action)
    : matcher(std::move(matcher))
    , action(std::move(action)) {
  }
  
  std::string operator()(int m) const {
    return matcher(m) ? action(m) : "";
  }
  
private:
  Matcher matcher;
  Action action;
};
``` 

#### 成本分析

当传递左值，存在一次「拷贝构造」，及其一次「移动构造」；当传递右值，存在两次次「移动构造」。相对上两个方案，两种场景其成本均多了一次低廉的「移动构造」。但是，该方案完全避免了组合爆炸的问题，及其模板的复杂度，代码最为简洁。

### 何时按值传递

综上述，可以归纳如下条件，可以考虑使用「按值传递」的方法。

- 产生副本；
- 类型可拷贝；
- 移动成本低廉。

例如，上述的函数对象`Atom`，完全满足上述必要条件。

- 产生副本：通过「拷贝构造」或「移动构造」，产生私有的`matcher`和`action`副本；
- 类型可拷贝：`std::function`类型可拷贝。
- 移动成本低廉：`std::function`的移动构造的成本非常低廉。

### 在Lambda中的应用

事实上，C++98风格的函数对象，都可以使用Lambda简化实现。例如，原来的函数对象`Atom`实现，可以使用更为简洁的Lambda表达式实现。

```cpp
using Rule = std::function<std::string(int)>;

Rule atom(Matcher matcher, Action action) {
  return [matcher, action](int m) {
    return matcher(m) : action(m) : "";
  };
}
```

但是，在Lambda表达式的「参数捕获列表」中，`matcher, action`是按值传递给闭包对象的，此时调用的是`std::function`的「拷贝构造函数」。幸运的是，`C++14`支持以初始化的方式将对象移动至闭包之中，弥补了这个遗憾。

```cpp
using Rule = std::function<std::string(int)>;

Rule atom(Matcher matcher, Action action) {
  return [matcher = std::move(matcher), action = std::move(action)](int m) {
    return matcher(m) : action(m) : "";
  };
}
```






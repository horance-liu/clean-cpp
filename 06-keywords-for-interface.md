## 应用准关键字定义抽象接口

使用`C++`定义纯粹的抽象接口类型，与定义普通的类并无区别，只是在形态上具有特殊的表现形式: 它拥有一个公开的、空实现的、虚拟的析构函数。

```cpp
struct Rule {
  virtual std::string apply(int n) const = 0;
  virtual ~Rule() {}
}
```

不幸的是，每当定义一个接口类型时，都要小心翼翼地定义虚拟析构函数，这不仅仅是重复设计的问题；更有甚者，这样的惯用法必然存在安全的隐患，万一存在一个程序员在喝酒之后定义给你一个抽象接口呢？

### 抽象接口

与其让程序员存在犯罪的风险，不如预防错误发生。既然程序员容易遗忘，那么就自动「生成代码」，这样便可高枕无忧了。

> 预防优于治疗，将错误扼杀在源头。

```cpp
namespace cub {

namespace details {
template <typename T>
struct Interface {
  virtual ~Interface() {
  }
};
}  // namespace details

#define INTERFACE(type) \
struct type : ::cub::details::Interface<type>

} // namespace cub
```

有了`INTERFACE`的宏定义，定义抽象的接口类型，可谓易如反掌。

```cpp
INTERFACE(Rule) {
  virtual std::string apply(int n) const = 0;
};
```

### 扩展接口

但是，应用`INTERFACE`定义多重继承的接口类型时，将面临困境。因此，需要定义另外的两个「准关键字」，`Java`程序员对这两个关键字早已司空见惯了。

```cpp
#define EXTENDS(...) , ##__VA_ARGS__
#define IMPLEMENTS(...) EXTENDS(__VA_ARGS__)
```

例如，接口`Matcher`扩展另外一个抽象接口`SelfDescribing`，则可以使用`EXTENDS`。

```cpp
struct Description;

INTERFACE(SelfDescribing) {
  virtual void decribeTo(Description&) const = 0;
};

INTERFACE(Matcher) EXTENDS(SelfDescribing) {
  virtual bool matches() const = 0;
};
```

### 抽象方法

程序员定义纯虚函数时，特别容易遗忘尾部的`= 0`的后缀。纯虚函数声明的是抽象方法，而虚函数不能够准确表达这个语义的。同理，应用「代码生成」的机制，杜绝错误发生的可能性。

```cpp
#define ABSTRACT(...) virtual __VA_ARGS__ = 0
```

使用`ABSTRACT`可以增强代码的可读性，向用户声明这是一个抽象方法，而不是纯虚函数。而且，`ABSTRACT`置于行首，而非纯虚函数首尾声明`virtual ... = 0`，相对更不容易犯错。

```cpp
INTERFACE(Rule) {
  ABSTRACT(std::string apply(int n) const);
};
```

### 覆写方法

当子类覆写虚函数时，可以使用`override`增强编译时的安全性，及其改善代码的可读性。但是，`C++11`标准的`override`需要标注到函数签名的尾部，往往也被程序员遗忘。事实上，遗忘`override`并非什么大错，但会失去编译时安全的保护。

```cpp
#define OVERRIDE(...) __VA_ARGS__ override
```

定义`OVRIRIDE`的准关键字，将其重要性置于行首，增强其重要性。

```cpp
struct Atom : Rule {
  Atom(Matcher* matcher, Action* action);
  
private:
  OVERRIDE(std::string apply(int n) const);
  
private:
  Matcher* matcher;
  Action*  action;  
};
```

### 其他关键字

那么是不是应该都将`C++`的其他关键字都定义为宏呢？例如`const, constexpr, static`。回答当然是"No"，我们只会定义程序员容易出错的关键场景，例如，定义抽象接口时遗漏虚拟析构函数，定义纯虚函数时遗漏行末的`= 0`，子类覆写虚函数时遗漏`override`。

也许你会对使用宏而心有余悸，其实完全没必要担心。当你习惯了这套「准关键字」，你想犯错都难；而使用原生C++的关键字，我肯定你会犯错。

### 增强的抽象接口

上文定义的`cub::details::Interface`类，它只定义了虚拟析构函数。遗憾的是，在新的`C++11`标准里，这将阻止`Interface`自动生成移动构造函数，及其移动赋值运算符；而且，自动生成的拷贝构造函数，及其拷贝赋值运算符的规则也被标准废弃，在未来的实现中可能被抛弃。

那么，使用`INTERFACE`定义的抽象接口，其实现该接口的所有子类将不可移动，只能做保守的拷贝操作。这样的拷贝行为，甚至被标准实现所废弃。

因此在`C++11`的实现中，`Interface`的基类需要进行稍许的改进，使用`default`显式声明所有需要自动生成的函数。注意，析构函数声明为`public, virtual`的，其他声明为`protected`即可。

```cpp
template <typename T>
struct Interface {
  virtual ~Interface() = default;
  
protected:
  Interface() = default;

  Interface(Interface&&) noexcept = default;
  Interface& operator=(Interface&&) noexcept = default;

  Interface(const Interface&) = default;
  Interface& operator=(const Interface&) = default; 
};
```

背景：云存储的计费模块，租户依据资源量与租期计算费用。

租户可以持有多份租期，在每个租期内可关联特定容量和大小的存储资源；支持多种类型的存储资源，包括对象存储，文件存储，块存储。




## 实战

> 需求1： 存在一个机器学习的模型列表，查找一个标注为resnet的模型。

### 快速实现

```cpp
Model* find_resnet(Model* models, U32 num) {
  for (U32 i = 0; i < num; i++) {
    if (models[i].name() == "ResNet") {
      return &models[i];
    }
  }
  return nullptr;
}
```

这段代码的逻辑已经足够简单了，但设计上也存在很多明显的坏味道。构造用例也极为麻烦，很多细节都暴露给了用户，增加了用户的心智负担。全局函数`find_resnet`缺失模块封装性，可重用性较低。

```cpp
FIXTURE(ModelStoreSpec) {
  enum {MAX_MODEL_NUM = 64};

  U8 num = 0;
  Model* models[MAX_MODEL_NUM];

  TEARDOWN {
    for (U8 i = 0; i < num; i++) {
      delete models[i];
    }
  }

  TEST("find first resnet by model name") {
    models[num++] = new TensorFlowModel("ResNet", 1);
    models[num++] = new TensorRtModel("GoogleNet", 1);

    ASSERT_NE(nullptr, find_resnet(*models, num));
  }
};
```

### 封装

全局函数`find_resnet`不是原罪，但它存在两个方面的问题。一方面，因为缺失相关领域概念，不易于被用户发现和调用；另一方面，每次调用`find_resnet`都要传递`models, num`两个参数。这不仅仅是繁琐的问题，更重要地是，它将数组存储的数据结构的细节暴露给所有的用户代码中，导致`find_resnet`与所有用户代码产生紧密的耦合，这是一种隐式的重复设计。

为了消除这个重复，就是要分离两者变化方向，让用户依赖于更稳定的抽象。此处，通过提取`ModelStore`的领域概念，封装其实现的具体数据结构细节，让用户依赖于更加稳定的抽象，实现两者之间的正交。

```cpp
// model_store.h
struct ModelStore {
  ModelStore();
  ~ModelStore();

  void add(Model*);

  Model* findResNet() const;

private:
  enum {MAX_MODEL_NUM = 64};

  U32 num;
  Model* models[MAX_MODEL_NUM];
};
```

此处，`ModelStore`使用静态内存管理的方式持有`Model`列表，简化了内存管理的成本。


```cpp
// model_store.cc
ModelStore::ModelStore() : num(0) {
}

ModelStore::~ModelStore() {
  for (U32 i = 0; i < num; i++) {
    delete models[i];
  }
}

void ModelStore::add(Model* model) {
  models[num++] = model;
}

Model* ModelStore::findResNet() const {
  for (U32 i = 0; i < num; i++) {
    if (models[i]->name() == "ResNet") {
      return models[i];
    }
  }
  return nullptr;
}
```

经过封装的重构，用户接口也变得更加简洁了。

```cpp
FIXTURE(ModelStoreSpec) {
  ModelStore store;
    
  TEST("find first resnet by model name") {
    store.add(new TensorFlowModel("ResNet", 1));
    store.add(new TensorRtModel("GoogleNet", 1));  

    ASSERT_NE(nullptr, store.findResNet());
  }
};
```

一般地，在嵌入式系统，如果频繁地申请动态内存，会导致内存碎片的堆叠。另外，原生的`new`在堆上申请内存，其实时性得不到保证，在实时性要求比较高的场景下是不可容忍的。因此，一般需要构建内存池的基础设施，重载运算符`operator new/operator delete`构造领域对象，在此不再阐述。

### 重复设计

`ModelStore::findResNet`遗留的问题也很明显，硬编码实现了字符串`ResNet`。硬编码存在的问题就是将算法逻辑与配置数据高度耦合在一起，两者的变化速度是不一致的。线性查找的算法逻辑较为稳定，而配置数据的变化速度较为不稳定，两者耦合在一块，必然导致线性查找的算法逻辑不可重用。例如，增加如下需求，必然导致重复设计。

> 需求2： 存在一个机器学习的模型列表，查找一个标注为googlenet的模型。

```cpp
Model* ModelStore::findGoogleNet() const {
  for (U32 i = 0; i < num; i++) {
    if (models[i]->name() == "GoogleNet") {
      return models[i];
    }
  }
  return nullptr;
}
```

重复是万恶之源，它不仅显著地增加了代码量，而且导致代码进一步恶化。例如，当需要优化算法实现时，将线性查找替换为散列查找，将导致两个函数的原型和实现都要修改，导致代码的散弹式修改。

### 参数化设计

为了消除这两个函数实现之间的重复逻辑，可以将算法逻辑与配置数据两个变化方向分离，可以引入参数化设计，消除两者之间的重复代码。此时，需要特别注意公共函数的命名，使其实现与命名保持在同一个抽象层次。`find`或`findModel`太过抽象，`findByName`刚好能够表达当前的设计意图。

```cpp
Model* ModelStore::findByName(const std::string& name) const {
  for (U32 i = 0; i < num; i++) {
    if (models[i]->name() == name) {
      return models[i];
    }
  }
  return nullptr;
}
```

经过重构，用户在使用新的接口时，根据上下文传递相应的配置数据。

```cpp
ASSERT_NE(nullptr, store.findByName("ResNet"));
```

### 重复再现

如果再增加如下需求，必然再次导致重复设计。

> 需求3： 存在一个机器学习的模型列表，查找一个版本号为2的模型。

```cpp
Model* ModelStore::findByVersion(U32 version) const {
  for (U32 i = 0; i < num; i++) {
    if (models[i]->version() == version) {
      return models[i];
    }
  }
  return nullptr;
}
```

此时，`findByName`与`findByVersion`虽然各自分离了自己相关的逻辑与配置，但两者之间的逻辑再次重复。如果盲目地依赖于参数化设计消除重复，则会导致函数参数的数目剧增。例如，此处引入枚举变量，消除两者之间的重复，导致代码的理解性和维护性急剧下降。参数化设计虽然简单，但过度依赖也是存在问题的，那问题究竟在哪里呢？让我们接着往下仔细分析。

```cpp
enum FindSpec {
  BY_NAME,
  BY_VERSION,
};

Model* ModelStore::find(
    FindSpec spec, const std::string& name, U32 version) const {
  switch (spec) {
  case BY_NAME:    
    return findByName(name);
  case BY_VERSION: 
    return findByVersion(version);
  default:
    return nullptr;
  }
}
```

### 抽象准则

上述引入参数化设计并没有完美地解决重复设计，主要因为算法逻辑与查找规则耦合在一起。参数化设计的抽象层次太低，不能有效地分离两者变化方向。此时，需要引入一个更本质的抽象，以便隔离这两个变化方向，让其两者能够独立变化。

提取`ModelSpec`的抽象概念，它表示一个`Model`满足某种规范，无论是匹配名字，还是匹配版本号。

```cpp
struct ModelSpec {
  virtual bool satisfy(const Model& model) const = 0;
  virtual ~ModelSpec() {}
};
```

此时，`ModelStore::find`的实现变得极为简单。

```cpp
Model* ModelStore::find(const ModelSpec& spec) const {
  for (U32 i = 0; i < num; i++) {
    if (spec.satisfy(*models[i])) {
      return models[i];
    }
  }
  return nullptr;
}
```

然后，将目前存在的两种可能查找规则对象化，建立了两个简单的算子。

```cpp
struct NameSpec : ModelSpec {
  NameSpec(std::string name) : name(std::move(name)) {
  }
  
private:
  bool satisfy(const Model& model) const override {
    return name == model.name();
  }
  
private:
  std::string name;
};

struct VersionSpec : ModelSpec {
  VersionSpec(U32 version) : version(version) {
  }
  
private:
  bool satisfy(const Model& model) const override {
    return version == model.version();
  }
  
private:
  U32 version;
};
```

当用户调用`ModelStore::find`时，根据具体场景，选择相应的算子和配置数据。

```cpp
ASSERT_NE(nullptr, store.find(NameSpec("ResNet")));
```

至此，`ModelStore::find`的重用程度是极高的，它与具体的查找算子是正交的，都可以独立地变化，互不影响。


### 结构性重复

`NameSpec`和`VersionSpec`之间存在「结构型重复」，它们都拥有一个构造函数，一个私有成员，及其覆写`satisfy`成员函数。这两个小类存在无非是为了实现「闭包」的能力，可以使用lambda表达式，消除两者之间的重复代码，简明扼要。

为了引入`Lambda`，需要重构`ModelStore::find`成员函数，应用编译时多态，替代运行时多态。

```cpp
template <typename ModelSpec>
Model* ModelStore::find(ModelSpec spec) const {
  for (U32 i = 0; i < num; i++) {
    if (spec(*models[i])) {
      return models[i];
    }
  }
  return nullptr;
}
```

经过重构，便可以完全删除`NameSpec`和`VersionSpec`两个小类了。而代表用户接口的测试用例，可以直接使用更加短小的Lambda表达式。

```cpp
ASSERT_NE(nullptr, store.find([](auto& model) {
  return model.name() == "ResNet";
}));

ASSERT_NE(nullptr, store.find([](auto& model) {
  return model.name() == "GoogleNet";
}));
```

### 重复的Lambda

经仔细分析，上述的测试用例中，两个Lambda表达式也存在结构性重复。为了消除重复，分离算法与配置两个变化方向。可以应用工厂方法，创建Lambda表达式。

```cpp
using ModelSpec = std::function<bool(const Model&)>;

ModelSpec model_name(std::string name) {
  return [name = std::move(name)](auto& model) {
    return model.name() == name;
  };
}

ModelSpec model_version(int version) {
  return [version](auto& model) {
    return model.version() == version;
  };
}
```

经过重构，测试用例中的重复代码被搬迁到公共的工厂方法中，并进一步提高了代码的表达力。

```cpp
ASSERT_NE(nullptr, store.find(model_name("RestNet")));
ASSERT_NE(nullptr, store.find(model_name("GoogleNet")));
```

### 重复的工厂方法

> 需求4： 存在一个机器学习的模型列表，查找第一个版本号不为1，且名为ResNet的模型。

为了实现该需求，在既有的实现中，需要分两步走。第一步，重命名既有的`model_version`为`model_version_eq`；第二步，实现一个全新的工厂方法`model_version_ne`。

```cpp
ModelSpec model_version_eq(int version) {
  return [version](auto& model) {
    return model.version() == version;
  };
}

ModelSpec model_version_ne(int version) {
  return [version](auto& model) {
    return model.version() != version;
  };
}
```

这两个工厂方法存在结构化重复，为了消除两者之间的重复，需要分离新的变化方向。这个变化蕴含了比较和逻辑运算。

- 比较运算：==, !=
- 逻辑运算：&&, ||

#### 比较语义

提炼新的抽象`Matcher`，它表示与期望值匹配的抽象行为。

```cpp
template <typename Expected>
struct EqualMatcher {
  EqualMatcher(Expected expected) : expected(expected) {
  }
  
  template <typename Actual>
  bool operator()(const Actual& actual) const {
    return actual == expected;
  }
  
private:
  Expected expected;
};
```

#### 引入工厂

为了进一步提高表达力，可以引入工厂方法，简化`EqualMatcher`对象构造的复杂度。

```cpp
template <typename Expected>
auto equals_to(Expected expected) {
  return EqualMatcher<Expected>(expected);
}
```

```cpp
template <typename Matcher>
ModelSpec model_name(Matcher matcher) {
  return [matcher = std::move(matcher)](auto& model) {
    return matcher(model.name());
  };
}

template <typename Matcher>
ModelSpec model_version(Matcher matcher) {
  return [matcher = std::move(matcher)](auto& model) {
    return matcher(model.version());
  };
}
```

```cpp
ASSERT_NE(nullptr, store.find(model_name(equals_to("RestNet"))));
ASSERT_NE(nullptr, store.find(model_version(equals_to(2))));
```

#### 语法糖

```cpp
const EqualsMatcher<std::string> is_empty("");
const EqualsMatcher<const void*> is_nil(nullptr);
```


#### 修饰语义

```cpp
template <typename Expected>
struct NotEqualMatcher {
  NotEqualMatcher(Expected expected) : expected(expected) {
  }
  
  template <typename Actual>
  bool operator()(const Actual& actual) const {
    return actual != expected;
  }
  
private:
  Expected expected;
};
```

```cpp
template <typename Matcher>
struct NotMatcher {
  NotMatcher(Matcher matcher) : matcher(std::move(matcher)) {
  }
  
  template <typename Actual>
  bool operator()(const Actual& actual) const {
    return !matcher(actual);
  }
  
private:
  Matcher matcher;
};
```

#### 引入工厂

```cpp
template <typename Matcher>
auto is_not(Matcher matcher) {
  return NotMatcher<Matcher>(std::move(matcher));
}
```

```cpp
ASSERT_NE(nullptr, store.find(model_name(is_not(equals_to("RestNet")))));
```

#### 逻辑与

```cpp
template<typename ... Matchers>
struct AllMatcher;

template<typename Head, typename ... Tail>
struct AllMatcher<Head, Tail...> : AllMatcher<Tail...> {
  AllMatcher(Head head, Tail ... tail)
      : AllMatcher<Tail...>(std::move(tail)...)
      , head(std::move(head)) {
  }

  template<typename Actual>
  bool operator()(const Actual& actual) const {
    return head(actual) && 
           AllMatcher<Tail...>::operator()(actual);
  }

private:
  Head head;
};

template<>
struct AllMatcher<> {
  template<typename Actual>
  bool operator()(const Actual& actual) const {
    return true;
  }
};
```

```cpp
auto spec = all(
  model_version(is_not(equals_to(1))),
  model_name(equals_to("ResNet"))
);

ASSERT_NE(nullptr, store.find(spec));
```

#### 逻辑或

```cpp
template<typename ... Matchers>
struct AnyMatcher;

template<typename Head, typename ... Tail>
struct AnyMatcher<Head, Tail...> : AnyMatcher<Tail...> {
  AnyMatcher(Head head, Tail ... tail)
      : AnyMatcher<Tail...>(std::move(tail)...)
      , head(std::move(head)) {
  }

  template<typename Actual>
  bool operator()(const Actual& actual) const {
    return head(actual) ||
           AnyMatcher<Tail...>::operator()(actual);
  }

private:
  Head head;
};

template<>
struct AnyMatcher<> {
  template<typename Actual>
  bool operator()(const Actual& actual) const {
    return false;
  }
};
```

```cpp
auto spec = any(
  model_version(equals_to(1)),
  model_version(equals_to(2))
);

ASSERT_NE(nullptr, store.find(spec));
```

### 字符串算子

#### 原子字符串算子

```cpp
struct StartMatcher {
  StartMatcher(std::string prefix) : prefix(std::move(prefix)) {
  }

  bool operator()(const std::string& s) const {
    return s.length() >= prefix.length() &&
      s.compare(0, prefix.length(), prefix) == 0;
  }

private:
  std::string prefix;
};

struct EndMatcher {
  EndMatcher(const std::string &suffix) : suffix(suffix) {
  }

  bool operator()(const std::string &s) const {
    return s.length() >= suffix.length() &&
      s.compare(s.length() - suffix.length(), suffix.length(), suffix) == 0;
  }

private:
  std::string suffix;
};
```

#### 修饰字符串算子

```cpp
struct IgnoringCase {
  IgnoringCase(const std::string& expected)
    : matcher(strutils::to_lower(expected)) {
  }

  bool operator()(const std::string& actual) const {
    return matcher(strutils::to_lower(actual));
  }

private:
  Matcher matcher;
};
```

```cpp
auto ignoring_case_starts(const std::string& prefix) {
  return IgnoringCase<StartMatcher>(prefix);
}

auto ignoring_case_ends(const std::string& suffix) {
  return IgnoringCase<EndMatcher>(suffix);
}
```

### 占位符

```cpp
template<typename T, bool value>
struct Placeholder {
  bool operator()(const T&) const {
    return value;
  }
};

template <typename T>
using always = Placeholder<T, true>;

template <typename T>
using never = Placeholder<T, false>;
```


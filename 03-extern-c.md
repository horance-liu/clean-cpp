## Clean C++：正确使用`extern "C"`的姿势


`extern "C"`用于明确告诉`C++`编译器放弃名字粉碎的工作机制，使其保留原始的符号名称。

### 纯粹的C库

即使你提供的是一个纯粹的`C`库，也必须正确使用`extern "C"`，因为你的`C`函数可能被`C++`的程序所使用。

```cpp
// oss/include/oss_memery.h
#ifndef HF0916DFB_1CD1_4811_B82B_9B8EB1A007D8
#define HF0916DFB_1CD1_4811_B82B_9B8EB1A007D8

#include "oss_common.h"

#ifdef  __cplusplus
extern "C" {
#endif

void* oss_alloc(size_t);
void  oss_free(void*);

#ifdef  __cplusplus
}
#endif

#endif
```

如果在`C`库中遗漏``extern "C"``的声明，则`C++`程序员为了引用该库的符号时，唯一的选择就是使用`extern "C"`包含整个头文件，而这种模式恰好是一种反模式。因此，`C`程序员定义对外公开的头文件时，需要特别的注意这件事情。

```
#ifdef  __cplusplus
extern "C" {
#endif

#include "oss_memory.h"

#ifdef  __cplusplus
}
#endif
```

### 反模式

在`extern "C"`中包含头文件，是一种典型的反模式。如下示例代码，`oss_common.h`置于`extern "C"`之中，将导致其内部或间接通过`#include`引入的符号都置于`extern "C"`的作用域之内；更为严重的是，如果`oss_common.h`也使用了`extern "C"`，将导致嵌套的`extern "C"`语句，导致混乱的符号声明。

```cpp
// oss/include/oss_memery.h
#ifndef HF0916DFB_1CD1_4811_B82B_9B8EB1A007D8
#define HF0916DFB_1CD1_4811_B82B_9B8EB1A007D8

#ifdef  __cplusplus
extern "C" {
#endif

#include "oss_common.h"

void* oss_alloc(size_t);
void  oss_free(void*);

#ifdef  __cplusplus
}
#endif

#endif
```

### 实用宏

可以定义一组实用宏，简化`extern "C"`的作用域定义。在代码的可读性没有下降的前提下，代码从`6`行减少至`2`行。

```cpp
// cub/base/externc.h
#ifdef __cplusplus
# define EXTERN_STDC_BEGIN extern "C" {
# define EXTERN_STDC_END }
#else
# define EXTERN_STDC_BEGIN
# define EXTERN_STDC_END
#endif
```

例如，上例使用宏定义后的代码如下所示。

```cpp
// oss/include/oss_memery.h
#ifndef HF0916DFB_1CD1_4811_B82B_9B8EB1A007D8
#define HF0916DFB_1CD1_4811_B82B_9B8EB1A007D8

#include "cub/base/externc.h"
#include "oss_common.h"

EXTERN_STDC_BEGIN

void* oss_alloc(size_t);
void  oss_free(void*);

EXTERN_STDC_END

#endif
```


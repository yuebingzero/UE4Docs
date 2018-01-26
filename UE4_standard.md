## 命名规则
模板类 前缀 **T**  
继承自UObject **U**  
继承自AActor **A**  
继承自Swidget **S**
抽象借口 **I**  
Enums **E**  
Boolean变量 **b**  
其他类 **F**
TypeDef根据类型选择 **U** 或者 **F**

避免缩写

返回bool类型的函数需要是一个询问状态的名字，例如IsVisiable()  

没有返回值的函数（procedure），需要用一个目的明确的动词开始（一个动宾结构）。  

建议函数参数如果是引用类型，前缀标记为 **Out**  

## Portable Aliases for Basic C++ Types
* bool for boolean values (NEVER assume the size of bool). BOOL will not compile.
* TCHAR for a character (NEVER assume the size of TCHAR).
* int8 for signed bytes (1 byte).
* uint8 for unsigned bytes (1 byte).
* uint16 for unsigned "shorts" (2 bytes).
* int16 for signed "shorts" (2 bytes).
* uint32 for unsigned ints (4 bytes).
* int32 for signed ints (4 bytes).
* uint64 for unsigned "quad words" (8 bytes).
* int64 for signed "quad words" (8 bytes).
* float for single precision floating point (4 bytes).
* double for double precision floating point (8 bytes).
* PTRINT for an integer that may hold a pointer (NEVER assume the size of PTRINT).  

## 通用格式
很多个的条件判断，单独拿出来用临时变量表示  

继承类的虚函数如果需要override，在函数后记得标注overrides  

用FType\* P 而不是FType \*P

## UE4智能指针库
- Shared Pointer(TSharedPtr)
- Shared Reference(TSharedRef)
- Weak Pointer(TWeakPtr)

优点：
+ 当没有引用后，自动释放
+ 有线程安全的方法
+ 运行时安全 Shared Reference不会为空
+ 不会有引用循环，用Weak Pointer来避免循环引用
+ 能明确Object的Owner

所有的Shared类型(TSharedPtr, TSharedRef, TWeakPtr)（32bits指针）8bytes，包含
- c++ pointer(uint32)
- Reference controller pointer(uint32)  

Reference controller object 12bytes (32bits指针)
- c++ pointer(uint32)
- Shared reference count (uint32)
- Weak reference count (uint32)

Shared Reference是非空的Shared Pointer，任何时候Shared Reference不会为空，也就不需要IsValid()函数。如果可行的话，建议总是使用Shared Reference代替Shared Pointer。如果你需要空指针，那么用Shared Pointer。

TSharedRef转TSharedPtr隐式转换，TSharedPtr转TSharedRef需要显示转换ToSharedRef()。

Weak Pointer的引用不会阻止object的释放。Weak Pointer会自动置为空，当Object释放之后。可以用Weak Pointer来确定Object有没有被销毁。   

## Actor
### Spawn Actor
### Components  
actor相当于包含多个components的容器  
actor的Transfrom取决于 root component  
### Ticking
### LifeCycle
![ActorLifeCycleImg](image/ActorLifeCycle1.jpg)  

### Destroy
actor并不会自动垃圾回收，需要调用Destroy()函数

# Chapter2 Allocator
## 标准接口

2.2 SGI特殊的空间配置器 std::alloc
C++内存配置操作和释放操作是这样的：
```
class Foo{};
Foo *fo = new Foo;
delete fo;
```
new包含两个阶段的操作：（1）调用::operator new配置内存；（2）调用Foo::Foo()构造对象内容；  
delete也包含两个阶段的操作：（1）调用Foo::~Foo()将对象析构；（2）调用::operator delete释放内存。  

STL allocator决定将这两个阶段操作区分开，内存配置操作由alloc::allocate()负责，内存释放由alloc::deallocate()负责；对象构造由::construct()负责，对象析构由::destroy()负责。 

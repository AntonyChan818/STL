# Chapter2 Allocator
## 标准接口

2.2 SGI特殊的空间配置器 std::alloc
C++内存配置操作和释放操作是这样的：
```
class Foo{};
Foo *fo = new Foo;
delete fo;
```
2.2.1
new包含两个阶段的操作：（1）调用::operator new配置内存；（2）调用Foo::Foo()构造对象内容；  
delete也包含两个阶段的操作：（1）调用Foo::~Foo()将对象析构；（2）调用::operator delete释放内存。  
STL allocator决定将这两个阶段操作区分开，内存配置操作由alloc::allocate()负责，内存释放由alloc::deallocate()负责；对象构造由::construct()负责，对象析构由::destroy()负责。 

2.2.2
STL将配置器定义于<memory>之中，SGI<memory>中包含两个文件：
  ```
  #include <stl_alloc.h> //负责内存空间的配置和释放
  #include <stl_construct.h> //负责对象内容的构造和析构
  ```
  ![](https://github.com/AntonyChan818/STL/blob/master/image/img2_1.png)

2.2.3 construct()/destroy()  
![](https://github.com/AntonyChan818/STL/blob/master/image/img2_2.png)  

trivial destructor: 如果用户不定义析构函数，而是系统自带的，则说明析构函数基本没什么用（会被默认调用），称之为trivial destructor。  
non-trivial destructor：如果特定定义了析构函数，则说明在释放空间之前需要做一些事情，则这个析构函数称为non-trivial destructor。  

construct()接受一个指针p和一个初值value，函数的用途是将初值设定到指针所指的空间上。C++的placement new运算子可用来完成这个任务。  
```
template <class T1, class T2>
inline void construct(T1 *p, const T2& value){
  new (p) T1(value);  //placement new; 调用T1::T1(value)
}
```  

destroy()有两个版本，第一个版本接受一个指针，直接调用该对象的析构函数即可。
```
template <class T>
inline void destroy(T* pointer) {
  pointer->~T();
}
```

第二个版本，接受两个迭代器，准备将[first，last]范围内的所有对象析构掉。我们不知道这个范围有多大，万一很大，而每个对象的析构函数都无关痛痒，那么一次次调用这些析构函数，对效率是一种伤害，因此这里首先利用value_type()获得迭代器所指对象的类别，再利用_type_traits<T>判断该型别的析构函数是否无关痛痒，若是(_true_type)，则什么也不做就结束，若否(_false_type),这才以循环的方式巡访整个范围，并在循环中每经历一个对象就调用第一个版本的destory()。
  

# Chapter2 Allocator
## 标准接口

**2.2 SGI特殊的空间配置器 std::alloc**
C++内存配置操作和释放操作是这样的：
```
class Foo{};
Foo *fo = new Foo;
delete fo;
```
**2.2.1**
new包含两个阶段的操作：（1）调用::operator new配置内存；（2）调用Foo::Foo()构造对象内容；  
delete也包含两个阶段的操作：（1）调用Foo::~Foo()将对象析构；（2）调用::operator delete释放内存。  
STL allocator决定将这两个阶段操作区分开，内存配置操作由alloc::allocate()负责，内存释放由alloc::deallocate()负责；对象构造由::construct()负责，对象析构由::destroy()负责。 

**2.2.2**
STL将配置器定义于<memory>之中，SGI<memory>中包含两个文件：
  ```
  #include <stl_alloc.h> //负责内存空间的配置和释放
  #include <stl_construct.h> //负责对象内容的构造和析构
  ```
  ![](https://github.com/AntonyChan818/STL/blob/master/image/img2_1.png)

**2.2.3 construct()/destroy()构造与析构**  
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
  
**2.2.4 空间的配置与释放 std::alloc**  
对象构造前要进行空间配置，对象析构后要进行空间释放，由<stl_alloc.h>负责：  
- 向system heap要求空间
- 考虑多线程的状态
- 考虑内存不足的应变措施
- 考虑过多小型区块可能造成的内存碎片问题  

C++内存配置的基本操作是 ```::operator new()``` ，内存释放的基本操作是 ```::operator delete()```， C++的allocator只是对new和delete做简单的包装，这样效率很低，如频繁请求小块内存有可能会导致内存内部碎片的产生(由于内存对齐等原因)。或者有一些内置类型无需构造或析构操作(type_traits致力于解决此问题）。  
这两个全局函数相当于C的```malloc(), free()```函数。SGI STL正是以```malloc(), free()```完成内存的配置和释放。考虑到内存破碎的问题，SGI设计了双层级配置器：
- 第一级配置器直接使用```malloc(), free()```
- 第二级配置器视情况采用不同的策略：
  - 当配置区块大于128bytes时，调用第一级配置器；
  - 当配置区块小于128bytes时，为了降低负担，采用memory pool整理方式。  

使用哪一种配置器取决于预处理命令__USE_MALLOC:
```
#ifdef __USE_MALLOC
typedef __malloc_alloc_template<0> malloc_alloc;
typedef malloc_alloc alloc; // alloc为第一级配置器
#else
typedef __default_alloc_template<__NODE_ALLOCATOR_THREADS, 0> alloc;
```  
而SGI还为这些配置器包装一个专门的接口：  
```
template<class T, class Alloc>
class simple_alloc{
public:
  static T *allocate(size_t n){
    return 0 == n? 0 : (T*)Alloc::allocate(n*sizeof(T));
  }
  static T *allocate(void){
    return (T*) Alloc::allocate(sizeof(T));
  }
  static void deallocate(T *p, size_t n){
    if (0 != n ) Alloc::deallocate(p, n*sizeof(T));
  }
  static void deallocate(T *p){
    Alloc::deallocate(p, sizeof(T));
  }
};
```  
STL的所有容器都使用这个simple_alloc接口：
举例子：
```
template <class T, class Alloc = alloc>
class vector{
protected:
  typedef simple_alloc<value_type, Alloc> data_allocator;
  void deallocate(){
    if(...)
      data_allocator::deallocate(start, end_of_storage - start);
  }
};
```  

一二级配置器的关系和接口包装运用方式如图：  
![](https://github.com/AntonyChan818/STL/blob/master/image/img2_3.png)  

**2.2.5 第一级配置器__malloc_alloc_template剖析**  
```
template<int, inst>
class __malloc_alloc_template{
private:
  //空指针和void*类型指针
  //空指针：#define NULL ((void*)0)， NULL实际上是((void*)0)， void*类型是用来存放地址的，所以0为地址，而内存分配中，较小的地址
  //        是不用来存放数据也不允许程序访问，所以指针指向它，就是这个指针不能操作它指向的较小的地址。
  //void*：这个类型的指针指向了存放数据的地址，只是改地址存放的数据的数据类型暂时不知道。强转来存放我们想存放的类型：
  //        char* str = (char*)malloc(sizeof(char*)13);
  
  //out of memory
  static void *oom_malloc(size_t); 
  static void *oom_realloc(void*, size_t);
  static void (* __malloc_alloc_oom_handler)();
  
 public:
   static void *allocate(size_t n){
     void *result = malloc(n); //第一级配置器直接用malloc()
     if(0 == result)
       result = oom_alloc(n);
       return result;
   }
   
   static void *deallocate(void *p, size_t n){
     free(p);
   }
   
   static void *reallocate(void *p, size_t old_sz, size_t new_sz){
     void *result = realloc(p, new_sz);
     if(0 == result)
       result = oom_realloc(p, new_sz);
     return result;
   }
   
   //set_malloc_handle是一个函数，函数参数为void(*f)()这个参数是一个函数指针，参数为空，返回值也为空； 函数名前有*,
   //所以返回一个指针；这个指针也有参数，因此指针指向一个函数，这个函数参数为空，返回类型为void。
   static void (* set_malloc_handle(void (*f)()))(){
     void (* old)() = _malloc_alloc_oom_handler;
     __malloc_alloc_oom_handler = f;
     return(old);
   }
}

```  
- 函数指针作为返回值：
```
using F = int(int*, int); //F为函数类型  函数返回int， 参数为int*,int
using PF = int(*)(int*, int) //PF为函数指针类型
PF f1(int);  //返回函数指针。
F* f2(int);  //返回函数指针。
//f1等同于 
int (*f1(int)) (int*, int);
//f1有形参列表，是一个函数;前面有一个*,所以f1返回一个指针；进一步观察，指针类型本身也包含形参列表，|
//因此指针指向函数，该函数(就是指针指向的函数)的返回类型为int.
```  
第一级配置器不能直接使用C++ new-handler的机制，因为并非使用::operator new来配置内存。因此需要仿真一个类似的set_malloc_handler()来解决内存不足时候的操作。  

**2.2.6 第二级配置器 __default_alloc_template**  
第二级配置器添加机制避免太多小额区块造成内存的碎片和overhead，overhead多出来的空间用来管理内存，但区块越小，overhead占的比例就越大。
![](https://github.com/AntonyChan818/STL/blob/master/image/img_2.2.6.png)  
第二级配置器的做法：  
- 如果区块>128bytes，就移交第一级配置器处理  
- 如果区块<128bytes，用内存池(memory pool/sub-allocation)管理：每次配置一大块内存，并维护对应之自由链表(free-list)。下次如果再有相同大小的内存需求，就直接从free-lists中取出。如果释放小额区块，就由配置器回收到free-lists中。第二级配置器会主动将任何小额区块的内存需求量上调至8的倍数并维护16个free-lists，各自管理大小分别为8,16,24,32,40,48,56,64,72,80,88，96,104,112,120,128bytes的小额区块。  
free-lists的节点结构：  
```
unioin obj{
  union obj *free_list_link;
  char client_data[1];
}
```  
![](https://github.com/AntonyChan818/STL/blob/master/image/img_2.2.6_2.png)  



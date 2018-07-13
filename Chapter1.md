# Chapter1
## **STL六大组件**  
1. 容器：vector、list、deque、set、map，用来存放数据。是一种class template  
2. 算法：如sort、search、copy、erase。是一种function template
3. 迭代器：容器和算法之间的胶合剂，泛型指针容器都有自己专属的迭代器。原生指针也是一种迭代器。
4. 仿函数（functors）：行为类似函数，可以作为算法的某种策略，是一种重载了operator()的class或class template，函数指针是一种。
5. 配接器（adapter）: 修饰容器或仿函数或迭代器接口的东西。如queue和stack。
6. 配置器（allocators）：负责空间配置和管理。实现了动态空间配置、空间管理和空间释放的class template。
![](https://raw.githubusercontent.com/AntonyChan818/STL/master/image/img1_1.png)

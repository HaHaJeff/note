#C++ 11 standard

##关于std::remove_if的一个bug(准确来说不应该是bug，而是不理解c++标准对于Predicate的定义)
``` cpp
class Nth {
  private:
    int nth;
    int count;
  public:
    Nth (int n) : nth(n), count(0) {

    }

    bool operator() (int) {
      return ++count == nth;
    }
};

int main()
{
  list<int> coll = { 1, 2, 3, 4, 5, 6, 7, 8, 9, 10 };
  PRINT_ELEMENTS(coll, "coll:   ");
  list<int>::iterator pos;
  pos = remove_if(coll.begin(), coll.end(), Nth(3));
  coll.erase(pos, coll.end());
  PRINT_ELEMENTS(coll, "coll:   ");
}
```
**Result**
``` bash
coll:   1 2 3 4 5 6 7 8 9 10 
coll:   1 2 4 5 7 8 9 10 
```

**Reason**
```
  //参数列表对应的__pred即为Nth(3)，为值传递
  template<typename _ForwardIterator, typename _Predicate>
    _ForwardIterator
    __remove_if(_ForwardIterator __first, _ForwardIterator __last,
		_Predicate __pred)
    {
      __first = std::__find_if(__first, __last, __pred);
      if (__first == __last)
        return __first;
      _ForwardIterator __result = __first;
      ++__first;
      for (; __first != __last; ++__first)
        if (!__pred(__first))
          {
            *__result = _GLIBCXX_MOVE(*__first);
            ++__result;
          }
      return __result;
    }

  //对于find_if而言同样为值传递
  template<typename _Iterator, typename _Predicate>
    inline _Iterator
    __find_if(_Iterator __first, _Iterator __last, _Predicate __pred)
    {
      return __find_if(__first, __last, __pred,
		       std::__iterator_category(__first));
   
```
**remove_if有两次关于__pred的调用，第一次调用为find_if操作找到3，然后再for循环中再次对__pred进行调用，对应Nth(3)，由于find_if是值传递，所以count没有发生变化，于是再次找到6**

##c++11新特性
- Template表达式内的空格
```
vector<list<int> > //OK in each c++ version
vector<list<int>>  //OK since c++11
```

- nullptr
c++11使用nullptr取代0或NULL，用来表示一个pointer(指针)指向所谓的novalue。
nullptr是个新关键字。它被自动转换为各种pointer类型，但不会转换为任何整数类型。

- 以auto完成类型的自动推导
通过auto关键字自动完成类型的推导，以auto声明的变量，其类型会根据其初值被自动推导出来，因此一定需要一个初始化操作：
```
	auto i;   //ERROR:无法推导出i的类型
```

- 一致性初始化与初值列
···
int values[] {1, 2, 3};
std::vector<int> v{2,3,4,5};
std::vector<std::string> cities {
"a", "b", "c"
};
std::complex<double> c{4.6, 3.0};
···
**初值列**会强迫造成所谓的**value initialization**，意思是某个local变量属于某种基础类型(通常不会被初始化)也会被初始化为0(nullptr):
```
int i;    // i undefined value
int j{};  // j is initialized by 0
int *p;   // p undefined value
int *q{}; // q is initialized by nullptr
```
**窄化对大括号不成立：**
```
int x1(1.1)   // OK x1 become 1
int x2{1.1}   // ERROR
char c1{7};   // OK even though 7 is an int, 但是不会发生窄化
char c2{277}; // ERROR 发生了窄化 因为277超过char的最大值
```
**std::initializer_list<int> value**
```
void Print(std::initializer_list<int> value) {
	for (auto p = value.begin(); p != value.end(); ++p) {
		std::cout << *p << "\n";
	}
}
```

- Range-Based for循环
```
for (int i: {1,2,3,4,5}) {
	std::cout << i << std::endl;
}
```

- Move语义和Rvalue Reference
该特性的主要目标：**避免非必要拷贝和临时对象**
```
void CreateAndInsert(std::set<X>& coll)
{
	X x; 
	...
	coll.insert(x);   //1
	coll.insert(x+x); //2
	coll.insert(x);   //3
}
```
1. 对于注释1而言，使用copy插入是有意义的，因为x对象在以后用到了；
2. 对于注释2而言，**x+x** 产生一个临时对象，此时使用copy构造会产生额外的开销；
3. 对于注释3而言，**x** 这个局部对象已经无用了，所以使用copy构造也会产生额外的开销0。

- 新式的字符串字面常量

**Raw String Literal**
作用：可以直接使用特殊字符
例如：在c++11之前想打印\w\\w
```
string s = "\\w\\\\w";
```
在c++11之后，可以使用Raw String
```
string s = R"(\w\\w)"
```

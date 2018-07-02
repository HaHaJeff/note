# C++ 11 standard

## 关于std::remove_if的一个bug(准确来说不应该是bug，而是不理解c++标准对于Predicate的定义)
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
**Result **
``` bash
coll:   1 2 3 4 5 6 7 8 9 10 
coll:   1 2 4 5 7 8 9 10 
```

**Reason **
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

## c++11新特性
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

```
int values[] {1, 2, 3};
std::vector<int> v{2,3,4,5};
std::vector<std::string> cities {
"a", "b", "c"
};
std::complex<double> c{4.6, 3.0};
```

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
- 
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

- 关键字 constexpr

自c++11起，constexpr可用来让表达式核定于编译器。例如：
```
constexpr int square(int x)
{
	return x * x;
}
float a[square(9)];
```

- 崭新的Tempate特性
自c++11起，template可以拥有"得以接受个数不定的template实参"的参数。

```
void print()
{
}

template <typename T, typename... Types>
void print(const T& firstArg, const Types&... args)
{
	std::cout << firstArg << std::endl;
	print(args);
}
```
**Alias Template(带别名的模板，或者叫Template Typedef)**
自c++11起，支持template type definition。然而由于关键字typename用于此处时总是处于某种原因而失败，所以引入关键字using：

```
template <typename T>
using Vec = std::vector<T, MyAlloc<T>>;
Vec<int> coll
```
上述等价于：

```
std::vector<int, MyAlloc<int>> coll
```

- lambda
语法：

```
[] {
	std::cout << "hello lambda" << std::endl;
};
```

直接调用：

```
[] {
	std::cout << "hello lambad" << std::endl; 
}();
```
或是将它传递给对象：

```
auto l = [] {
	std::cout << "hello lambad" << std::endl; 
};
l();
```

使用参数的lambda：
```
auto l = [](const std::string& str) {
	std::cout << str << std::endl; 
};
l("hello");
```

**值得注意的是：lambda不可以是template**

使用返回类型的lambda：

```
[]()->double {
	return 43.0;
}
```
**double不一定需要指定，该返回类型会根据返回值自动被推导出来**

访问外部作用域：
1. [=]意味着外部作用域以by value方式传递给lambda。
2. [&]意味着外部作用域以by reference方式传递给lambda。

```
int x = 0;
int y = 42;
auto q = [x, &y] {
	std::cout << "x" << x << std::endl;
	std::cout << "y" << y << std::endl;
	++y; //OK
};
x = y = 77;
q();
q();
std::cout << "final y: " << y << std::endl;
```

输出：
```
//第一次q()
x: 0
y: 77
//第二次q()
x: 0
y: 78
final y: 79
```

**由于x是by value而获得一份拷贝，在lambda中你可以读取它，但是++x是不被允许的。y以by reference传递，所以你可以对y进行改写**

by value和by reference的混合体

```
int id = 0;
auto f = [id]()mutable{
	std::cout << "id: " << id << std::endl;
	++id; //OK
};

id = 42;
f();
f();
f();
std::cotu << id << std::endl;
```
输出：
```
//第一次调用f()
id: 0
//第二次调用f()
id: 1
//第三次调用f()
id: 2
42
```
上述功能可以理解为：
```
class {
private:
	int id;
public:
	void operator() () {
		std::cout << "id: " << id << std::endl;
		++id; //OK
	}
};
```

**lambda的类型**

lambda的类型，是个不具名function object。每个lambda表达式的类型是独一无二的。因此，如果想要根据该类型声明对象，可借助于template或auto。如果需要以该类型作为函数参数，则可以使用decltype关键字，例如把一个lambda作为hash function或ordering准则或sorting准则传给容器。

```
std::function<int(int, int)>> returnLambda() {
	return [](int x, int y) {
		return x*y;
	};
}
int main()
{
	auto lf = returnLambda();
	std::cout << lf(6, 7) << std::endl;
}
```

- 关键字decltype

关键字decltype可以让编译器找出表达式类型，这其实就是typeof的特性体现：
```
std::map<std::string, float> coll;
decltype(coll)::value_type elem;
```

- 新的函数声明语法
有时候，函数的返回类型取决于某个表达式对实参的处理。然而类似：

```
template <typename T1, typename T2>
decltype(x+y) add(T1 x, T2 y)
```
在c++11之前是不可能的，因为返回式所使用的对象尚未被引用，但是在c++11：
```
template <typename T1, typename T2>
auto add(T1 x, T2 y) -> decltype(x+y)
```

- 新的基础类型

1. char16_t和char32_t;
2. long long和unsigned int;
3. std::nullptr_t;

# 随旧犹新的语言提特性

- Nontype Template Parameter(非类型模板参数)
```
bitset<32> flags32;
bitset<50> flags50
```

- Default Tempate Parameter
```
template <typename T, typename container = vector<T>>
class MyClass;

//可以只传入一个参数
MyClass<int> x1;
```

-Member Tempate
Class的成员函数可以是template，然后不能是virtual函数(template在编译时转换，virtual运行时绑定，矛盾)
```
tempalte<T>
class MyClass {

	template <typebane T>
	void f(T);
};
```

**考虑下面一个例子**

```
template <typename T>
clss MyClass {
public:
	T value;
public:
	void assign(const MyClass<T>& x) {
		value = x.value;
	}
};

void f()
{
	MyClass<double> d;
	MyClass<int>    i;
	d.assign(d); //OK
	d.assign(i); //ERROR  参数类型不匹配
}
```

**解决方案**

```
template <typename T>
clss MyClass {
public:
	T value;
public:
	template<typename X>
	void assign(const MyClass<X>& x) {
		value = x.value;
	}
};

void f()
{
	MyClass<double> d;
	MyClass<int>    i;
	d.assign(d); //OK
	d.assign(i); //OK  double可以转成int
}
```

- 基础类型的明确初始化

如果使用"一个明确的构造函数调用，但不给实参"这样的语法，基础类型会被设定初值为0：
```
int i1;   		//undefined value
int i2 = int(); //initialized with zero
int i3{};		//initialized with zero(c++11)
```
这个特性使得我们可以写出"确保无论任何类型，其值都有一个确凿的默认值"的template code。例如下面的函数中，初始化机制保证了"x如果是基础类型，会被初始化为0"：
```
template <typename T>
void f()
{
	T x = T();
}
```
该语法叫做：zero initialized


# Smart Pointer

- shared_ptr
shared_ptr的构造函数是explicit
```
std::shared_ptr<string> p = new string("asd"); //ERROR
std::shared_ptr<string> p{ new string("asd")}; //OK
```

**需要注意的是shared_ptr不支持指向数组**

```
std::shared_ptr<int> p(new int[10]);  //OK when compile, but error in deconstruct, memory lack
//因为shared_ptr的默认deleter是 delete，所以无法完成数组的析构

std::shared_ptr<int> p(new int[10], [](int *p){delete []p;}); //OK in compile and deconstruct
//但是通过shared_ptr访问数组去不方便，因为shared_ptr没有重载[]
//访问数组：p.get()[index]
std::shared_ptr<int[]> p(new int[10]) //ERROR 因为shared_ptr没有数组类型模板参数

std::unique_ptr<int[]> s(new int[10]) // OK unique_ptr支持数组类型，且unique_ptr的deleter支持[]，并且unique_ptr对于[]类型还重载了[]操作符；
```

**shared_ptr只提供operator*和operator->，指针运算和operator[]都未提供，因此，如果想访问内存，必须使用get()获得被shared_ptr报过的内部指针**

- class weak_ptr
错误的使用shared_ptr会发生资源泄露：
1. cyclic reference(环式指向)，如果两对象使用shared_ptr互相指向对方，即使不存在其他reference指向它们时，资源也不会释放；
通过class weak_ptr，可以避免环式指向的发生
**weak_ptr的语义：共享但不拥有，需要注意的是：weak_ptr不提供operator*和operator->**


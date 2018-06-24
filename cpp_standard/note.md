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
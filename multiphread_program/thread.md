## 多线程编程的误区

- 在程序终止之前，没有等待线程结束，这将会导致程序崩溃
```
#include <iostream>
#include <thread>
using namespace std;
void LaunchRocket()
{
    cout << "Launching Rocket" << endl;
}
int main()
{
    thread t1(LaunchRocket);
    //t1.join(); // somehow we forgot to join this to main thread - will cause a crash.
    return 0;
}
```
为什么会崩溃？因为线程的析构函数会被调用，如果发现线程可以join的，则调用std::terminate()，该行为导致的默认操作是程序crash
```
~thread() _NOEXCEPT
{  // clean up
  if (joinable())
    _XSTD terminate();
}
```

- 试图join一个已经detached的线程
```
#include <iostream>
#include <thread>
using namespace std;
void LaunchRocket()
{
    cout << "Launching Rocket" << endl;
}
int main()
{
    thread t1(LaunchRocket);
    t1.detach();
    //..... 100 lines of code
    t1.join(); // CRASH !!!
    return 0;
}
```
解决方案：
```
int main()
{
  thread t1(LaunchRocket);
  t1.detach();
  //..... 100 lines of code
  
  if (t1.joinable())
  {
    t1.join(); 
  }
  
  return 0;
```

- 没有意识到join是个阻塞操作

- 认为线程函数的参数默认是引用传递

- 没有共享数据或共享资源进行临界区保护

- 没有对锁进行释放

- 没有使临界区"小巧精致"

- 没有按照同样的顺序对多个锁进行加锁操作

- 试图对同一个mutex进行两次加锁操作

- 当使用std::atomic可以完成任务的时候，还在使用mutex

- 对线程的创建销毁太过频繁，却没有考虑使用线程池

- 没有在线程中处理异常

- 使用线程仿真std::async可以完成的异步任务


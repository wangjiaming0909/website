<link rel="stylesheet" type="text/css" href="../css/style.css">  

# Exception Handling: A False Sense of Security -- Tom Cargill 1994

# Exceptional C++

## Item 8. Writing Exception-Safe Code -- Part 1

继 `Tom Cargill` 1994年的critique [Exception Handling: A False Sense of Security](#exception-handling-a-false-sense-of-security----tom-cargill-1994)

Common questions:  

- What are the different "level" of exception safety?
- Can or should generic containers be fully exception-neutral?
- Are the standard library containers exception-safe or exception-neutral?
- Does exception safety affect the design of your container's public interface?
- Should generic containers use exception specifications?

The Stack in `Tom cargill`'s critique:  
```c++
template <class T> class Stack {
public:
    Stack();
    ~Stack();
    /*...*/
private:
    T* v_; // ptr to a memory area big
    size_t vsize_; // enough for 'vsize_' T's
    size_t vused_; // # of T's actually in use
};
```

Your mission:  

- make Stack exception-safe and exception-neutral
- Stack objects should always be in a correct and consistent state, regardless of any exceptions that might be thrown in the course of executing Stack's member functions.
- If any exceptions are thrown, they should be propagated seamlessly through to the caller.

`default constructor`  

```c++
//Is this safe?
//What about exception-safe and exception-neutral?
template<class T>
Stack<T>::Stack()
    : v_(0),
    vsize_(10),
    vused_(0) // nothing used yet
{
    //只有这里会抛出异常.
    //1. new 表达式分配内存失败抛出std::bad_alloc异常
    //2. T的构造函数任何异常
    //无论是1还是2,编译器保证了任何已经构造好的对象会被析构,已经分配的内存会被operator delete[]()释放.
    //因此这是exception-safe
    //这里的任何异常都会抛出,因此也是exception-neutral的
    //当然如果在构造某个非第一个元素的时候抛出了异常,那么在调用T的析构函数如果又抛出了异常,那么程序会直接调用terminate()结束.
    v_ = new T[vsize_]; // initial allocation
}
```

<DEF>GuideLine: </DEF> <BIG>If a function isn't going to handle (or translate or deliberately absorb) an exception, it should allow the exception to propagate up to a caller who can
handle it.</BIG>

<DEF>GuideLine: </DEF> <BIG>Always structure your code so that resources are correctly freed and data is in a consistent state even in the presence of exceptions.</BIG>

```c++
//作者更推荐这样写
//initializing members in initializer lists whenever possible
template<class T> Stack<T>::Stack()
: v_(new T[10]), // default allocation
vsize_(10),
vused_(0) // nothing used yet
{ }
```

`destruction`  

```c++
template<class T> Stack<T>::~Stack()
{
    //标准库的opreator delete[]()肯定是throw()或者noexcept的
    //但是若有人替换了全局的operator delete[](),那么也应当保证不抛出异常
    //还有可能是T::~T()有异常, 不过这里假设析构不会有异常
    //书上说T如果不能保证析构没有异常,那么Stack也就不能保证`exception-safe`
    delete[] v_; // this can't throw
}
```

```c++
void operator delete[]( void* ) throw();
void operator delete[]( void*, size_t ) throw();
```

<DEF>GuideLine: </DEF><BIG>Observe the canonical exception safety rules: Never allow an exception to escape from a destructor or from an overloaded operator delete() or
operator delete[](); write every destructor and deallocation function as though it had an exception specification of "throw()". More on this as we go on; this is an important theme.</BIG>

****

## Item 9. Writing Exception-Safe Code -- Part 2

The operator assignment and copy constructor of Stack<T>.  

```c++
template <class T>
Class Stack {
public:
    Stack();
    ~Stack();
    Stack(const Stack &);
    Stack& operator=(const Stack&);
private:
    T* v_; // ptr to a memory area big enough for 'vsize_' T's of T's actually in use
    size_t vsize_;
    size_t vused_;
};
```

Two principles need to obey:  

- `exception-safe` (work properly in the presence of exceptions)
- `exception-neutral` (propagate all exceptions to the caller, without causing integrity problems in a Stack object)

A new helper Function:

```c++
template <class T>
T* NewCopy(const T* src, size_t srcsize, size_t destsize) {
    assert(destsize >= srcsize);
    //如果这里抛出异常, 无论是new表达式的std::bad_alloc或者T的构造函数抛出的任何其他异常,所有已分配内存都会自动释放.
    //并且异常会出去,因此这里保持了`leak-free` and `exception-neutral`
    T* dest = new T[destsize];
    try {
        //这里会调用T::operator=(), 如果此函数抛出异常, 那么异常被catch住,释放了所有内存.
        //并且异常也会出去, 因此这里也保持了`leak-free` and `exception-neutral`
        //唯一一点就是当实现T::operator=()的时候,如果抛出了异常,一定要保证此时的T依然是`destructable`的.
        std::copy(src, src+srcsize, dest);
    } catch (...) {
        //析构所有T对象,释放内存
        delete[] dest;
        throw;
    }
    return dest;
}
```

有了NewCopy, copy constructor和operator=()就容易了:  

``` c++
template <class T>
Stack<T>& Stack<T>::operator=(const Stack<T>& other) {
    if (this != &other) {
        //This is the only line that could throw exception
        //If NewCopy throw a exception, it's propagated without affecting the Stack object's state.
        //To the caller, if the assignment throws then the state is unchanged, no memory is leaked.
        T* v_new = NewCopy(other.v_, other.vsize_, other.vsize_);
        delete[] v_; //this can't throw
        v_ = v_new; //take ownnership
        vsize_ = other.vsize_;
        vused_ = other.vused_;
    }
    return *this;
}
```

<DEF>GuideLine: </DEF><BIG>Observe the canonical exception-safety rules: In each function, take all the code that might emit an exception and do all that work safely off to the side. Only then, when you know that the real work has succeeded, should you modify the program state(and clean up) using only nonthorwing operations</BIG>

****

## Item 10. Writing Exception-Safe Code—Part 3

其他三个接口, count(), push(), pop()

```c++
template <class T> class Stack {
public:
    Stack();
    ~Stack();
    Stack(const Stack&);
    Stack& operator=(const Stack&);
    size_t Count() const;
    void Push(const T&);
    T Pop(); // if empty, throws exception
private:
    T* v_; // ptr to a memory area big
    size_t vsize_; // enough for 'vsize_' T's
    size_t vused_; // # of T's actually in use
};
```

```c++
template<class T> size_t Stack<T>::Count() const {
    return vused_; // safe, builtins don't throw
}
```

```c++
template<class T>
void Stack<T>::Push( const T& t ) {
    if( vused_ == vsize_ ) // grow if necessary
    {                      // by some grow factor
        size_t vsize_new = vsize_*2+1;
        //如果NewCopy抛出异常,那么Stack状态没有任何变更,并且没有内存泄漏,并且异常会出去.
        T* v_new = NewCopy( v_, vsize_, vsize_new );
        delete[] v_; // this can't throw
        v_ = v_new;  // take ownership
        vsize_ = vsize_new;
    }
    //可能有异常, 因此先进行赋值,再增大vused_,异常不会更改对Stack状态有变更.
    v_[vused_] = t;
    ++vused_;
}
```

The initial attempt for `T::pop()`  

```c++
// Hmmm... how safe is it really?
template<class T> T Stack<T>::Pop() {
    if( vused_ == 0) {
        throw "pop from empty stack";
    } else {
        //copy constructor could throw
        //不过异常会出去,对当前Stack状态没有变更,没有问题
        T result = v_[vused_-1];
        //修改大小
        --vused_;
        //复制构造可能有异常, 但是这个T对象已经被pop了,如果出现异常,那么这个对象就丢了.
        //当然编译器可能会进行返回值优化,只进行一次拷贝: "zero or once copies"
        return result;
    }
}
```

```c++
//考虑如下使用Stack的代码
//如果T的复制构造除了异常,那么对象丢了
string s(s.Pop());
string s2;
//如果T的operator=()有异常,那么对象也丢了.
s2 = s.Pop();
```

因此 `mutator functions should not return T objects by value`  
<DEF>GuideLine: </DEF><BIG>Prefer cohesion. Always endeavor to give each piece of code—each module, each class, each function—a single, well-defined responsibility</BIG>

```c++
template<class T> T& Stack<T>::Top() {
    if( vused_ == 0) {
        throw "empty stack";
    }
    return v_[vused_-1];
}
template<class T> void Stack<T>::Pop() {
    if( vused_ == 0) {
        throw "pop from empty stack";
    } else {
        --vused_;
    }
}
```

## Item 11. Writing Exception-Safe Code—Part 4

Now that we have implemented an `exception-safe` and `exception-neutual` Stack<T>, answer these questions:  

1. What are the important exception-safety guarantees?

1. For the Stack<T> that was just implemented, what are the requirements on T, the contained type?

The guarentees of exception safety:  

1. Basic guarantee: Even in the presence of exceptions thrown by T or other exceptions, Stack objects don't leak resources.  
This implies that the container will be destructible and usable even if an exception is thrown while performing some container operation.  
However, if an exception is thrown, the container will be in a consistent, but not necessarily predictable, state.

1. Strong guarantee: 


# Effective C++

## Item 8 Prevent exceptions from leaving destructors

书上说一个vector内有10个元素，如果在析构第一个元素的时候抛出了异常，那么其他9个也应该被销毁.  
也就是说其他9个元素的析构也应该被调用.  

然而我在ubuntu16, 使用g++ 5.4和g++ 8.1.0在一个vector内每个元素析构都会抛出异常时发现, 只有第一个元素被调用了析构,  
其他所有元素的析构都没有被调用, 这将导致非常严重的内存泄露, 如代码:  
```c++
#include <vector>
#include <string>
#include <iostream>
#include <exception>
using namespace std;

struct B{
  ~B() {
    cout << "destructor called for B" << endl;
  }
  int a;
  std::string s;
};
struct A{
  static int destruct_times;
  A(const char* s) : s(s), p(new B) {
    p->a = 1;
    p->s = s;
  }
  A(const A&a) {
    cout << "copy constructor called: " << s << endl;
    s = a.s;
    // deep copy
    p = new B();
    p->a = a.p->a;
    p->s = a.p->s;
  }
  ~A() {
    destruct_times++;
    delete p;
    cout << "destructor called: [" << s << "]" << destruct_times << endl;
    throw std::exception();
  }
  std::string s;
  B *p = NULL;
};
int A::destruct_times = 0;

int main()
{
  //test_boost_exception();
  std::vector<A> *v = new std::vector<A>();
  v->reserve(100);
  // ignore this mem leak
  A *a=new A("a"), *b = new A("b"), *c = new A("c");
  v->push_back(*a);
  v->push_back(*b);
  v->push_back(*c);
  try {
    delete v;
  } catch (...) {
    cout << "catch a exception" << endl;
  }
  return 0;
}
```
``` bash
+/tmp $ g++ --version
g++ (GCC) 8.1.0
Copyright (C) 2018 Free Software Foundation, Inc.
This is free software; see the source for copying conditions.  There is NO
warranty; not even for MERCHANTABILITY or FITNESS FOR A PARTICULAR PURPOSE.

+/tmp $ g++ -std=c++98 -ggdb -O0 ./1.cc
+/tmp $ ./a.out
copy constructor called:
copy constructor called:
copy constructor called:
destructor called for B
destructor called: [a]1
catch a exception
```
程序正常退出了, 而且只有第一个A析构被调用了,可是确有严重的内存泄露.  
在复制构造函数中几个新new的B对象都没有被析构

```c
+/tmp $ valgrind --leak-check=full ./a.out
==19807== Memcheck, a memory error detector
==19807== Copyright (C) 2002-2015, and GNU GPL'd, by Julian Seward et al.
==19807== Using Valgrind-3.11.0 and LibVEX; rerun with -h for copyright info
==19807== Command: ./a.out
==19807==
copy constructor called:
copy constructor called:
copy constructor called:
destructor called for B
destructor called: [a]1
catch a exception
==19807==
==19807== HEAP SUMMARY:
==19807==     in use at exit: 73,024 bytes in 9 blocks
==19807==   total heap usage: 14 allocs, 5 frees, 78,248 bytes allocated
==19807==
==19807== 40 bytes in 1 blocks are definitely lost in loss record 4 of 9
==19807==    at 0x4C2E0EF: operator new(unsigned long) (in /usr/lib/valgrind/vgpreload_memcheck-amd64-linux.so)
==19807==    by 0x401525: A::A(A const&) (1.cc:24)
==19807==    by 0x401F5D: __gnu_cxx::new_allocator<A>::construct(A*, A const&) (new_allocator.h:146)
==19807==    by 0x401B5B: void __gnu_cxx::__alloc_traits<std::allocator<A>, A>::construct<A>(std::allocator<A>&, A*, A const&) (alloc_traits.h:137)
==19807==    by 0x401847: std::vector<A, std::allocator<A> >::push_back(A const&) (stl_vector.h:1079)
==19807==    by 0x4011E6: main (1.cc:46)
==19807==
==19807== 40 bytes in 1 blocks are definitely lost in loss record 5 of 9
==19807==    at 0x4C2E0EF: operator new(unsigned long) (in /usr/lib/valgrind/vgpreload_memcheck-amd64-linux.so)
==19807==    by 0x401525: A::A(A const&) (1.cc:24)
==19807==    by 0x401F5D: __gnu_cxx::new_allocator<A>::construct(A*, A const&) (new_allocator.h:146)
==19807==    by 0x401B5B: void __gnu_cxx::__alloc_traits<std::allocator<A>, A>::construct<A>(std::allocator<A>&, A*, A const&) (alloc_traits.h:137)
==19807==    by 0x401847: std::vector<A, std::allocator<A> >::push_back(A const&) (stl_vector.h:1079)
==19807==    by 0x4011F9: main (1.cc:47)
==19807==
==19807== 80 (40 direct, 40 indirect) bytes in 1 blocks are definitely lost in loss record 6 of 9
==19807==    at 0x4C2E0EF: operator new(unsigned long) (in /usr/lib/valgrind/vgpreload_memcheck-amd64-linux.so)
==19807==    by 0x401170: main (1.cc:44)
==19807==
==19807== 80 (40 direct, 40 indirect) bytes in 1 blocks are definitely lost in loss record 7 of 9
==19807==    at 0x4C2E0EF: operator new(unsigned long) (in /usr/lib/valgrind/vgpreload_memcheck-amd64-linux.so)
==19807==    by 0x40118E: main (1.cc:44)
==19807==
==19807== 80 (40 direct, 40 indirect) bytes in 1 blocks are definitely lost in loss record 8 of 9
==19807==    at 0x4C2E0EF: operator new(unsigned long) (in /usr/lib/valgrind/vgpreload_memcheck-amd64-linux.so)
==19807==    by 0x4011AC: main (1.cc:44)
==19807==
==19807== LEAK SUMMARY:
==19807==    definitely lost: 200 bytes in 5 blocks
==19807==    indirectly lost: 120 bytes in 3 blocks
==19807==      possibly lost: 0 bytes in 0 blocks
==19807==    still reachable: 72,704 bytes in 1 blocks
==19807==         suppressed: 0 bytes in 0 blocks
==19807== Reachable blocks (those to which a pointer was found) are not shown.
==19807== To see them, rerun with: --leak-check=full --show-leak-kinds=all
==19807==
==19807== For counts of detected and suppressed errors, rerun with: -v
==19807== ERROR SUMMARY: 5 errors from 5 contexts (suppressed: 0 from 0)
+/tmp $

```

## Item 9 绝不在构造和析构函数中调用virtual函数
例子:  
```c++
class Transaction {
public:
  Transaction() {
    ...
    logTransaction();
  }
  virtual void logTransaction() const = 0;
};

class BuyTransaction : public Transaction {
public:
  virtual void logTransaction() const;
};

int main() {
  BuyTransaction b;
}
```
父类在构造过程中，调用了虚函数，结果就是不会调用子类的虚函数。
上面的例子是编译不通过的，因为父类的logTransaction是纯虚的，会报undefined reference to logTransaction  
如果改成虚函数，那么就可以编译了，执行起来调用的是父类的logTransaction  

`也就是说在父类构造期间，虚函数还不是虚函数`。  

有个有意思的现象:  
如果将调用纯虚函数间接一层调用:  
```c++
...
public:
  Transaction() {
    init();
  }
  void init() {logTransaction();}
  virtual void logTransaction() const = 0;
```
这么写之后，就可以编译了。但是运行就会core dump.  
```
#0  0x00007ffff71a0438 in raise () from /lib/x86_64-linux-gnu/libc.so.6
#1  0x00007ffff71a203a in abort () from /lib/x86_64-linux-gnu/libc.so.6
#2  0x00007ffff7ae09f3 in __gnu_cxx::__verbose_terminate_handler () at ../../../../libstdc++-v3/libsupc++/vterminate.cc:95
#3  0x00007ffff7ae6946 in __cxxabiv1::__terminate (handler=<optimized out>) at ../../../../libstdc++-v3/libsupc++/eh_terminate.cc:47
#4  0x00007ffff7ae6981 in std::terminate () at ../../../../libstdc++-v3/libsupc++/eh_terminate.cc:57
#5  0x00007ffff7ae769f in __cxxabiv1::__cxa_pure_virtual () at ../../../../libstdc++-v3/libsupc++/pure.cc:50
#6  0x0000000000400c07 in Transaction::init (this=0x7fffffffe2e0) at ./1.cc:62
#7  0x0000000000400b49 in Transaction::Transaction (this=0x7fffffffe2e0) at ./1.cc:49
#8  0x0000000000400c81 in BuyTransaction::BuyTransaction (this=0x7fffffffe2e0, s=0x400e06 "c") at ./1.cc:73
#9  0x00000000004009f8 in main () at ./1.cc:83

```
当然，如果不是纯虚，那么也会默默的调用虚函数的父类版本。



# C++ Concurrency in Action

## 5. The C++ memory model and operations on atomic types

### 5.1 Basics
关于`memory model`有两个方面：
1. structural  
  relate to how things are laid out in memory
2. concurrency

#### 5.1.1 Objects and memory locations
- Every variable is an object, including 内部的成员.
- Every object occupies `at least one` memory location.
- fundamental types like int char occupy `exactly one` memory location.
- Adjacent bit fields are part of the same memory location.

#### 5.1.2 Objects, memory locations, and concurrency
两个线程同时访问一个内存位置, 并且存在一个写操作, 那么就存在Race Condition.  
为了avoid race condition, 需要an `enforced ordering` between the accesses in the two threads.
这个顺序可以是:  
- a `fixed ordering` such that one access is always before the other,
- or an ordering that varies between runs of the application, but guarantees that there is `some defined ordering`.  
  `defined ordering`的一种形式如mutex, 可以保证两个操作必须一个接着一个, 虽然并不知道谁先谁后.  
  也可以用`atomic`

If there's no enforced ordering between two accesses to a single memory location from seperate threads, one or both of those accesses is not atomic, and if one or both is a write, then this is a data race and causes `underfined behavior`.  

#### 5.1.3 Modification orders
Modification order包含了这个程序中从变量初始化开始, 所有线程对某个变量的写操作, 通常每次运行时顺序都不一样, 但是在某次运行中,所有线程对此顺序的认知是相同的.  
如果不同线程看到了不同的modification order, 那么就存在`undefiend behavior`.  

也就是说某些执行序列是不允许出现的, 举例说明:  
存在一个modification order, Thread A在读取了其中的某一步的结果, 那么在此读取操作之后的所有读取操作都应该返回最后写入的值, 假设Thread A在当前读操作之后也有一个写操作, 那么在整个的modification order中, Thread A的写操作肯定在刚读取的写操作之后.  
并且同一线程内的写+读操作中读操作要么返回自己刚刚写入的数据,要么读取到其他线程写操作写入的数据, 如果读取到了其他线程写入的数据,那么在modification order中, 当前线程的写操作肯定在其他线程的写操作之前.

### 5.2 Atomic operations and types in c++

关于 `std::atomic<bool>`

STORING A NEW VALUE (OR NOT) DEPENDING ON THE CURRENT VALUE  
`compare-exchange` operation, 有两个函数:  
```c++
bool compare_exchange_weak( T& expected, T desired,
                            std::memory_order success,
                            std::memory_order failure ) noexcept;
bool compare_exchange_weak( T& expected, T desired,
                            std::memory_order success,
                            std::memory_order failure ) volatile noexcept;
bool compare_exchange_weak( T& expected, T desired,
                            std::memory_order order =
                            std::memory_order_seq_cst ) noexcept;
bool compare_exchange_weak( T& expected, T desired,
                            std::memory_order order =
                            std::memory_order_seq_cst ) volatile noexcept;
bool compare_exchange_strong( T& expected, T desired,
                              std::memory_order success,
                              std::memory_order failure ) noexcept;
bool compare_exchange_strong( T& expected, T desired,
                              std::memory_order success,
                              std::memory_order failure ) volatile noexcept;
bool compare_exchange_strong( T& expected, T desired,
                              std::memory_order order =
                              std::memory_order_seq_cst ) noexcept;
bool compare_exchange_strong( T& expected, T desired,
                              std::memory_order order =
                              std::memory_order_seq_cst ) volatile noexcept;
```

通过比较当前值和参数expected, 如果相等则把desired的值存入, 如果不等,那么expected的值会被替换为当前值.如果进行了存入操作,那么返回true, 否则返回false.  

对于`compare_exchange_weak`, 即便当前值和expected值相同,也可能不会进行存入操作, 并且返回false.  
这大概率是因为当前机器缺少`a single compare-and-exchange instruction`, 处理无法保证当前操作can be executed atomically.  
后果是可能在调用`compare_exchange_weak`的时候出现假失败, 一般可以把该函数调用放在while循环里:  
```c++
bool expected = false;
extern atomic<bool> b; // set somewhere else
while (!b.compare_exchange_weak(expected, true) && !expected);
```
如果当前值是true, 那么返回false, expected被改为true,结束
如果当前值为false, 那么有可能返回false, 并且没有改变当前值为true, expected的值还是false, 那么接着循环.  

`compare_exchange_strong`函数保证了只在expected与当前值不等的条件下才会返回false.  

如果不论当前值是多少都想修改当前值为desired, 无论是`compare_exchange_weak`或者`compare_exchange_strong`, 都可以直接调用两次就可以完成.  

这两个函数还有几个重载的版本接收两个std::memory_order类型的参数.  
一共有六种内存序类型:  
```c++
typedef enum memory_order {
  memory_order_relaxed,
  memory_order_consume,
  memory_order_acquire,
  memory_order_release,
  memory_order_acq_rel,
  memory_order_seq_cst
} memory_order;
```
一般默认的内存序是`memory_order_seq_cst`, 是最严格的内存序.  
不同操作可以使用的内存序不同:  

- Store Operation: relaxed, release, seq_cst
- Load Operation: relaxed, consume, acquire, seq_cst
- Read-modify-write Operation: relaxed, consume, acquire, release, acq_rel, seq_cst

`compare_exchange_`函数可以传入两个内存序变量,分别表示成功和失败时使用的内存序.  
一般可以传入success时的内存序为: acq_rel, failed时为:relaxed  
因为失败操作不会发生Store操作,因此failed的内存序不能使用release和memory_order_acq_rel, 并且failed的内存序不能比成功的内存序更严格.  
如果不传入failed时的内存序,那么failed时的内存序使用success时的内存序去掉Store部分的内存序要求,如:  
- success为release内存序,那么failed就成了relaxed  
- acq_rel就成了acquire

`Operations on std::atomic<T*>: pointer arithmetic`

std::atomic<T*>的接口和std::atomic<bool>是一样的, 同样也不是copy-constructable, 也不是copy-assignable.

std::atomic<T*>还有一些新增的操作, 指针算数运算操作: `fetch_add()`, `fetch_sub()`;
以及`+=, -= 前置++, 后置++`等都是这两个操作的封装.
`fetch_add和fetch_sub`需要注意的是, 这两个操作的返回值是原先的值, 而非修改之后的值. 也叫做`exchange-and-add`操作, 是一个原子的`read-modify-write`操作. 因此这些操作也有一个memory order的参数. operator版本的几个函数没有位置来提供内存序参数,因此他们都是seq-cst的.

```c++
std::atomic<T>
```
使用自定义类型时需要满足以下几个条件: 
- 自定义类型必须使用`trivial` copy assignment operator
  - 即不能有虚函数,不能有虚基类
  - 必须使用编译器生成的`copy-assignment operator`.
- 所有基类及所有非静态的自定义类型成员,必须有`trivial copy-assignment operator`.

这样编译器才能使用memcpy()或者等价的assignment operation. 

`compare-exchange`函数在比对是否相同时并不会使用自定义类型的`operator =`, 而是使用`bitwise comparison` 如`memcmp`.  
如果用户自定义的比较函数有特殊语义或者当前类型有一些padding的位,那么有可能导致比较函数认为相同的两个对象比较为不同.
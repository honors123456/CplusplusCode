

### 一.并发

并发：单个系统同时处理多个独立任务

方式：多进程并发和多线程并发

作用：

​			1.解耦合

​			2.性能





### 二.线程管理

#### 1. C++  线程库  < thread > 	------基于 c++ 17

##### 1.1 基本使用：

```c++
#include <iostream>
#include <thread>

typedef struct Data_{
    void* pointer;
    int count;
    char buffer[128];
}Data;

struct func
{
    int& i;
    func(int& i_) : i(i_) {}
    void operator() ()
    {
        for (unsigned j=0 ; j<1000000 ; ++j)
        {
            do_something(i);
        }
    }
};

void f(int i,std::string const& s)
{
    std::cout<<"  ------------  "<<i<<" s"<<std::endl;
}

void f(int i,Data& data)
{
    
}

void f(int i,std::shared_ptr<Data> ptr)
{
    
}


int main()
{
    int i=0;
    func my_func(i);
    std::thread thread1(my_func);
   	
    //1.分离线程，不等待线程结束
    //thread1.detach(); // 会访问已经销毁的变量i
    //2.等待线程结束
    thread1.join();
    
    //3.线程传参，值传递(浅拷贝，深拷贝（有深拷贝机制的对象）)，引用传递（左值引用std::ref,右值移动std::move），指针传递（指针地址浅拷贝）
    std::thread thread2(f,0,"hello");
    thread2.detach();
    
    //4.线程传参
    char buffer[256];
    int j=1024;
    sprintf(buffer,"%s",j);
    
    //std::thread thread3(f,0,buffer); //错误，buffer进行浅拷贝传入线程，生命周期结束，线程依旧访问
    std::thread thread3(f,0,std::string(buffer)); //正确,buffer是左值，std::string()会对左值进行深拷贝，右值进行移动；std::string(buffer)是临时对象，是右值，会移动到线程中，此时buffer的数据副本所有权属于线程,f是引用，直接使用线程上下文中的buffer数据副本
    
    thread3.detach();
    
    Data data;
    //std::thread thread4(f,0,data);  //编译错误，data是左值，线程会进行浅拷贝，浅拷贝会以右值的方式传入f函数，但是f函数参数是引用，会引发错误
    std::thread thread4(f,0,std::ref(data)); //正确,std::ref会以直接引用的方式传入线程上下文
    thread4.join();
    
    
    //thread3中的std::string(buffer)是临时变量，是右值，会自动进行移动，但是std::shared_ptr这类仅支持移动的左值必须显示进行移动
    std::shared_ptr<Data> ptr=std::make_shared<Data>();
    std::thread thread5(f,0,std::move(ptr));
    
    return 0;
}
```

​	分离线程也叫做守护线程，用于长时间运行，例如任务管理器

##### 1.2 所有权转移

​		一个运行的线程所有权可以转移给另一个非运行的线程

```c++
void some_function();
void some_other_function();
std::thread t1(some_function);
std::thread t2=std::move(t1);
t1=std::thread(some_other_function); 	//右值，自动进行移动语义
std::thread t3;
t3=std::move(t2);
t1=std::move(t3); 		//错误，t1已经是运行中的线程
```

​		函数内的线程所有权转移

```c++
std::thread g()
{
    void some_other_function(int);
    std::thread t(some_other_function,42);
    return t;
}

void f(std::thread t);
void g()
{
    void some_function();
    f(std::thread(some_function));	//右值，自动进行移动语义
    std::thread t(some_function);
    f(std::move(t));
}
```



std::thread::hardware_concurrency()  线程数量

std::this_thread::get_id() 线程id



### 三.共享数据

#### 1. 数据竞争

​		1.对数据结构采取某种保护机制

​		2.无锁编程，即对数据结构和不变量进行修改，修改后的结构必须不可分割

​		3.使用事务的方式去处理数据结构的更新，即“软件事务内存”，类似于Git的提交，合并



#### 2.互斥量

```c++
#include <list>
#include <mutex>
#include <algorithm>

std::list<int> nodes;
std::mutex mutex;

void add(int value)
{
    std::lock_guard<std::mutex> guard(mutex);
    nodes.push_back(value);
}

bool contains(int value)
{
    std::lock_guard<std::mutex> guard(mutex);
    return std::find(nodes.begin(),nodes.end(),value) !=
        nodes.end();
}

//c++ 17 可以进行模板类参数推导
//可以简化为 std::lock_guard guard(mutex);
```

##### **指针或引用的“后门”问题**

当类暴露数据的 **指针或引用** 时，外部代码可以通过这些指针或引用直接访问类的数据，而不受互斥锁的保护。具体来说，互斥锁 **只会保护它所在函数的代码**，而一旦你返回了指向数据的指针或引用，外部代码就可以在 **任何时刻**、**任何地方** 访问或修改数据。



#### 3.保护共享数据

​	使用互斥量来保护数据，并不是在每一个成员函数中加入一个 std::lock_guard 对象那么简单。一个指针或引用，也会让这种保护形同虚设。

例如：

```c++
class some_data
{
    int a;
    std::string b;
public:
    void do_something();
};

class data_wrapper
{
private:
    some_data data;
    std::mutex m;
public:
    template<typename Function>
    void process_data(Function func)
    {
        std::lock_guard<std::mutex> l(m);
        func(data); // 1 传递“保护”数据给用户函数
    }
};

some_data* unprotected;

void malicious_function(some_data& protected_data)
{
 	unprotected=&protected_data;
}


data_wrapper x;
void foo()
{
    x.process_data(malicious_function); // 将x实例中的data指向了unprotected指针
    unprotected->do_something(); // 相当于在外部调用了data
}
```


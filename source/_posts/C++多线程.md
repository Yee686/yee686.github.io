---
title: C和C++多线程
tag: [c/c++, multithread]
index_img: /img/cppthread.png
date: 2024-03-26 12:00:00
category: 工具与框架
---

# 多线程

## C多线程

## 1 线程

### 1.1 创建线程

`int pthread_create(pthread_t * tid, const pthread_attr_t * attr, void * ( * func) (void * ), void * arg);`

- 创建进程成功返回1, 否则返回错误码
- `pthread_t * tid`: 带回线程id, `pthread_t`是一个无符号整数类型
- `pthread_attr_t * attr`: 指定线程属性, 如线程优先级/初始栈大小
- `void * ( * func) (void * )`: 多线程执行函数
- `void * arg`: 多线程函数的若干参数, 多个参数需要封装为结构体

### 1.2 结束线程

`void pthread_exit (void *status);`

- 结束线程, `status`指针存储结束后返回状态

### 1.3 等待线程

`int pthread_join (pthread_t tid, void ** status);`

- `pthread_t tid`: 等待线程的id
- `void ** status`: 等待线程的返回状态, 二重指针

### 1.4 其他相关函数

- `pthread_t pthread_self (void);`: 返回当前进程ID
- `int pthread_detach (pthread_t tid);`: 将指定线程转为分离状态. 若指定线程是分离状态，则如果线程退出，那么它所有的资源都将释放, 即回收资源; 如果线程不是分离状态,线程必须保留它的线程ID,退出状态, 直到其他线程对他调用的`pthread_join()`函数.

## 2 同步互斥

### 2.1 互斥锁

- `pthread_mutex_t mutex`: 互斥锁
- `pthread_mutex_init(&mutex,NULL)`: 动态初始化互斥锁为解锁状态
- `pthread_mutex_t mutex = PTHREAD_MUTEX_INITIALIZER`: 静态初始化为解锁状态
- `pthread_mutex_lock(&mutex)`: 访问临界区加锁操作
- `pthread_mutex_unlock(&mutex)`: 访问临界区解锁操作
- `pthread_mutex_destroy(&mutex)`: 销毁互斥锁

``` c
#include <pthread.h>

int pthread_mutex_init(pthread_mutex_t *restrict mutex,     
                    const pthread_mutexattr_t *restrict attr);       /*初始化互斥量*/
int pthread_mutex_destroy(pthread_mutex_t *mutex);      /*销毁互斥量*/
int pthread_mutex_lock(pthread_mutex_t *mutex);
int pthread_mutex_trylock(pthread_mutex_t *mutex);
int pthread_mutex_unlock(pthread_mutex_t *mutex);
```

### 2.2 信号量

- 适用于支持有限个线程的共享资源, 

``` c
#include <semaphore.h>

int sem_init(sem_t *sem, int pshared, unsigned int value);
int sem_wait(sem_t *sem);
int sem_trywait(sem_t *sem);
int sem_post(sem_t * sem);
int sem_destroy(sem_t * sem);
```

### 2.3 条件变量

- `pthread_cond_wait()`的工作原理如下
  - 阻塞等待条件变量cond满足
  - 解除已绑定的互斥锁(类似pthread_mutex_unlock)
  - 当线程被唤醒, pthread_cond_wait()返回时, 解除阻塞并重新绑定互斥锁(类似pthread_mutex_lock)
  - 重新绑定的原因是第三点不是原子操作, 防止其他运行态线程修改条件变量, 使得本线程拿到互斥锁后条件变量改变任然阻塞
  
``` c
#include <pthread.h>

int pthread_cond_destroy(pthread_cond_t *cond);
int pthread_cond_init(pthread_cond_t *restrict cond, const pthread_condattr_t *restrict attr);

// 阻塞等待条件变量cond, mutex为与当前线程绑定的互斥锁
int pthread_cond_wait(pthread_cond_t *restrict cond, pthread_mutex_t *restrict mutex);

// 制定了最长阻塞时长
int pthread_cond_timedwait(pthread_cond_t *restrict cond, pthread_mutex_t *restrict mutex, 
                            const struct timespec *restrict abstime);

// 以广播形式唤醒所以阻塞在条件变量上的线程
int pthread_cond_broadcast(pthread_cond_t *cond);

// 以信号形式唤醒一个阻塞在条件变量上的线程, 由调度策略决定线程唤醒顺序
int pthread_cond_signal(pthread_cond_t *cond);
```

## C++ 多线程

- 头文件`<thread>`
- `std::mutex`: 互斥量, lock()成员函数加锁, unlock()成员函数解锁
- `std::lock_guard`: 作用域锁
- `std::unique_lock`:  独占锁, 手动加锁和解锁
- `std::condition_variable`: 条件变量

  ``` c++
   std::condition_variable cv;
   std::mutex mtx;
   bool ready = false;

   std::unique_lock<std::mutex> lock(mtx);
   cv.wait(lock, []{ return ready; }); // 等待条件满足
   // 条件满足后执行
  ```

- `std::atomi<>`: 原子操作
- `thread_local [type] [var]`: 线程局部存储TLS
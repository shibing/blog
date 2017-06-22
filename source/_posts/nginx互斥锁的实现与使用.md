---
title: nginx互斥锁的实现
date: 2017-06-22 16:34:05
tags:
  nginx
  C
---

nginx 基于原子操作、信号量以及文件锁实现了一个简单高效的互斥锁，当多个 worker 进程之间需要互斥操作时都会用到。下面来看下 nginx 是如何实现它的。

## 原子操作

在实现互斥锁时用到了原子操作，先来了解一下 nginx 下提供的两个原子操作相关的函数：

```c
static ngx_inline ngx_atomic_uint_t
ngx_atomic_cmp_set(ngx_atomic_t *lock, ngx_atomic_uint_t old, ngx_atomic_uint_t set)

static ngx_inline ngx_atomic_int_t
ngx_atomic_fetch_add(ngx_atomic_t *value, ngx_atomic_int_t add)
```
第一个函数是一个 `CAS` 操作，首先它比较 `lock` 地址处的变量是否等于 `old`， 如果相等，就把 `lock` 地址处的变量设为 `set` 变返回成功，否则返回失败。注意上述过程是作为一个原子一起进行的，不会被打断。 用代码可以描述如下：

```c
static ngx_inline ngx_atomic_uint_t
ngx_atomic_cmp_set(ngx_atomic_t *lock, ngx_atomic_uint_t old, ngx_atomic_uint_t set)
{
    if (*lock == old) {
        *lock = set;
        return 1;
    }

    return 0;
}
```

第二个函数是读取 `value` 地址处的变量，并将其与 `add` 相加的结果再写入 `*lock`，然后返回原来 `*lock` 的值，这些操作也是作为一个整体完成的，不会被打断。用代码可描述如下：

```c
static ngx_inline ngx_atomic_int_t
ngx_atomic_fetch_add(ngx_atomic_t *value, ngx_atomic_int_t add)
{
    ngx_atomic_int_t  old;

    old = *value;
    *value += add;

    return old;
}
```

nginx 在实现这两个函数时会首先判断有没有支持原子操作的库，如果有，则直接使用库提供的原子操作实现，如果没有，则会使用汇编语言自己实现。下面以 `x86` 平台下实现 ngx_atomic_cmp_set 的汇编实现方式，实现主要使用了 `cmpxchgq` 指令，代码如下：

```c
static ngx_inline ngx_atomic_uint_t
ngx_atomic_cmp_set(ngx_atomic_t *lock, ngx_atomic_uint_t old,
    ngx_atomic_uint_t set)
{
    u_char  res;

    __asm__ volatile ( //volatile 关键字告诉编译器不要对下面的指令循序进行调整与优化。

         NGX_SMP_LOCK  //这是一个宏，如果是当前是单cpu，则展开为空，如果是多cpu，则展开为lock;，它会锁住内存地址进行排他访问
    "    cmpxchgl  %3, %1;   " //进行 cas 操作
    "    sete      %0;       " //将操作结果写到 res 变量

    : "=a" (res)  // 输出部分
    : "m" (*lock), "a" (old), "r" (set) // 输入部分
    : "cc", "memory" // 破坏描述部分，表示修改了哪些寄存器和内存，提醒编译器优化时要注意。
    );

    return res;
}
```

上面的代码采用 gcc 嵌入汇编方式来进行编写，了解了 `cmpxchgq` 指令后还是比较容易理解的。



## 锁结构体

首先 nginx 使用 `ngx_shmtx_lock` 结构体表示锁，它的各个成员变量如下：

```c 
typedef struct {
#if (NGX_HAVE_ATOMIC_OPS) 
    ngx_atomic_t  *lock;
#if (NGX_HAVE_POSIX_SEM)
    ngx_atomic_t  *wait;
    ngx_uint_t     semaphore;
    sem_t          sem;
#endif
#else  // 不支持原子变量，使用文件锁，效率稍低。
    ngx_fd_t       fd;
    u_char        *name;
#endif
    ngx_uint_t     spin; //获取锁时尝试的自旋次数，使用原子操作实现锁时才有意义
} ngx_shmtx_t;
```

上面的结构体定义使用了两个宏：`NGX_HAVE_ATOMIC_OPS` 与 `NGX_HAVE_POSIX_SEM`，分别用来代表操作系统是否支持原子变量操作与信号量。根据这两个宏的取值，可以有3种不同的互斥锁实现：

1. 不支持原子操作。
2. 支持原子操作，但不支持信号量
3. 支持原子操作，也支持信号量


第1种情况最简单，会直接使用文件锁来实现互斥锁，这时该结构体只有 `fd` 、 `name`和  `spin` 三个字段，但 `spin` 字段是不起作用的。对于2和3两种情况 nginx 均会使用原子变量操作来实现一个自旋锁，其中 `spin` 表示自旋次数。它们两个的区别是：在支持信号量的情况下，如果自旋次数达到了上限而进程还未获取到锁，则进程会在信号量上阻塞等待，进入睡眠状态。不支持信号量的情况，则不会有这样的操作，而是通过调度器直接 「让出」cpu。 下面对这三种情况下锁的实现分别进行介绍。

## 基于文件锁实现的锁
### 锁的创建

首先通过下面的函数创建一个锁：

```c 
  /*
    mtx：要创建的锁
    addr：使用文件锁实现互斥锁时不会用到该变量
    name：文件锁使用的文件
  */
  ngx_int_t
  ngx_shmtx_create(ngx_shmtx_t *mtx, ngx_shmtx_sh_t *addr, u_char *name)
  {

      if (mtx->name) { // mtx->name不为NULL，说明它之前已经创建过锁

          if (ngx_strcmp(name, mtx->name) == 0) { // 之前创建过锁，且与这次创建锁的文件相同，则不需要创建，直接返回
              mtx->name = name;
              return NGX_OK;
          }
          // 销毁之前创建到锁，其实就是关闭之前创建锁时打开的文件。
          ngx_shmtx_destroy(mtx);
      }

      //打开文件
      mtx->fd = ngx_open_file(name, NGX_FILE_RDWR, NGX_FILE_CREATE_OR_OPEN,
                              NGX_FILE_DEFAULT_ACCESS);
      
      //打开文件失败，打印日志，然后返回
      if (mtx->fd == NGX_INVALID_FILE) {
          ngx_log_error(NGX_LOG_EMERG, ngx_cycle->log, ngx_errno,
                        ngx_open_file_n " \"%s\" failed", name);
          return NGX_ERROR;
      }
      
      //使用锁时只需要该文件在内核中的inode信息，所以将该文件删掉
      if (ngx_delete_file(name) == NGX_FILE_ERROR) {
          ngx_log_error(NGX_LOG_ALERT, ngx_cycle->log, ngx_errno,
                        ngx_delete_file_n " \"%s\" failed", name);
      }

      mtx->name = name;

      return NGX_OK;
  }

```


### 阻塞锁的获取

当进程需要进行阻塞加锁时，通过下面的函数进行：

```c
  /*
   通过文件锁的方式实现互斥锁，如果该文件锁正被其他进程占有，则会导致进程阻塞。
  */
  void
  ngx_shmtx_lock(ngx_shmtx_t *mtx)
  {
      ngx_err_t  err;

      //通过获取文件锁来进行加锁。
      err = ngx_lock_fd(mtx->fd);

      if (err == 0) {
          return;
      }

      ngx_log_abort(err, ngx_lock_fd_n " %s failed", mtx->name);
  }
```

上面函数主要的操作就是通过 `ngx_lock_fd` 来获取锁，它的实现如下：

```c
  ngx_err_t
  ngx_lock_fd(ngx_fd_t fd)
  {
      struct flock  fl;

      ngx_memzero(&fl, sizeof(struct flock));

      //设置文件锁的类型为写锁，即互斥锁
      fl.l_type = F_WRLCK;
      fl.l_whence = SEEK_SET;

      //设置操作为F_SETLKW，表示获取不到文件锁时，会阻塞直到可以获取
      if (fcntl(fd, F_SETLKW, &fl) == -1) {
          return ngx_errno;
      }

      return 0;
  }
```

它主要是通过 `fcntl` 函数来获取文件锁。

### 非阻塞锁的获取

上面获取锁的方式是阻塞式的，在获取不到锁时进程会阻塞，但有时候我们并不希望这样，而是不能获取锁时直接返回，nginx 通过这么函数来非阻塞的获取锁：

```c
  ngx_uint_t
  ngx_shmtx_trylock(ngx_shmtx_t *mtx)
  {
      ngx_err_t  err;

      // 与上面的阻塞版本比较，最主要的变化是将 ngx_lock_fd 函数换成了 ngx_trylock_fd
      err = ngx_trylock_fd(mtx->fd);

      // 获取锁成功，返回1
      if (err == 0) {
          return 1;
      }

      // 获取锁失败，如果错误码是 NGX_EAGAIN，表示文件锁正被其他进程占用，返回0
      if (err == NGX_EAGAIN) {
          return 0;
      }

  #if __osf__ /* Tru64 UNIX */

      if (err == NGX_EACCES) {
          return 0;
      }

  #endif

      // 其他错误都不应该发生，打印错误日志
      ngx_log_abort(err, ngx_trylock_fd_n " %s failed", mtx->name);

      return 0;
  }
```

可以看到与阻塞版本相比，非阻塞版本最主要的变化 `ngx_lock_fd` 换成了 `ngx_trylock_fd`, 它的实现如下：

```c
  ngx_err_t
  ngx_trylock_fd(ngx_fd_t fd)
  {
      struct flock  fl;

      ngx_memzero(&fl, sizeof(struct flock));

      //锁的类型同样是写锁
      fl.l_type = F_WRLCK;
      fl.l_whence = SEEK_SET;

      //操作变成了 F_SETLK, 该操作在获取不到锁时会直接返回，而不会阻塞进程。
      if (fcntl(fd, F_SETLK, &fl) == -1) {
          return ngx_errno;
      }

      return 0;
  }
```

### 锁的释放

上面说了如何加锁，接下来看一下如何释放锁，逻辑比较简单，直接放代码：

```c
  void
  ngx_shmtx_unlock(ngx_shmtx_t *mtx)
  {
      ngx_err_t  err;

      //调用 ngx_unlock_fd函数释放锁
      err = ngx_unlock_fd(mtx->fd);

      if (err == 0) {
          return;
      }

      ngx_log_abort(err, ngx_unlock_fd_n " %s failed", mtx->name);
  }
```

```c
  ngx_err_t
  ngx_unlock_fd(ngx_fd_t fd)
  {
      struct flock  fl;

      ngx_memzero(&fl, sizeof(struct flock));

      //锁的类型为 F_UNLCK, 表示释放锁
      fl.l_type = F_UNLCK;
      fl.l_whence = SEEK_SET;

      if (fcntl(fd, F_SETLK, &fl) == -1) {
          return  ngx_errno;
      }

      return 0;
  }
```

## 基于原子操作实现锁

上面谈到了在不支持原子操作时，nginx 如何使用文件锁来实现互斥锁。现在操作系统一般都支持原子操作，用它实现互斥锁效率会较文件锁的方式更高，这也是 nginx 默认选用该种方式实现锁的原因，下面看一下它是如何实现的。

### 锁的创建

与上面一样，我们还是先看是如何创建锁的：

```c
  /*
   mtx： 要创建的锁
   addr：创建锁时，内部用到的原子变量
   name：没有意义，只有上
  */
  ngx_int_t
  ngx_shmtx_create(ngx_shmtx_t *mtx, ngx_shmtx_sh_t *addr, u_char *name)
  {
      // 保存原子变量的地址，由于锁时多个进程之间共享的，那么原子变量一般在共享内存进行分配
      // 上面的addr就表示在共享内存中分配的内存地址，至于共享内存的分配下次再说
      mtx->lock = &addr->lock;

      // 在不支持信号量时，spin只表示锁的自旋次数，那么该值为0或负数表示不进行自旋，直接让出cpu，
      // 当支持信号量时，它为-1表示，不要使用信号量将进程置于睡眠状态，这对 nginx 的性能至关重要。
      if (mtx->spin == (ngx_uint_t) -1) {
          return NGX_OK;
      }
      // 默认自旋次数是2048
      mtx->spin = 2048;

      // 支持信号量，继续执行下面代码，主要是信号量的初始化。
  #if (NGX_HAVE_POSIX_SEM)

      mtx->wait = &addr->wait;

      //初始化信号量，第二个参数1表示，信号量使用在多进程环境中，第三个参数0表示信号量的初始值
      //当信号量的值小于等于0时，尝试等待信号量会阻塞
      //当信号量大于0时，尝试等待信号量会成功，并把信号量的值减一。
      if (sem_init(&mtx->sem, 1, 0) == -1) {
          ngx_log_error(NGX_LOG_ALERT, ngx_cycle->log, ngx_errno,
                        "sem_init() failed");
      } else {
          mtx->semaphore = 1;
      }

  #endif

      return NGX_OK;
  }
```

该函数的 `addr` 指针变量指向进行原子操作用到的原子变量，它的类型如下：

```c
typedef struct {
    // 通过对该变量进行原子操作来进行锁的获取与释放
    ngx_atomic_t   lock;
#if (NGX_HAVE_POSIX_SEM)
    // 支持信号量时才会有该成语变量，表示当前在在变量上等待的进程数目。
    ngx_atomic_t   wait;
#endif
} ngx_shmtx_sh_t;
```
由于锁是多个进程之间共享的， 所以 `addr` 指向的内存都是在共享内存进行分配的。

### 阻塞锁的获取

与文件锁实现的互斥锁一样，依然有阻塞和非阻塞类型，下面首先来看下阻塞锁的实现，相比于文件锁实现的方式要复杂很多:

```c
  void
  ngx_shmtx_lock(ngx_shmtx_t *mtx)
  {
      ngx_uint_t         i, n;

      ngx_log_debug0(NGX_LOG_DEBUG_CORE, ngx_cycle->log, 0, "shmtx lock");

      for ( ;; ) {

          //尝试获取锁，如果*mtx->lock为0，表示锁未被其他进程占有，
          //这时调用ngx_atomic_cmp_set这个原子操作尝试将*mtx->lock设置为进程id，如果设置成功，则表示加锁成功，否则失败。
          //注意：由于在多进程环境下执行，*mtx->lock == 0 为真时，并不能确保ngx_atomic_cmp_set函数执行成功
          if (*mtx->lock == 0 && ngx_atomic_cmp_set(mtx->lock, 0, ngx_pid)) {
              return;
          }

          // 获取锁失败了，这时候判断cpu的数目，如果数目大于1，则先自旋一段时间，然后再让出cpu
          // 如果cpu数目为1，则没必要进行自旋了，应该直接让出cpu给其他进程执行。
          if (ngx_ncpu > 1) {

              for (n = 1; n < mtx->spin; n <<= 1) {

                  for (i = 0; i < n; i++) {
                      // ngx_cpu_pause函数并不是真的将程序暂停，而是为了提升循环等待时的性能，并且可以降低系统功耗。
                      // 实现它时往往是一个指令： `__asm__`("pause")
                      ngx_cpu_pause();
                  }
                  
                  // 再次尝试获取锁
                  if (*mtx->lock == 0
                      && ngx_atomic_cmp_set(mtx->lock, 0, ngx_pid))
                  {
                      return;
                  }
              }
          }

          // 如果支持信号量，会执行到这里
  #if (NGX_HAVE_POSIX_SEM)
          // 上面自旋次数已经达到，依然没有获取锁，将进程在信号量上挂起，等待其他进程释放锁后再唤醒。
          if (mtx->semaphore) { // 使用信号量进行阻塞，即上面设置创建锁时，mtx的spin成员变量的值不是-1
              
              // 当前在该信号量上等待的进程数目加一
              (void) ngx_atomic_fetch_add(mtx->wait, 1);

              // 尝试获取一次锁，如果获取成功，将等待的进程数目减一，然后返回
              if (*mtx->lock == 0 && ngx_atomic_cmp_set(mtx->lock, 0, ngx_pid)) {
                  (void) ngx_atomic_fetch_add(mtx->wait, -1);
                  return;
              }

              ngx_log_debug1(NGX_LOG_DEBUG_CORE, ngx_cycle->log, 0,
                             "shmtx wait %uA", *mtx->wait);

              //  在信号量上进行等待
              while (sem_wait(&mtx->sem) == -1) {
                  ngx_err_t  err;

                  err = ngx_errno;

                  if (err != NGX_EINTR) {
                      ngx_log_error(NGX_LOG_ALERT, ngx_cycle->log, err,
                                    "sem_wait() failed while waiting on shmtx");
                      break;
                  }
              }

              ngx_log_debug0(NGX_LOG_DEBUG_CORE, ngx_cycle->log, 0,
                             "shmtx awoke");
              // 执行到此，肯定是其他进程释放锁了，所以继续回到循环的开始，尝试再次获取锁，注意它并不会执行下面的386行
              continue;
          }

  #endif
          // 在没有获取到锁，且不使用信号量时，会执行到这里，他一般通过 sched_yield 函数实现，让调度器暂时将进程切出，让其他进程执行。
          // 在其它进程执行后有可能释放锁，那么下次调度到本进程时，则有可能获取成功。
          ngx_sched_yield();
      }
  }
  
```

上面代码的实现流程通过注释已经描述的很清楚了，再强调一点就是，使用信号量与否的区别就在于获取不到锁时进行的操作不同，如果使用信号量，则会在信号量上阻塞，进程进入睡眠状态。而不使用信号量，则是暂时「让出」cpu，进程并不会进入睡眠状态，这会减少内核态与用户态度切换带来的开销，所以往往性能更好，因此在 nginx 中使用锁时一般不使用信号量，比如负载均衡均衡锁的初始化方式如下：

```c
ngx_accept_mutex.spin = (ngx_uint_t) -1;
```

将spin值设为-1，表示不使用信号量。

### 非阻塞锁的获取

非阻塞锁的代码就比较简单了，因为是非阻塞的，所以在获取不到锁时不需要考虑进程是否需要睡眠，也就不需要使用信号量，实现如下：

```c
ngx_uint_t
ngx_shmtx_trylock(ngx_shmtx_t *mtx)
{
    return (*mtx->lock == 0 && ngx_atomic_cmp_set(mtx->lock, 0, ngx_pid));
}
```

### 锁的释放

释放锁时，主要操作是将原子变量设为0，如果使用信号量，则可能还需要唤醒在信号量上等候的进程：

```c
void
ngx_shmtx_unlock(ngx_shmtx_t *mtx)
{
    if (mtx->spin != (ngx_uint_t) -1) {
        ngx_log_debug0(NGX_LOG_DEBUG_CORE, ngx_cycle->log, 0, "shmtx unlock");
    }

    //释放锁，将原子变量设为0，同时唤醒在信号量上等待的进程
    if (ngx_atomic_cmp_set(mtx->lock, ngx_pid, 0)) {
        ngx_shmtx_wakeup(mtx);
    }
}
```

其中  `ngx_shmtx_wakeup` 的实现如下：

```c
  static void
  ngx_shmtx_wakeup(ngx_shmtx_t *mtx)
  {
  // 如果不支持信号量，那么该函数为空，啥也不做
  #if (NGX_HAVE_POSIX_SEM)
      ngx_atomic_uint_t  wait;

      // 如果没有使用信号量，直接返回
      if (!mtx->semaphore) {
          return;
      }

      // 将在信号量上等待的进程数减1，因为是多进程环境，ngx_atomic_cmp_set不一定能一次成功，所以需要循环调用
      for ( ;; ) {

          wait = *mtx->wait;

          // wait 小于等于0，说明当前没有进程在信号量上睡眠
          if ((ngx_atomic_int_t) wait <= 0) {
              return;
          }

          if (ngx_atomic_cmp_set(mtx->wait, wait, wait - 1)) {
              break;
          }
      }

      ngx_log_debug1(NGX_LOG_DEBUG_CORE, ngx_cycle->log, 0,
                     "shmtx wake %uA", wait);

      // 将信号量的值加1
      if (sem_post(&mtx->sem) == -1) {
          ngx_log_error(NGX_LOG_ALERT, ngx_cycle->log, ngx_errno,
                        "sem_post() failed while wake shmtx");
      }

  #endif
  }
```

## 锁的销毁

因为锁的销毁代码比较简单，就不分开进行说明了。对于基于文件锁实现的互斥锁在销毁时需要关闭打开的文件。对于基于原子变量实现的锁，如果支持信号量，则需要销毁创建的信号量，代码分别入下：

基于文件锁实现的锁的销毁：

```c
void
ngx_shmtx_destroy(ngx_shmtx_t *mtx)
{
    if (ngx_close_file(mtx->fd) == NGX_FILE_ERROR) {
        ngx_log_error(NGX_LOG_ALERT, ngx_cycle->log, ngx_errno,
                      ngx_close_file_n " \"%s\" failed", mtx->name);
    }
}
```

基于原子操作实现的锁的销毁：

```c
void
ngx_shmtx_destroy(ngx_shmtx_t *mtx)
{
#if (NGX_HAVE_POSIX_SEM)

    if (mtx->semaphore) {
        if (sem_destroy(&mtx->sem) == -1) {
            ngx_log_error(NGX_LOG_ALERT, ngx_cycle->log, ngx_errno,
                          "sem_destroy() failed");
        }
    }

#endif
}
```












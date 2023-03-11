# 【NO.189】手把手教你纯c实现异常捕获try-catch组件

## 1.try / catch / finally / throw 介绍

在java，python，c++里面都有try catch异常捕获。在try代码块里面执行的函数，如果出错有异常了，就会throw把异常抛出来，抛出来的异常被catch接收进行处理，而finally意味着无论有没有异常，都会执行finally代码块内的代码。

```text
try{
    connect_sql();//throw
}catch(){

}finally {
    
};
```

## 2.如何实现try-catch这一机制？

关于跳转，有两个跳转。那么在这里我们必然选用长跳转。

1. goto：函数内跳转，短跳转
2. setjmp/longjmp：跨函数跳转，长跳转

setjmp/longjmp这两个函数是不存在压栈出栈的，也就是说longjmp跳转到setjmp的地方后，会覆盖之前的栈。

## 3.setjmp/longjmp使用介绍(重点)

- setjmp(env)：设置跳转的位置，第一次返回0，后续返回longjmp的第二个参数
- longjmp(env, idx)：跳转到设置env的位置，第二个参数就是setjmp()的返回值

```text
#include <stdio.h>
#include <setjmp.h>


jmp_buf env;
int count = 0;

void sub_func(int idx) {
    printf("sub_func -->idx : %d\n", idx);
    //第二个参数就是setjmp()的返回值
    longjmp(env, idx);
}

int main() {
    int idx = 0;
    //设置跳转标签，第一次返回0
    count = setjmp(env);
    if (count == 0) {
        printf("count : %d\n", count);
        sub_func(++idx);
    }
    else if (count == 1) {
        printf("count : %d\n", count);
        sub_func(++idx);
    }
    else if (count == 2) {
        printf("count : %d\n", count);
        sub_func(++idx);
    }
    else {
        printf("other count \n");
    }

    return 0;
}
```



```text
count : 0
sub_func -->idx : 1
count : 1
sub_func -->idx : 2
count : 2
sub_func -->idx : 3
other count 
```

## 4.try-catch 和 setjmp/longjmp 的关系

```text
try ---> setjmp(env)

throw ---> longjmp(env,Exception)

catch(Exception)
```

![img](https://pic3.zhimg.com/80/v2-9218537f2584fa9088242f6b0f9af132_720w.webp)

我们其实可以分析出来，setjmp和count==0的地方，相当于try，后面的else if 相当于catch，最后一个else，其实并不是finally，因为finally是不管怎么样都会执行，上图我标注的其实是误导的。应该是下图这样才对。

![img](https://pic4.zhimg.com/80/v2-308dfb49a985ba5e62b9d5b313e8d587_720w.webp)

![img](https://pic1.zhimg.com/80/v2-0f3fa86e32618276f82ec6a7e4338794_720w.webp)

宏定义实现try-catch Demo

4个关键字分析出来它们的关系之后，其实我们就能用宏定义来实现了。

```text
#include <stdio.h>
#include <setjmp.h>


typedef struct _Exception {
    jmp_buf env;
    int exceptype;
} Exception;

#define Try(excep) if((excep.exceptype=setjmp(excep.env))==0)
#define Catch(excep, ExcepType) else if(excep.exceptype==ExcepType)
#define Throw(excep, ExcepType) longjmp(excep.env,ExcepType)
#define Finally

void throw_func(Exception ex, int idx) {
    printf("throw_func -->idx : %d\n", idx);
    Throw(ex, idx);
}

int main() {
    int idx = 0;

    Exception ex;
    Try(ex) {
        printf("ex.exceptype : %d\n", ex.exceptype);
        throw_func(ex, ++idx);
    }
    Catch(ex, 1) {
        printf("ex.exceptype : %d\n", ex.exceptype);
    }
    Catch(ex, 2) {
        printf("ex.exceptype : %d\n", ex.exceptype);
    }
    Catch(ex, 3) {
        printf("ex.exceptype : %d\n", ex.exceptype);
    }
    Finally{
        printf("Finally\n");
    };

    return 0;
}
```



```text
ex.exceptype : 0
throw_func -->idx : 1
ex.exceptype : 1
Finally
```

![img](https://pic4.zhimg.com/80/v2-1bc2b5615f74a8697b9b28a5d778cba3_720w.webp)



## 5.实现try-catch的三个问题

虽然现在demo版看起来像这么回事了，但是还是有两个问题：

1. 在哪个文件哪个函数哪个行抛的异常？
2. try-catch嵌套怎么做？
3. try-catch线程安全怎么做？

### 5.1 在哪个文件哪个函数哪个行抛的异常

系统提供了三个宏可以供我们使用，如果我们没有catch到异常，我们就可以打印出来

```text
__func__, __FILE__, __LINE__
```

### 5.2 try-catch嵌套怎么做？

我们知道try-catch是可以嵌套的，那么这就形成了一个栈的数据结构，现在下面有三个try，每个`setjmp`对应的都是不同的`jmp_buf`,那么我们可以定义一个jmp_buf的栈。

```text
try{
    try{
        try{

        }catch(){

        }
    }catch(){

    }
}catch(){

}finally{

};
```

那么我们很容易能写出来，既然是栈，try的时候我们就插入一个结点，catch的时候我们就pop一个出来。

```text
#define EXCEPTION_MESSAGE_LENGTH                512

typedef struct _ntyException {
    const char *name;
} ntyException;

ntyException SQLException = {"SQLException"};
ntyException TimeoutException = {"TimeoutException"};


typedef struct _ntyExceptionFrame {
    jmp_buf env;

    int line;
    const char *func;
    const char *file;

    ntyException *exception;
    struct _ntyExceptionFrame *next;

    char message[EXCEPTION_MESSAGE_LENGTH + 1];
} ntyExceptionFrame;

enum {
    ExceptionEntered = 0,//0
    ExceptionThrown,    //1
    ExceptionHandled, //2
    ExceptionFinalized//3
};
```

### 5.3 try-catch线程安全

每个线程都可以try-catch，但是我们以及知道了是个栈结构，既ExceptionStack，那么每个线程是独有一个ExceptionStack呢？还是共享同一个ExceptionStack？很明显，A线程的异常应该有A的处理，而不是由B线程处理。

```text
/* ** **** ******** **************** Thread safety **************** ******** **** ** */

#define ntyThreadLocalData                      pthread_key_t
#define ntyThreadLocalDataSet(key, value)       pthread_setspecific((key), (value))
#define ntyThreadLocalDataGet(key)              pthread_getspecific((key))
#define ntyThreadLocalDataCreate(key)           pthread_key_create(&(key), NULL)

ntyThreadLocalData ExceptionStack;

static void init_once(void) {
    ntyThreadLocalDataCreate(ExceptionStack);
}

static pthread_once_t once_control = PTHREAD_ONCE_INIT;

void ntyExceptionInit(void) {
    pthread_once(&once_control, init_once);
}
```

## 6.代码实现与解释

### 6.1 try

首先创建一个新节点入栈，然后setjmp设置一个标记，接下来就是大括号里面的操作了，如果有异常，那么就会被throw抛出来，为什么这里最后一行是if？因为longjmp的时候，返回的地方是setjmp，不要忘了！要时刻扣住longjmp和setjmp。

```text
#define Try do {                                                                        \
            volatile int Exception_flag;                                                \
            ntyExceptionFrame frame;                                                    \
            frame.message[0] = 0;                                                       \
            frame.next = (ntyExceptionFrame*)ntyThreadLocalDataGet(ExceptionStack);     \
            ntyThreadLocalDataSet(ExceptionStack, &frame);                              \
            Exception_flag = setjmp(frame.env);                                         \
            if (Exception_flag == ExceptionEntered) {
```



```text
Try{
	//...
    Throw(A, "A");
}
```

### 6.2 throw

在这里，我们不应该把throw定义成宏，而应该定义成函数。这里分两类，一类是try里面的throw，一类是没有try直接throw。

- 对于try里面的异常，我们将其状态变成ExceptionThrown，然后longjmp到setjmp的地方，由catch处理
- 对于直接抛的异常，必然没有catch去捕获，那么我们直接打印出来
- 如果第一种情况的异常，没有被catch捕获到怎么办呢？后面会被ReThrow出来，对于再次被抛出，我们就直接进行打印异常

这里的`##__VA_ARGS__`是可变参数，具体不多介绍了，不是本文重点。

```text
#define ReThrow                    ntyExceptionThrow(frame.exception, frame.func, frame.file, frame.line, NULL)
#define Throw(e, cause, ...)    ntyExceptionThrow(&(e), __func__, __FILE__, __LINE__, cause, ##__VA_ARGS__,NULL)
```



```text
void ntyExceptionThrow(ntyException *excep, const char *func, const char *file, int line, const char *cause, ...) {
    va_list ap;
    ntyExceptionFrame *frame = (ntyExceptionFrame *) ntyThreadLocalDataGet(ExceptionStack);

    if (frame) {
        //异常名
        frame->exception = excep;
        frame->func = func;
        frame->file = file;
        frame->line = line;
        //异常打印的信息
        if (cause) {
            va_start(ap, cause);
            vsnprintf(frame->message, EXCEPTION_MESSAGE_LENGTH, cause, ap);
            va_end(ap);
        }

        ntyExceptionPopStack;

        longjmp(frame->env, ExceptionThrown);
    }
        //没有被catch,直接throw
    else if (cause) {
        char message[EXCEPTION_MESSAGE_LENGTH + 1];

        va_start(ap, cause);
        vsnprintf(message, EXCEPTION_MESSAGE_LENGTH, cause, ap);
        va_end(ap);
        printf("%s: %s\n raised in %s at %s:%d\n", excep->name, message, func ? func : "?", file ? file : "?", line);
    }
    else {
        printf("%s: %p\n raised in %s at %s:%d\n", excep->name, excep, func ? func : "?", file ? file : "?", line);
    }
}
```

### 6.3 Catch

如果还是ExceptionEntered状态，说明没有异常，没有throw。如果捕获到异常了，那么其状态就是ExceptionHandled。

```text
#define Catch(nty_exception) \
                if (Exception_flag == ExceptionEntered) ntyExceptionPopStack; \
            } else if (frame.exception == &(nty_exception)) { \
                Exception_flag = ExceptionHandled;
```

### 6.4 Finally

finally也是一样，如果还是ExceptionEntered状态，说明没有异常没有捕获，那么现在状态是终止阶段。

```text
#define Finally \
                if (Exception_flag == ExceptionEntered) ntyExceptionPopStack; \
            } { \
                if (Exception_flag == ExceptionEntered)    \
                    Exception_flag = ExceptionFinalized;
```

### 6.5 EndTry

有些人看到EndTry可能会有疑问，try-catch一共不就4个关键字吗？怎么你多了一个。我们先来看看EndTry做了什么，首先如果是ExceptionEntered状态，那意味着什么？意味着没有throw，没有catch，没有finally，只有try，我们需要对这种情况进行处理，要出栈。还有一种情况，如果是ExceptionThrown状态，说明什么？没有被catch捕获到，那么我们就再次抛出，进行打印错误。至于为什么多个EndTry，写起来方便呗~

```text
#define EndTry \
                if (Exception_flag == ExceptionEntered) ntyExceptionPopStack; \
            } if (Exception_flag == ExceptionThrown) {ReThrow;} \
            } while (0)
```

### 6.6 try-catch代码

```text
/* ** **** ******** **************** try / catch / finally / throw **************** ******** **** ** */

#define EXCEPTION_MESSAGE_LENGTH                512

typedef struct _ntyException {
    const char *name;
} ntyException;

ntyException SQLException = {"SQLException"};
ntyException TimeoutException = {"TimeoutException"};


typedef struct _ntyExceptionFrame {
    jmp_buf env;

    int line;
    const char *func;
    const char *file;

    ntyException *exception;
    struct _ntyExceptionFrame *next;

    char message[EXCEPTION_MESSAGE_LENGTH + 1];
} ntyExceptionFrame;

enum {
    ExceptionEntered = 0,//0
    ExceptionThrown,    //1
    ExceptionHandled, //2
    ExceptionFinalized//3
};

#define ntyExceptionPopStack    \
    ntyThreadLocalDataSet(ExceptionStack, ((ntyExceptionFrame*)ntyThreadLocalDataGet(ExceptionStack))->next)


#define ReThrow                    ntyExceptionThrow(frame.exception, frame.func, frame.file, frame.line, NULL)
#define Throw(e, cause, ...)    ntyExceptionThrow(&(e), __func__, __FILE__, __LINE__, cause, ##__VA_ARGS__,NULL)

#define Try do {                                                                        \
            volatile int Exception_flag;                                                \
            ntyExceptionFrame frame;                                                    \
            frame.message[0] = 0;                                                       \
            frame.next = (ntyExceptionFrame*)ntyThreadLocalDataGet(ExceptionStack);     \
            ntyThreadLocalDataSet(ExceptionStack, &frame);                              \
            Exception_flag = setjmp(frame.env);                                         \
            if (Exception_flag == ExceptionEntered) {

#define Catch(nty_exception) \
                if (Exception_flag == ExceptionEntered) ntyExceptionPopStack; \
            } else if (frame.exception == &(nty_exception)) { \
                Exception_flag = ExceptionHandled;


#define Finally \
                if (Exception_flag == ExceptionEntered) ntyExceptionPopStack; \
            } { \
                if (Exception_flag == ExceptionEntered)    \
                    Exception_flag = ExceptionFinalized;
#define EndTry \
                if (Exception_flag == ExceptionEntered) ntyExceptionPopStack; \
            } if (Exception_flag == ExceptionThrown) {ReThrow;} \
            } while (0)

void ntyExceptionThrow(ntyException *excep, const char *func, const char *file, int line, const char *cause, ...) {
    va_list ap;
    ntyExceptionFrame *frame = (ntyExceptionFrame *) ntyThreadLocalDataGet(ExceptionStack);

    if (frame) {
        //异常名
        frame->exception = excep;
        frame->func = func;
        frame->file = file;
        frame->line = line;
        //异常打印的信息
        if (cause) {
            va_start(ap, cause);
            vsnprintf(frame->message, EXCEPTION_MESSAGE_LENGTH, cause, ap);
            va_end(ap);
        }

        ntyExceptionPopStack;

        longjmp(frame->env, ExceptionThrown);
    }
        //没有被catch,直接throw
    else if (cause) {
        char message[EXCEPTION_MESSAGE_LENGTH + 1];

        va_start(ap, cause);
        vsnprintf(message, EXCEPTION_MESSAGE_LENGTH, cause, ap);
        va_end(ap);
        printf("%s: %s\n raised in %s at %s:%d\n", excep->name, message, func ? func : "?", file ? file : "?", line);
    }
    else {
        printf("%s: %p\n raised in %s at %s:%d\n", excep->name, excep, func ? func : "?", file ? file : "?", line);
    }
}
```

### 6.7 Debug测试代码

```text
/* ** **** ******** **************** debug **************** ******** **** ** */

ntyException A = {"AException"};
ntyException B = {"BException"};
ntyException C = {"CException"};
ntyException D = {"DException"};

void *thread(void *args) {
    pthread_t selfid = pthread_self();
    Try
            {
                Throw(A, "A");
            }
        Catch (A)
            {
                printf("catch A : %ld\n", selfid);
            }
    EndTry;

    Try
            {
                Throw(B, "B");
            }
        Catch (B)
            {
                printf("catch B : %ld\n", selfid);
            }
    EndTry;

    Try
            {
                Throw(C, "C");
            }
        Catch (C)
            {
                printf("catch C : %ld\n", selfid);
            }
    EndTry;

    Try
            {
                Throw(D, "D");
            }
        Catch (D)
            {
                printf("catch D : %ld\n", selfid);
            }
    EndTry;

    Try
            {
                Throw(A, "A Again");
                Throw(B, "B Again");
                Throw(C, "C Again");
                Throw(D, "D Again");
            }
        Catch (A)
            {
                printf("catch A again : %ld\n", selfid);
            }
        Catch (B)
            {
                printf("catch B again : %ld\n", selfid);
            }
        Catch (C)
            {
                printf("catch C again : %ld\n", selfid);
            }
        Catch (D)
            {
                printf("catch B again : %ld\n", selfid);
            }
    EndTry;
}


#define PTHREAD_NUM        8

int main(void) {
    ntyExceptionInit();

    printf("\n\n=> Test1: Throw\n");
    {
        Throw(D, NULL);     //ntyExceptionThrow(&(D), "_function_name_", "_file_name_", 202, ((void *) 0), ((void *) 0))
        Throw(C, "null C"); //ntyExceptionThrow(&(C), "_function_name_", "_file_name_", 203, "null C", ((void *) 0))
    }
    printf("=> Test1: Ok\n\n");


    printf("\n\n=> Test2: Try-Catch Double Nesting\n");
    {
        Try
                {
                    Try
                            {
                                Throw(B, "call B");
                            }
                        Catch (B)
                            {
                                printf("catch B \n");
                            }
                    EndTry;
                    Throw(A, NULL);
                }
            Catch(A)
                {
                    printf("catch A \n");
                    printf("Result: Ok\n");
                }
        EndTry;
    }
    printf("=> Test2: Ok\n\n");


    printf("\n\n=> Test3: Try-Catch Triple  Nesting\n");
    {
        Try
                {
                    Try
                            {

                                Try
                                        {
                                            Throw(C, "call C");
                                        }
                                    Catch (C)
                                        {
                                            printf("catch C\n");
                                        }
                                EndTry;
                                Throw(B, "call B");
                            }
                        Catch (B)
                            {
                                printf("catch B\n");
                            }
                    EndTry;
                    Throw(A, NULL);
                }
            Catch(A)
                {
                    printf("catch A\n");
                }
        EndTry;
    }
    printf("=> Test3: Ok\n\n");


    printf("=> Test4: Test Thread-safeness\n");
    int i = 0;
    pthread_t th_id[PTHREAD_NUM];

    for (i = 0; i < PTHREAD_NUM; i++) {
        pthread_create(&th_id[i], NULL, thread, NULL);
    }

    for (i = 0; i < PTHREAD_NUM; i++) {
        pthread_join(th_id[i], NULL);
    }
    printf("=> Test4: Ok\n\n");

    printf("\n\n=> Test5: No Success Catch\n");
    {
        Try
                {
                    Throw(A, "no catch A ,should Rethrow");
                }
        EndTry;
    }
    printf("=> Test5: Rethrow Success\n\n");

    printf("\n\n=> Test6: Normal Test\n");
    {
        Try
                {
                    Throw(A, "call A");
                }
            Catch(A)
                {
                    printf("catch A\n");

                }
            Finally
                {
                    printf("wxf nb\n");
                };
        EndTry;
    }
    printf("=> Test6: ok\n\n");

}
```

## 7.线程安全、try-catch、Debug测试代码 汇总

```text
#include <stdio.h>
#include <setjmp.h>
#include <pthread.h>
#include <stdarg.h>


/* ** **** ******** **************** Thread safety **************** ******** **** ** */

#define ntyThreadLocalData                      pthread_key_t
#define ntyThreadLocalDataSet(key, value)       pthread_setspecific((key), (value))
#define ntyThreadLocalDataGet(key)              pthread_getspecific((key))
#define ntyThreadLocalDataCreate(key)           pthread_key_create(&(key), NULL)

ntyThreadLocalData ExceptionStack;

static void init_once(void) {
    ntyThreadLocalDataCreate(ExceptionStack);
}

static pthread_once_t once_control = PTHREAD_ONCE_INIT;

void ntyExceptionInit(void) {
    pthread_once(&once_control, init_once);
}

/* ** **** ******** **************** try / catch / finally / throw **************** ******** **** ** */

#define EXCEPTION_MESSAGE_LENGTH                512

typedef struct _ntyException {
    const char *name;
} ntyException;

ntyException SQLException = {"SQLException"};
ntyException TimeoutException = {"TimeoutException"};


typedef struct _ntyExceptionFrame {
    jmp_buf env;

    int line;
    const char *func;
    const char *file;

    ntyException *exception;
    struct _ntyExceptionFrame *next;

    char message[EXCEPTION_MESSAGE_LENGTH + 1];
} ntyExceptionFrame;

enum {
    ExceptionEntered = 0,//0
    ExceptionThrown,    //1
    ExceptionHandled, //2
    ExceptionFinalized//3
};

#define ntyExceptionPopStack    \
    ntyThreadLocalDataSet(ExceptionStack, ((ntyExceptionFrame*)ntyThreadLocalDataGet(ExceptionStack))->next)


#define ReThrow                    ntyExceptionThrow(frame.exception, frame.func, frame.file, frame.line, NULL)
#define Throw(e, cause, ...)    ntyExceptionThrow(&(e), __func__, __FILE__, __LINE__, cause, ##__VA_ARGS__,NULL)

#define Try do {                                                                        \
            volatile int Exception_flag;                                                \
            ntyExceptionFrame frame;                                                    \
            frame.message[0] = 0;                                                       \
            frame.next = (ntyExceptionFrame*)ntyThreadLocalDataGet(ExceptionStack);     \
            ntyThreadLocalDataSet(ExceptionStack, &frame);                              \
            Exception_flag = setjmp(frame.env);                                         \
            if (Exception_flag == ExceptionEntered) {

#define Catch(nty_exception) \
                if (Exception_flag == ExceptionEntered) ntyExceptionPopStack; \
            } else if (frame.exception == &(nty_exception)) { \
                Exception_flag = ExceptionHandled;


#define Finally \
                if (Exception_flag == ExceptionEntered) ntyExceptionPopStack; \
            } { \
                if (Exception_flag == ExceptionEntered)    \
                    Exception_flag = ExceptionFinalized;
#define EndTry \
                if (Exception_flag == ExceptionEntered) ntyExceptionPopStack; \
            } if (Exception_flag == ExceptionThrown) {ReThrow;} \
            } while (0)

void ntyExceptionThrow(ntyException *excep, const char *func, const char *file, int line, const char *cause, ...) {
    va_list ap;
    ntyExceptionFrame *frame = (ntyExceptionFrame *) ntyThreadLocalDataGet(ExceptionStack);

    if (frame) {
        //异常名
        frame->exception = excep;
        frame->func = func;
        frame->file = file;
        frame->line = line;
        //异常打印的信息
        if (cause) {
            va_start(ap, cause);
            vsnprintf(frame->message, EXCEPTION_MESSAGE_LENGTH, cause, ap);
            va_end(ap);
        }

        ntyExceptionPopStack;

        longjmp(frame->env, ExceptionThrown);
    }
        //没有被catch,直接throw
    else if (cause) {
        char message[EXCEPTION_MESSAGE_LENGTH + 1];

        va_start(ap, cause);
        vsnprintf(message, EXCEPTION_MESSAGE_LENGTH, cause, ap);
        va_end(ap);
        printf("%s: %s\n raised in %s at %s:%d\n", excep->name, message, func ? func : "?", file ? file : "?", line);
    }
    else {
        printf("%s: %p\n raised in %s at %s:%d\n", excep->name, excep, func ? func : "?", file ? file : "?", line);
    }
}


/* ** **** ******** **************** debug **************** ******** **** ** */

ntyException A = {"AException"};
ntyException B = {"BException"};
ntyException C = {"CException"};
ntyException D = {"DException"};

void *thread(void *args) {
    pthread_t selfid = pthread_self();
    Try
            {
                Throw(A, "A");
            }
        Catch (A)
            {
                printf("catch A : %ld\n", selfid);
            }
    EndTry;

    Try
            {
                Throw(B, "B");
            }
        Catch (B)
            {
                printf("catch B : %ld\n", selfid);
            }
    EndTry;

    Try
            {
                Throw(C, "C");
            }
        Catch (C)
            {
                printf("catch C : %ld\n", selfid);
            }
    EndTry;

    Try
            {
                Throw(D, "D");
            }
        Catch (D)
            {
                printf("catch D : %ld\n", selfid);
            }
    EndTry;

    Try
            {
                Throw(A, "A Again");
                Throw(B, "B Again");
                Throw(C, "C Again");
                Throw(D, "D Again");
            }
        Catch (A)
            {
                printf("catch A again : %ld\n", selfid);
            }
        Catch (B)
            {
                printf("catch B again : %ld\n", selfid);
            }
        Catch (C)
            {
                printf("catch C again : %ld\n", selfid);
            }
        Catch (D)
            {
                printf("catch B again : %ld\n", selfid);
            }
    EndTry;
}


#define PTHREAD_NUM        8

int main(void) {
    ntyExceptionInit();

    printf("\n\n=> Test1: Throw\n");
    {
        Throw(D, NULL);     //ntyExceptionThrow(&(D), "_function_name_", "_file_name_", 202, ((void *) 0), ((void *) 0))
        Throw(C, "null C"); //ntyExceptionThrow(&(C), "_function_name_", "_file_name_", 203, "null C", ((void *) 0))
    }
    printf("=> Test1: Ok\n\n");


    printf("\n\n=> Test2: Try-Catch Double Nesting\n");
    {
        Try
                {
                    Try
                            {
                                Throw(B, "call B");
                            }
                        Catch (B)
                            {
                                printf("catch B \n");
                            }
                    EndTry;
                    Throw(A, NULL);
                }
            Catch(A)
                {
                    printf("catch A \n");
                    printf("Result: Ok\n");
                }
        EndTry;
    }
    printf("=> Test2: Ok\n\n");


    printf("\n\n=> Test3: Try-Catch Triple  Nesting\n");
    {
        Try
                {
                    Try
                            {

                                Try
                                        {
                                            Throw(C, "call C");
                                        }
                                    Catch (C)
                                        {
                                            printf("catch C\n");
                                        }
                                EndTry;
                                Throw(B, "call B");
                            }
                        Catch (B)
                            {
                                printf("catch B\n");
                            }
                    EndTry;
                    Throw(A, NULL);
                }
            Catch(A)
                {
                    printf("catch A\n");
                }
        EndTry;
    }
    printf("=> Test3: Ok\n\n");


    printf("=> Test4: Test Thread-safeness\n");
    int i = 0;
    pthread_t th_id[PTHREAD_NUM];

    for (i = 0; i < PTHREAD_NUM; i++) {
        pthread_create(&th_id[i], NULL, thread, NULL);
    }

    for (i = 0; i < PTHREAD_NUM; i++) {
        pthread_join(th_id[i], NULL);
    }
    printf("=> Test4: Ok\n\n");

    printf("\n\n=> Test5: No Success Catch\n");
    {
        Try
                {
                    Throw(A, "no catch A ,should Rethrow");
                }
        EndTry;
    }
    printf("=> Test5: Rethrow Success\n\n");

    printf("\n\n=> Test6: Normal Test\n");
    {
        Try
                {
                    Throw(A, "call A");
                }
            Catch(A)
                {
                    printf("catch A\n");

                }
            Finally
                {
                    printf("wxf nb\n");
                };
        EndTry;
    }
    printf("=> Test6: ok\n\n");

}

void all() {
    ///try
    do {
        volatile int Exception_flag;
        ntyExceptionFrame frame;
        frame.message[0] = 0;
        frame.next = (ntyExceptionFrame *) pthread_getspecific((ExceptionStack));
        pthread_setspecific((ExceptionStack), (&frame));
        Exception_flag = _setjmp(frame.env);
        if (Exception_flag == ExceptionEntered) {
            ///
            {
                ///try
                do {
                    volatile int Exception_flag;
                    ntyExceptionFrame frame;
                    frame.message[0] = 0;
                    frame.next = (ntyExceptionFrame *) pthread_getspecific((ExceptionStack));
                    pthread_setspecific((ExceptionStack), (&frame));
                    Exception_flag = _setjmp(frame.env);
                    if (Exception_flag == ExceptionEntered) {
                        ///
                        {
                            ///Throw(B, "recall B"); --->longjmp ExceptionThrown
                            ntyExceptionThrow(&(B), "_function_name_", "_file_name_", 302, "recall B", ((void *) 0));
                            ///
                        }
                        ///Catch (B)
                        if (Exception_flag == ExceptionEntered)
                            ntyExceptionPopStack;
                    }
                    else if (frame.exception == &(B)) {
                        Exception_flag = ExceptionHandled;
                        ///
                        {    ///
                            printf("recall B \n");
                            ///
                        }
                        fin
                        if (Exception_flag == ExceptionEntered)
                            ntyExceptionPopStack;
                        if (Exception_flag == ExceptionEntered)
                            Exception_flag = ExceptionFinalized;
                        /{
                        {
                            printf("fin\n");
                        };
                        }
                        ///EndTry;
                        if (Exception_flag == ExceptionEntered)
                            ntyExceptionPopStack;
                    }
                    if (Exception_flag == ExceptionThrown)
                        ntyExceptionThrow(frame.exception, frame.func, frame.file, frame.line, ((void *) 0));
                } while (0);
                ///Throw(A, NULL); longjmp ExceptionThrown
                ntyExceptionThrow(&(A), "_function_name_", "_file_name_", 329, ((void *) 0), ((void *) 0));
                ///
            }
            ///Catch(A)
            if (Exception_flag == ExceptionEntered)
                ntyExceptionPopStack;
        }
        else if (frame.exception == &(A)) {
            Exception_flag = ExceptionHandled;
            ///
            {
                ///
                printf("\tResult: Ok\n");
                ///
            }
            /// EndTry;
            if (Exception_flag == ExceptionEntered)
                ntyExceptionPopStack;
        }
        if (Exception_flag == ExceptionThrown)
            ntyExceptionThrow(frame.exception, frame.func, frame.file, frame.line, ((void *) 0));
    } while (0);
    ///
}
```

原文地址：https://zhuanlan.zhihu.com/p/550202898

作者：linux
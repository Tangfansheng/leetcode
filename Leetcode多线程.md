### Leetcode多线程

#### 交替打印FooBar

```java
//初始版本 
//超时 只运行到了第70个用例
class FooBar {
    private int n;

    public FooBar(int n) {
        this.n = n;
    }
    volatile boolean isFoo = true;

    public void foo(Runnable printFoo) throws InterruptedException {
        
        for (int i = 0; i < n; i++) {
            
        	// printFoo.run() outputs "foo". Do not change or remove this line.
            while(!isFoo){}
        	printFoo.run();
            isFoo =!isFoo ;
        }
    }

    public void bar(Runnable printBar) throws InterruptedException {
        
        for (int i = 0; i < n; i++) {
            while(isFoo){}
            // printBar.run() outputs "bar". Do not change or remove this line.
        	printBar.run();
            isFoo = !isFoo;
        }
    }
}
```

只要用到while这种等待，首先肯定的是内存占用会飙高。

```java
//synchronized实现
class FooBar {
    private int n;

    public FooBar(int n) {
        this.n = n;
    }
    Object obj = new Object();
    boolean isFoo = true;

    public void foo(Runnable printFoo) throws InterruptedException {
        
        for (int i = 0; i < n; i++) {
            
        	// printFoo.run() outputs "foo". Do not change or remove this line.
            synchronized(obj){
                if(!isFoo) obj.wait();
                printFoo.run();
                isFoo = false;
                obj.notifyAll();
            }
        
        }
    }

    public void bar(Runnable printBar) throws InterruptedException {
        
        for (int i = 0; i < n; i++) {
           synchronized(obj){
                if(isFoo) obj.wait();
                printBar.run();
                isFoo = true;
                obj.notifyAll();
            }
        }
    }
}
```

改用synchronize锁来实现互斥，然后加入判断。能过但是内存消耗还是很高，耗时能接受。

<img src="D:\ProgramData\MyNotes\leetcode\Leetcode\foobar.png" alt="synchronized" style="zoom:80%;" />

```java
//用重入锁试试
class FooBar {
    private int n;
    public ReentrantLock lock = new ReentrantLock();
    public Condition fooCondition = lock.newCondition();
    public Condition barCondition = lock.newCondition();

    public FooBar(int n) {
        this.n = n;
    }
    boolean isFoo = true;

    public void foo(Runnable printFoo) throws InterruptedException {
        
        for (int i = 0; i < n; i++) {
            
        	// printFoo.run() outputs "foo". Do not change or remove this line.
            try{
                lock.lock();
                if(!isFoo) fooCondition.await();
                printFoo.run();
                isFoo = false;
                barCondition.signal();
            }catch(InterruptedException e){

            }finally{
                lock.unlock();
            }
        
        }
    }

    public void bar(Runnable printBar) throws InterruptedException {
        
        for (int i = 0; i < n; i++) {
           try{
                lock.lock();
                if(isFoo) barCondition.await();
                printBar.run();
                isFoo = false;
                fooCondition.signal();
            }catch(InterruptedException e){
                
            }finally{
                lock.unlock();
            }
        }
    }
}
```

没有明显的优化。

<img src="D:\ProgramData\MyNotes\leetcode\Leetcode\reentrantLock.png" alt="image-20200609223355051" style="zoom:80%;" />

尝试了一下内存消耗最高和耗时最短的，其实相差不大。



#### 打印零与奇偶数



```java
//我的没过
class ZeroEvenOdd {

    private int n;
    ReentrantLock lock = new ReentrantLock();
    Condition zeroP = lock.newCondition();
    Condition evenP = lock.newCondition();
    Condition oddP = lock.newCondition();
    int z = 1;//正整数
    int num = 0;//0的个数 
    int total = 0;//总数

    public ZeroEvenOdd(int n) {
        this.n = n;
    }

    // printNumber.accept(x) outputs "x", where x is an integer.
    public void zero(IntConsumer printNumber) throws InterruptedException {
        try{
            lock.lock();
            while(num < n){
                if(total%2!=0){
                    zeroP.await();
                }
                printNumber.accept(0);
                num++;
                total++;
                oddP.signal();
                evenP.signal();
            }
        }finally{
            lock.unlock();
        }
    }

    public void even(IntConsumer printNumber) throws InterruptedException {
         try{
            while(num <= n){
                lock.lock();
                if(z%2!=0){
                    evenP.await();
                }
                printNumber.accept(z++);
                total++;
                if(num != n)
                    zeroP.signal();
            }
        }finally{
            lock.unlock();
        }
    }



    public void odd(IntConsumer printNumber) throws InterruptedException {
        try{
            while(num<=n){
                lock.lock();
                if(z%2==0){
                    oddP.await();
                }
                printNumber.accept(z++);
                total++;
                if(num != n)
                    zeroP.signal();
            }
            
        }finally{
            lock.unlock();
        }
    }
}
```

**我的方法错误点：**

1不应该用0的个数来控制打印，因为数字在零的后面。会出现打印了0，但不打印整数的结果；

2.没有意识到让循环结束的问题。最终even和odd总会有一个在await状态，导致超时。



##### 他山之石 可以攻玉

```java
//跟我的思路差不多 但是比较好的解决的上面的两个问题
class ZeroEvenOdd {

    private int n;
    ReentrantLock lock = new ReentrantLock();
    Condition zeroP = lock.newCondition();
    Condition evenP = lock.newCondition();
    Condition oddP = lock.newCondition();
    volatile int z = 1;//一是代表次数，二是代表当前应该打印的整数 
    volatile int turn = 0;//j

    public ZeroEvenOdd(int n) {
        this.n = n;
    }

    // printNumber.accept(x) outputs "x", where x is an integer.
    public void zero(IntConsumer printNumber) throws InterruptedException {
         lock.lock();
        try{
            while(z <=n){
                if(turn!=0){
                    zeroP.await();
                }
                printNumber.accept(0);
                if(z % 2 ==0){
                    turn = 2;
                    evenP.signal();
                }
                else{
                    turn = 1;
                    oddP.signal();
                }
                //z可能没有来得及变化 使得又一次进入循环 因此要再次await
                zeroP.await();
            }
            //如果不加这两行 even或者odd其中之一会在循环中一直等待。
             oddP.signal();
             evenP.signal();
        }finally{
            lock.unlock();
        }
    }

    public void even(IntConsumer printNumber) throws InterruptedException {
        lock.lock();
         try{
            while(z <= n){
                if(turn!=2){
                    evenP.await();
                }else{
                    printNumber.accept(z++);
                    turn = 0;
                    zeroP.signal();                    
                 }
            }
        }finally{
            lock.unlock();
        }
    }



    public void odd(IntConsumer printNumber) throws InterruptedException {
        lock.lock();
        try{
            
            while(z <= n){
                if(turn!=1){
                    oddP.await();
                }else{
                    printNumber.accept(z++);
                    turn = 0;
                    zeroP.signal();           
                }
            }  
        }finally{
            lock.unlock();
        }
    }
}
```

##### 信号量解法

```java
class ZeroEvenOdd {
    private int n;
    Semaphore s1, s2, s3;
    
    public ZeroEvenOdd(int n) {
        this.n = n;
       	s1 = new Semaphore(1);//一开始就允许进入
        s2 = new Semaphore(0);//一开始直接阻塞
        s3 = new Semaphore(0);
    }

    // printNumber.accept(x) outputs "x", where x is an integer.
    public void zero(IntConsumer printNumber) throws InterruptedException {
        for (int i = 0; i < n; i++) {
            s1.acquire();
            printNumber.accept(0);
            if (i % 2 == 0) {
                s3.release();
            } else {
                s2.release();
            }
        }
    }

    public void even(IntConsumer printNumber) throws InterruptedException {
        for (int i = 2; i <= n; i += 2) {
            s2.acquire();
            printNumber.accept(i);
            s1.release();
        }
    }

    public void odd(IntConsumer printNumber) throws InterruptedException {
        for (int i = 1; i <= n; i += 2) {
            s3.acquire();
            printNumber.accept(i);
            s1.release();
        }
    }
}
```

##### 只用加锁同步的方式：

```java
class ZeroEvenOdd {
    private int n;
    private boolean zero = true;
    int i = 1;

    public ZeroEvenOdd(int n) {
        this.n = n;
    }

    public void zero(IntConsumer printNumber) throws InterruptedException {
        while (true) {
            synchronized (this) {
                while (!zero) {
                    if(i>n) {
                        return;
                    }
                    wait();
                }
                if(i>n) {
                    return;
                }
                printNumber.accept(0);
                zero = false;
                notifyAll();
                
            }
        }

    }

    public void even(IntConsumer printNumber) throws InterruptedException {
        while (true) {
            synchronized (this) {
                while (zero || i % 2 != 0) {
                    if(i>n) {
                        return;
                    }
                    wait();
                }
                printNumber.accept(i);
                i++;
                zero = true;
                notifyAll();
                
            }
        }

    }

    public void odd(IntConsumer printNumber) throws InterruptedException {
        while (true) {
            synchronized (this) {
                while (zero || i % 2 == 0) {
                    if(i>n) {
                        return;
                    }
                    wait();
                }
                printNumber.accept(i);
                i++;
                zero = true;
                notifyAll();
                
            }
        }

    }

}
```


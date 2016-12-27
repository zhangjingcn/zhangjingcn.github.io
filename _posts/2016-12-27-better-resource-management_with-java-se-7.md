---
layout: post
title:  "[译]Better Resource Management with Java SE 7: Beyond Syntactic Sugar"
date:   2016-10-26 19:54:11 +0800
categories: java
---

# [译]Better Resource Management with Java SE 7: Beyond Syntactic Sugar

原作者：Julien Ponge   
原文地址：http://www.oracle.com/technetwork/articles/java/trywithresources-401775.html

这篇文章表述了java 7解决自动资源管理问题的思路，翻译的时候尽量简化，因为作者废话太多。
## 简介

一个典型的java程序会处理包括文件、流、socket、数据库连接等多种类型的资源。这些资源都需要占用实际的系统资源，所以我们在处理时候需要格外小心，需要确保即使是发生异常的时候，这些资源都能够被正确释放。经常会有数据库连接或文件流由于异常而没有正确关闭，导致系统资源用完的情况。

现在比较常用的资源管理方法大家都比较清楚了，初始化一个资源后，就要调用这个资源的close方法，一般会使用try/catch/finally块来确保最后一定能调用到close方法。但是即使这样，还是有时候会出现资源未正确关闭的情况，引起一些问题。

这篇文章表述了java 7解决自动资源管理问题的思路，这个思路最早是由OpenJdk的Project Coin提出的，叫做try-with-resources表达式. 这种方法类似foreach的语法糖的方案，但是又不仅仅是增加一个语法糖，而且还解决了异常覆盖的问题。

## try-with-resources表达式

以前的try/catch/finally块会导致在业务在代码里有很多“boilerplate code”， 类似这样：

```
	private void correctWriting() throws IOException {
       DataOutputStream out = null;
       try {
           out = new DataOutputStream(new FileOutputStream("data"));
           out.writeInt(666);
           out.writeUTF("Hello");
       } finally {
           if (out != null) {
               out.close();
           }
       }        
   }
```

使用try-with-resources后，会写成下面这个样子：

```
	private void writingWithARM() throws IOException {
       try (DataOutputStream out 
               = new DataOutputStream(new FileOutputStream("data"))) {
           out.writeInt(666);
           out.writeUTF("Hello");
       }
   }
```

try-with-resources有几点要注意的：

1. 在try block结束之后，无论是正常结束还是异常退出，都会执行资源的close方法以关闭资源；
2. 如果是在try里面定义多个资源，每个定义语句之间用分号分隔。
3. 在这样的try后面加catch和finally也是和以前一样的。   
4. 在try-with-resources statement结束时会依据资源的声明顺序反向调用close方法进行关闭。

## Auto-Closeable类

只有实现了java.lang.AutoCloseable接口的类才能在try-with-resources中自动释放，这个接口有一个close方法需要资源类去实现。java库中的好多资源类都实现了这个接口，包括java.io、java.nio、javax.crypto、java.security、java.util.zip、java.util.jar、javax.net和java.sql packages。

下面再看一个例子，

```
	public class AutoClose implements AutoCloseable {    
       
       @Override
       public void close() {
           System.out.println(">>> close()");
           throw new RuntimeException("Exception in close()");
       }
       
       public void work() throws MyException {
           System.out.println(">>> work()");
           throw new MyException("Exception in work()");
       }
       
       public static void main(String[] args) {
           try (AutoClose autoClose = new AutoClose()) {
               autoClose.work();
           } catch (MyException e) {
               e.printStackTrace();
           }
       }
   }
	class MyException extends Exception {
       
       public MyException() {
           super();
       }
       
       public MyException(String message) {
           super(message);
       }
   }
```

这个例子中，work方法和close方法都会抛出异常，最后在main方法中打印出的异常堆栈是：

```
>>> work()
   >>> close()
   MyException: Exception in work()
          at AutoClose.work(AutoClose.java:11)
          at AutoClose.main(AutoClose.java:16)
          Suppressed: java.lang.RuntimeException: Exception in close()
                 at AutoClose.close(AutoClose.java:6)
                 at AutoClose.main(AutoClose.java:17)
```

从这里可以看出，确实是先调用了close方法，然后再进入了catch语句，但是这里可以看到close语句中抛出的异常是被Suppressed， 关于这点，会在接下来的部分讲到。

## Exception Masking

我们来看下，如果用try-catch-finally会是什么样的情况：

```
	public static void runWithMasking() throws MyException {
       AutoClose autoClose = new AutoClose();
       try {
           autoClose.work();
       } finally {
           autoClose.close();
       }
   }

	public static void main(String[] args) {
       try {
           runWithMasking();        
       } catch (Throwable t) {
           t.printStackTrace();
       }
   }
```

输出结果如下：

```
>>> work()
   >>> close()
   java.lang.RuntimeException: Exception in close()
          at AutoClose.close(AutoClose.java:6)
          at AutoClose.runWithMasking(AutoClose.java:19)
          at AutoClose.main(AutoClose.java:52)
```

这里展示的来的是信息是close()抛出的异常，而我们知道，work()函数也抛出来异常，实际上我们也更关心work()抛出的异常，而work()中抛出的异常信息却没有展示出来。出现这样的原因是只能有一个异常被抛出来，我们虽然也可以通过getCause获取nested Exception来得到work()中抛出的异常信息，但总的来说，还是是有点不优雅的。这叫做Exception Masking。

## Supporting "Suppressed" Exceptions

通过上一段我们可以看到Exception Masking对于我们的实际使用，是存在一些不方便的地方。所以Java 7中引入了一个叫做“suppressed” exceptions， 这个“suppressed” exceptions能够挂到主异常上。 Java 7在java.lang.Throwable中增加了两个方法：

```
//appends a suppressed exception to another one, so as to avoid exception masking.
public final void addSuppressed(Throwable exception) 

//gets the suppressed exceptions that were added to an exception.
public final Throwable[] getSuppressed() 
```

这两个方法就是专门为try-with-resources而引入，解决Exception Masking问题而设计的。如果不通过try-with-resources，自己也是可以自己写代码来实现这样的“suppressed” exceptions，只是会比较麻烦，大概是下面这样：

```
	public static void runWithoutMasking() throws MyException {
       AutoClose autoClose = new AutoClose();
       MyException myException = null;
       try {
           autoClose.work();
       } catch (MyException e) {
           myException = e;
           throw e;
       } finally {
           if (myException != null) {
               try {
                   autoClose.close();
               } catch (Throwable t) {
                   myException.addSuppressed(t);
               }
           } else {
               autoClose.close();
           }
       }
   }
```

当work()方法抛出异常后，在catch中使用一个变量保存下当前异常，在finally中判断如果这个变量不为空，就为close()方法增加try-catch块，当close()抛出异常进入catch块后，将close()抛出的异常挂到主异常上，就完成了。

## Syntantic Sugar Demystified

使用try-with-resources方法也能实现相同的效果，如下面的代码：

```
	public static void runInARM() throws MyException {
       try (AutoClose autoClose = new AutoClose()) {
           autoClose.work();
       }
   }
```

代码经过编译后，再通过反编译工具JD-GUI转成java代码，来看一下：

```
	public static void runInARM() throws MyException {
       AutoClose localAutoClose = new AutoClose();
       Object localObject1 = null;
       try {
           localAutoClose.work();
       } catch (Throwable localThrowable2) {
           localObject1 = localThrowable2;
           throw localThrowable2;
       } finally {
           if (localAutoClose != null) {
               if (localObject1 != null) {
                   try {
                       localAutoClose.close();
                   } catch (Throwable localThrowable3) {
                       localObject1.addSuppressed(localThrowable3);
                   }
               } else {
                   localAutoClose.close();
               }
           }
       }
   }
```

基本上和我们刚在自己手写的代码是一样的了。try-with-resources表达式，自己就实现了“suppressed” exceptions的功能，简化了我们很多事情。而且如果try-with-resources中定义了多个资源，这个简化的意义就更大。让我们看一个在try块中定义多个资源的例子：

```
	private static void compress(String input, String output) throws IOException {
       try(
           FileInputStream fin = new FileInputStream(input);
           FileOutputStream fout = new FileOutputStream(output);
           GZIPOutputStream out = new GZIPOutputStream(fout)
       ) {
           byte[] buffer = new byte[4096];
           int nread = 0;
           while ((nread = fin.read(buffer)) != -1) {
               out.write(buffer, 0, nread);
           }
       }
   }
```
在这个例子中，try块定义了三个资源，通过反编译生成的代码就会非常复杂，那么有没有办法简化一下， 是有的，比如这样：

```
	private static void compress(String input, String output) throws IOException {
       try(
           FileInputStream fin = new FileInputStream(input);
           GZIPOutputStream out = new GZIPOutputStream(new FileOutputStream(output))
       ) {
           byte[] buffer = new byte[4096];
           int nread = 0;
           while ((nread = fin.read(buffer)) != -1) {
               out.write(buffer, 0, nread);
           }
       }
   }
```

这样做确实简化了，但也引入了一个问题，比如在执行out.close()时，可能会发生异常，导致GZIPOutputStream包装的FileOutputStream的close()方法没有正确执行，引起泄露。而上一段代码就没这个问题，因为FileOutputStream fout是单独定义出来的，会单独调用fout.close()。

## 总结

try-with-resources不仅实现了资源的自动关闭，也通过实现"Suppressed" Exceptions解决了Exception Masking的问题。
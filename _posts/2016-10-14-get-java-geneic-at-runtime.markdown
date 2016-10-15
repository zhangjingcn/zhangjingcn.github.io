---
layout: post
title:  "在运行期获取java泛型类型的方法"
date:   2016-10-14 10:23:39 +0800
categories: java
---

## 问题的来由 ##

Google的[Guice](https://github.com/google/guice "Guice")是一个轻量级的DI框架，在进行依赖关系的解析时，能自动将一个模型类T与它的模型提供者Provider<T>进行关联，也就是Guice能够解析Provider<T>中的泛型类型T。学习了一下源码，为了在运行时获取泛型的类型，Guice提供了TypeLiteral<T>类，继承了TypeLiteral<T>的类可以获取到其泛型类型。

## 学习的过程 ##

而这与我对泛型的理解是不一样的，以前学到的是泛型信息在编译期会擦除，在运行时是无法获取到泛型类型的，这主要是为了实现向后兼容，不管一个类型Class<T>，不管它的泛型类型是什么，在运行时它都是同一个类型，比如说下面的表达式是成立的。

    assert new ArrayList<String>().getClass() == new ArrayList<Integer>().getClass();
    
而实际上在一种情况下，java提供了在运行期获取泛型类型的方法，如下面的测试代码:

    public class GenericTest {
    	class MyGenericClass<T> { }
    	class MyStringSubClass extends MyGenericClass<String> { }

    	public static void main(String[] args) {
        	System.out.print(((ParameterizedType) MyStringSubClass.class
            	.getGenericSuperclass()).getActualTypeArguments()[0]);
    	}

	}

上面的测试代码执行后返回“class java.lang.String”，实现了获取泛型类型。MyGenericClass类有一个泛型T，MyStringSubClass继承了MyGenericClass并赋值了T=String。在这种情况下， java编译器能够在MyStringSubClass的字节码中记录下它继承的MyGenericClass类的泛型类型String。之所以这种情况下编译器能够记录下泛型信息，是因为MyStringSubClass能够确定T=String,不需要擦除泛型信息来实现向后兼容。

所以回到Guice的例子中，Guice通过TypeLiteral<T>类，继承TypeLiteral<T>的子类都可以通过这种方法获取到继承基类TypeLiteral时指定的泛型类型。

Java API提供了Class.getGenericSuperclass方法返回Type表示其直接超类，如果这个超类是一个泛型类，那么返回类型实际是ParameterizedType，其getActualTypeArguments方法以数组的形式存储了其多个泛型参数。

## gentyref库 ##

虽然可以用上面的代码自己实现，但google提供了一个[gentyref](http://code.google.com/p/gentyref "gentyref")库帮助我们实现了获取泛型类型的功能，下面是一段测试代码：

    public class GentyrefTest {
    	public <T> void chill(List<T> aListWithTypeT) {
        	System.out.println(GenericTypeReflector
            	.getTypeParameter(
            	aListWithTypeT.getClass(),Collection.class.getTypeParameters()[0]));
    	}

    	public void chillWildcard(List<?> aListWithTypeWildcard) {
    		System.out.println(
    		GenericTypeReflector.getTypeParameter(
    		aListWithTypeWildcard.getClass(),Collection.class.getTypeParameters()[0]));
    }

    	public static void main(String... args) {
        	GentyrefTest test = new GentyrefTest();
        	test.chill(new ArrayList<String>(){});
        	test.chillWildcard(new ArrayList<Integer>(){});
    	}
	}

这段代码的返回值是<code>class java.lang.String class java.lang.Integer</code>
其中需要注意的是，其原理和刚才分析的是一样的，所以不能直接使用<code>new ArrayList<String>()</code>作为函数的参数，而是要像例子中一样，创建其子类，如<code>new ArrayList<String>(){}</code>.

## 总结 ##

通过这个问题学习到了一种在运行期能够获取泛型类型的方法，也加深了对泛型的理解。
---
title: Android 单元测试入门
date: 2017-06-23 16:16:34
tags:
- Android
- 单元测试
- Unit Test
- JUnit
- Mockito
- mock
---

正在学习 Android 单元测试 ，光看文章怕有理解偏差，所以写一下博客帮助理解，如果能有人给我反馈建议或者错误就更好了。这篇文章主要讲到 *JUnit* 和 *Mockito* 这两个框架的使用，有了它们就已经可以跑绝大多数的单元测试了。

<!-- more -->

## JUnit

> 使用断言（Assertion）对 **带返回值** 的方法进行测试。  

### @Test

使用 `@Test` 标注的方法为测试方法，JUnit 会自动执行这些方法。
理论上测试方法的名字是可以随便取的，但为了方便阅读，应该在要测试的方法名前加上 `test` 来命名，比如方法 `add()` 的测试方法应该叫 `testAdd()` 。

除了常规的返回值测试， `@Test` 还可以用来测试异常和超时，举例来说：

- `@Test(expected=NullPointerException.class)`
  这样声明之后，只有当方法内抛出了空指针异常，才会认为测试通过。
- `@Test(timeout=3000)`
  这会给测试方法一个超时时间，在上面这个例子中，如果方法执行超过了3秒，方法会被系统强行终止，并汇报终止的原因是超时。

### 其他注解

有一些常用的注解可以简化一下测试代码：

- `@Before`
  这个注解标识的方法会在每一个测试方法前被调用，通常用来做一些初始化的工作。
- `@After`
  对应的，这个方法会在每一个测试方法后调用，用来做资源释放的工作。
- `@Ignore`
  忽略某个测试方法，通常用于真正的方法还没有实现而你又正在写测试用例的时候。这样测试结果会显示有几个方法被忽略，而不是直接让测试失败。
- `@BeforeClass`
- `@AfterClass`
  跟开头的两个注解类似，不过这两个是作用于整个测试类的，可以将它们想象成测试对象创建和释放时会调用的方法。与之前不同的是，这两个注解修饰的方法必须是 `public static` 的。
- `@RunWith` & `@Parameter`
  [Test runners · junit-team/junit4 Wiki · GitHub](https://github.com/junit-team/junit4/wiki/Test-runners)

### 举个栗子

```java
public class CalculatorTest {
    private Calculator mCalculator;

    @Before
    public void setUp() throws Exception {
        mCalculator = new Calculator();
    }

    @Test
    public void testSum() throws Exception {
        //expected: 6, sum of 1 and 5, tolerance 0
        assertEquals(6, mCalculator.sum(1, 5), 0);
    }

    @Test
    public void testSubstract() throws Exception {
        assertEquals(1, mCalculator.substract(5, 4), 0);
    }
} 
```

> 节选自[在Android Studio中进行单元测试和UI测试 - 简书](http://www.jianshu.com/p/03118c11c199)  

### @Rule

在 JUnit 4.7 版本开始，增加了一个新特性：Rule。

当我们使用 `@Before` 这类注解时，可以在测试方法执行前运行一些初始化的代码，然而如果这个初始化需要在多个测试类中被用到，比如初始化一个 *ContextHolder* ，还是要分别在每一个测试类的 `@Before` 方法中都写一遍。这就到了 Rule 发挥的时候了。

JUnit 提供了一些现成的 Rule，借用 InfoQ 上看到的一篇[博客](http://www.infoq.com/cn/news/2009/07/junit-4.7-rules)的截图：
{% img center /uploads/android-unit-test/test_rule.png Test Rules %}

简单来说，这些类继承自一个叫 `TestRule` 的类，告诉了 JUnit 我们定义了一些测试规则，并希望在我们初始化这些规则对象的测试类中，所有测试方法都要满足这些规则，否则测试失败。

既然如此，如果我们创建一个类继承 `TestRule`，也就可以定义我们自己的测试规则了，具体的方法可以看[这篇文章](https://segmentfault.com/a/1190000005923632)

### 小结

按照上文的叙述，这些注解修饰的方法会以下面这个顺序调用：
`@BeforeClass` -> `@Before` -> `@Test` -> `@After` -> `@AfterClass`

回到这个段落的开头：
{% img center /uploads/android-unit-test/junit.png JUnit %}
因为最终的测试工作落在了断言上，所以我们只能对数值的正确与否进行测试。

这在大多数情况下是足够了，然而有时候我们要测试的类可能会依赖许多其他的类：

```java
class ToBeTest {
	Class notTested;
	Class alsoNotTested;
	Class stillNotTested;

	@Test
	public void testMethod() {
		notTested.doSomething();
		alsoNotTested.doSomething();
		stillNotTested.doSomething();
	}
}
```

又或者，如果我们要验证的方法并没有返回值，怎么知道这个方法的正确性呢？
这时候我们需要了解一种叫 mock 的测试方式。

## Mockito

> Mock 测试：对不容易构建的对象，用一个虚拟对象来替代测试的方法  

简单理解，Mock 对象其实就是一个虚拟的对象，我们完全可以手动创建一个，并让它模拟真实对象的行为，从而达到测试的目的。一些 mock 测试框架（比如 Mockito）已经给我们提供了很好的工具，我们可以直接用它们来创建 mock 对象。

{% img center /uploads/android-unit-test/mock.png Mock %}
配图来源：[Unit tests with Mockito - Tutorial](http://www.vogella.com/tutorials/Mockito/article.html)

### 创建 mock 对象

我们有两种方法去创建一个 mock 对象，举例来说，它们分别长这样：

```java
// 方法一
TestClass tc = Mockito.mock(TestClass.class);

// 方法二
@Mock
TestClass tc; // 1
@Rule public MockitoRule mockitoRule = MockitoJUnit.rule(); //2
```

方法一就是让 Mockito 帮我们创建了一个指定类型的 mock 对象，方法二把这个过程分成了两步：

1. 通过 `@Mock` 注解声明我们要 mock 的对象
2. 使用 JUnit Rule 的方式来对这些对象进行了一次全局的配置，这一步类似于 dagger 里 `inject()` 的调用

经过 mock 之后的对象，

### 举个栗子

Mockito 更像是 JUnit 的一种补充，所以一般都会搭配 JUnit 使用，借用一个例子（来源 [Unit tests with Mockito - Tutorial](http://www.vogella.com/tutorials/Mockito/article.html)）：

```java
public class MockitoTest  {

    @Mock
    MyDatabase databaseMock; 

    @Rule public MockitoRule mockitoRule = MockitoJUnit.rule(); 

    @Test
    public void testQuery()  {
        ClassToTest t  = new ClassToTest(databaseMock); 
        boolean check = t.query("* from t"); 
        assertTrue(check); 
        verify(databaseMock).query("* from t"); 
    }
}
```

在上面的代码里，我们用了 `@Mock` 注解的方式生成了一个 mock 对象 `databaseMock` 来模拟数据库，并将它作为参数构造了我们的 `ClassToTest`。
在 `testQuery()` 方法里，我们用了 JUnit 的  `assertTrue()` 的方式去判断数据库查询是否成功，然后使用 Mockito 的  `verify()` 去测试 `databaseMock.query()` 是否被调用了。

### 限制

Mockito 也有这它的限制：

- 不能 mock 静态方法
- 不能 mock 私有方法
- 不能 mock 构造方法
- 不能 mock `equals()` & `hashCode()`

> 更详细的约束项可以看官方的[文档](https://github.com/mockito/mockito/wiki/FAQ#what-are-the-limitations-of-mockito)  

这几条约束其实已经足够搞死人了。
在原本没有集成单元测试的项目中，会发现一些较低层级的代码几乎不可测试，但对于使用了 MVP 模式的模块，这种情况理应会有所改善。

### 小结

JUnit 搭配 Mockito 之后，你会发现写单元测试是一件挺轻松的事情，毕竟最麻烦的依赖问题已经基本解决了。
如果碰到一些实在没法下手的情况，比如待测方法里包含了太多的类方法，那就只能祭出大杀器——重构了。

## 写在最后

`static` 方法在开发过程中还是挺常见的，比如 `Log.d()` 什么的。因为实现方式的问题，Mockito 是不支持这些方法的，所以有一个叫 Powermock 的工具应运而生了，为了提高单元测试的可用性，我会尝试使用这个工具或者其他的框架，到时候看看能不能再输出一篇文章。

到此为止，Android 开发中纯 Java 部分的单元测试已经可以展开了，但涉及到 Android SDK 的部分还是没办法动的。鉴于现在的项目是基于 React Native 开发的，没有太多关于 Android SDK 的部分，所以就到此为止了。

换个角度想想，如果项目中有很多 Android SDK 相关的部分，而你又不想进行单元测试的话，是不是可以考虑说服技术经理往 React Native 上迁移呢？

## 参考文档

- [Getting Started with Testing | Android Developers](https://developer.android.com/training/testing/start/index.html)
- [JUnit](http://junit.org/junit4/)
- [Unit tests with Mockito - Tutorial](http://www.vogella.com/tutorials/Mockito/article.html)
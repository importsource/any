#Parameters

testng的另外一个亮点就是Parameters。

在单元测试的过程中。我们经常初始化一些数据。过去我们需要在每个方法中都要初始化一遍。现在通过testng的paramter，我们可以一次性初始化好数据。

支持两种方式来set我们的paramters：

* testng.xml 
* programmatically 


1 - 通过 testng.xml 设置参数：

如果你只是想传入一些简单的value，你可以在你的testng.xml中来定义：

```java
@Parameters({ "first-name" })
@Test
public void testSingleString(String firstName) {
  System.out.println("Invoked testString " + firstName);
  assert "Cedric".equals(firstName);
}
```

像这样：

```xml

<suite name="My suite">
  <parameter name="first-name"  value="Cedric"/>
  <test name="Simple example">
</suite>

```


这种做法同样适用于那些annotation方法，比如@Before/After and @Factory ：

```java
@Parameters({ "datasource", "jdbcDriver" })
@BeforeMethod
public void beforeTest(String ds, String driver) {
  m_dataSource = ...;                              // look up the value of datasource
  m_jdbcDriver = driver;
}
```



你可以使用Optional annotation把参数设置为可选：

```java
@Parameters("db")
@Test
public void testNonExistentParameter(@Optional("mysql") String db) { ... }
```


如果在xml文件中没有名称为db的参数，那么你的测试方法将会收到一个默认的value，这个默认值为@Optional中配置的"mysql"。

 @Parameters annotation可以被用在如下位置:

*  On any method that already has a @Test, @Before/After or @Factory annotation.
*  On at most one constructor of your test class.  In this case, TestNG will invoke this particular constructor with the parameters initialized to the values specified in testng.xml whenever it needs to instantiate your test class.  This feature can be used to initialize fields inside your classes to values that will then be used by your test methods.

Notes:

* The XML parameters are mapped to the Java parameters in the same order as they are found in the annotation, and TestNG will issue an error if the numbers don't match.
* Parameters are scoped. In testng.xml, you can declare them either under a <suite> tag or under <test>. If two parameters have the same name, it's the one defined in <test> that has precedence. This is convenient if you need to specify a parameter applicable to all your tests and override its value only for certain tests.
* 

2 - 带DataProviders的参数



在xml中初始化一些简单的数据也许并不能满足你的胃口。大部分时候你需要intial更复杂的数据。或者这些数据是来自一个property 文件或者db，等等。在这种情况下，你可以使用DataProvider 来初始化数据。一个DataProvider 其实就是你的类里边的一个方法，这个方法返回了一个数组而已。

This method is annotated with @DataProvider:

```java
//This method will provide data to any test method that declares that its Data Provider
//is named "test1"
@DataProvider(name = "test1")
public Object[][] createData1() {
 return new Object[][] {
   { "Cedric", new Integer(36) },
   { "Anne", new Integer(37)},
 };
}

 
//This test method declares that its data should be supplied by the Data Provider
//named "test1"
@Test(dataProvider = "test1")
public void verifyData1(String n1, Integer n2) {
 System.out.println(n1 + " " + n2);
}

```

打印结果：

```log
Cedric 36
Anne 37
```



默认的话，dataProvider会被在当前类或者父类中寻找。如果你想把你的数据provider放在其他类中，那么就需要是一个static的方法或者一个无参constructor，然后你再在测试类需要用到这个数据的地方通过dataProviderClass 属性来引入，像下面这样：

```java
public class StaticProvider {
  @DataProvider(name = "create")
  public static Object[][] createData() {
    return new Object[][] {
      new Object[] { new Integer(42) }
    };
  }
}
```
 在测试类中引入：
 ```java
public class MyTest {
  @Test(dataProvider = "create", dataProviderClass = StaticProvider.class)
  public void test(Integer n) {
    // ...
  }
}
```



 

data provider也支持injection。TestNG将会使用test context。

Data Provider 方法可以返回下列两种类型的数据：


* An array of array of objects (Object[][])。

*  and you don't want to create all of them upfront.
   这个和Object[][]唯一的不同点就是Iterator可以允许你延迟创建的你的测试数据。lazy pattern，you know。
TestNG将会调用这个iterator，然后带参数的那个测试方法就会通过iterator一个一个return 数据。

  *note*：这个在你需要给一个方法传递很多参数而你又不想一次性把所有的数据都创建好的情况下非常有用

下面是例子：

```java
@DataProvider(name = "test1")
public Iterator<Object[]> createData() {
  return new MyIterator(DATA);
}
```

DataProvider也支持反射，像下面这样可以直接获得方法名：

```java
@DataProvider(name = "dp")
public Object[][] createData(Method m) {
  System.out.println(m.getName());  // print test method name
  return new Object[][] { new Object[] { "Cedric" }};
}
 
@Test(dataProvider = "dp")
public void test1(String s) {
}
 
@Test(dataProvider = "dp")
public void test2(String s) {
}
```

打印结果:

```log
test1
test2
```

Data providers 也支持parallel ，像下面这样：
```java
@DataProvider(parallel = true)
// ...
```

Parallel data providers的默认线程池的size是10。你可以在xml中修改：

```xml
<suite name="Suite1" data-provider-thread-count="20" >
...

```


如果你想在不同的线程池不同的dataprovider，你需要通过运行不同的xml文件来实现。

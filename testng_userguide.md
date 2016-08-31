test-ng引入文档

##Add Maven dependency：

```xml
<dependency>
		<groupId>org.testng</groupId>
		<artifactId>testng</artifactId>
		<version>6.9.12</version>
		<scope>test</scope>
</dependency>
```
##Add Elicpse plugin:

Install from update site

1.Select Help / Install New Software...
2.Enter the update site URL in "Work with:" field:
    Update site for release: http://beust.com/eclipse.
    Or, Update site for beta: http://testng.org/eclipse-beta .
    
##Hello World
安装好plugin后，然后“File”==>"New"==>"Other"==>"TestNG",create TestNG class,like this:
```java
package com.importsource.util.test.testng;

import org.testng.annotations.Test;
import org.testng.Assert;
import org.testng.annotations.AfterMethod;
import org.testng.annotations.BeforeMethod;

public class NewTest {
  @Test
  public void f() {
    //invoke your biz logic function ,then assert it
	  System.out.println("中");
  }
  @BeforeMethod
  public void beforeMethod() {
	  System.out.println("前");
  }
  
  @AfterMethod
  public void afterMethod() {
	  System.out.println("后");
  }

}

```
然后右键 ，"run"==>"TestNG Test"

然后可以在console中和Results中查看。



##Convert Junit to TestNG:

右键junit测试类所在package，“TestNG”==>"Convert to TestNG". ALL is ok!

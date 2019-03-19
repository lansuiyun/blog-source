---
title: JMockit
date: 2016-07-20 17:01:54
tags: test
categories: java tool
---
JMockit 是用以帮助开发人员编写测试程序的一组工具和API，与`JUnit` `TestNG`配合使用。
<!--more-->
### Maven地址
```
<dependency>
    <groupId>org.jmockit</groupId>
    <artifactId>jmockit</artifactId>
    <version>1.25</version>
</dependency>
```

### 注解
- @Mocked：mock类及其父类，Object除外
- @Injectable：mock类的某个实例，其他实例不受影响
- @Capturing：mock类及其子类
- @Tested：自动实例化，并注入mocked依赖
- @Mock：MockUp模式中，指定被Fake的方法
### 常用类
- `Expectations`：期望，指定必须被调用的方法，返回值，调用次数
- `StrictExpectations`：严格期望，指定的方法必须按顺序被调用
- `Verifications`：验证
- `Invocation`：工具类，可以获取调用信息
- `Delegate`：根据参数指定返回值
- `MockUp`：模拟函数实现
- `Deencapsulation`：反射攻击类

### 模式
JMocket有两种mock模式，基于行为的mock方式和基于状态的mock方式

### 基于状态的mock方式(Faking)
基于状态的mock直接改写被mock类方法的逻辑
```
public static class MockLoginContextThatFailsAuthentication extends MockUp<LoginContext>
{
   @Mock
   public void $init(String name) {}
 
   @Mock
   public void login() throws LoginException
   {
      throw new LoginException();
   }
}
 
@Test(expected = LoginException.class)
public void applyingAnotherMockClass() throws Exception
{
   new MockLoginContextThatFailsAuthentication();
 
   // Inside an application class:
   new LoginContext("test").login();
}
```
除了fack `public` 方法外，`private`,`protected`,`package-private`,`static`,`final`,`native`方法都开源被fake。
被fake的方法需要有实现，所以`abstract`方法，`interface`的方法不能直接被fack

#### In-line mock-up classes
如果只是需要在一个单独的测试中face一个类，可以在测试方法内部定义一个匿名的`mock-up`类。
```
@Test
public void applyingAnAnonymousMockup() throws Exception
{
   new MockUp<LoginContext>() {
      @Mock void $init(String name) { /* do nothing */ }
      @Mock void login() {}
   });
 
   new LoginContext("test").login();
}
```

#### Faking an interface
```
@Test
public void fakingAnInterface() throws Exception
{
   CallbackHandler callbackHandler = new MockUp<CallbackHandler>() {
      @Mock
      void handle(Callback[] callbacks)
      {
         // fake implementation here
      }
   }.getMockInstance();
 
   callbackHandler.handle(new Callback[] {new NameCallback("Enter name:")});
```
` MockUp#getMockInstance() `方法返回一个实现了`CallbackHandler`接口的代理对象

#### Faking unspecified implementation classes
```
public interface Service { int doSomething(); }
final class ServiceImpl implements Service { public int doSomething() { return 1; } }
 
public final class TestedUnit
{
   private final Service service1 = new ServiceImpl();
   private final Service service2 = new Service() { public int doSomething() { return 2; } };
 
   public int businessOperation()
   {
      return service1.doSomething() + service2.doSomething();
   }
}
```
```
@Test
public <T extends Service> void fakingImplementationClassesFromAGivenBaseType()
{
   new MockUp<T>() {
      @Mock int doSomething() { return 7; }
   };
 
   int result = new TestedUnit().businessOperation();
 
   assertEquals(14, result);
}
```
在上边的测试中，所有实现了`Service#doSomething()`的方法调用都会被重定向到mock的方法。

#### Accessing the invocation context
mock方法时，可以将`mockit.Invocation`作为方法的第一个参数（附加参数，调用方法时无需传入）传入，当方法被执行的时候，`Invocation`对象会自动被传入。
`Invocation`提供了一些getter方法，可以在mock方法内部使用。其中一个是`getInvokedInstance()`，返回被mocked的对象。
```
@Test
public void accessingTheMockedInstanceInMockMethods() throws Exception
{
   final Subject testSubject = new Subject();
 
   new MockUp<LoginContext>() {
      @Mock
      void $init(Invocation invocation, String name, Subject subject)
      {
         assertNotNull(name);
         assertSame(testSubject, subject);
 
         // Gets the invoked instance.
         LoginContext loginContext = invocation.getInvokedInstance();
 
         // Verifies that this is the first invocation.
         assertEquals(1, invocation.getInvocationCount());
 
         // Forces setting of private Subject field, since no setter is available.
         Deencapsulation.setField(loginContext, subject);
      }
 
      @Mock
      void login(Invocation invocation)
      {
         // Gets the invoked instance.
         LoginContext loginContext = invocation.getInvokedInstance();
 
         // getSubject() returns null until the subject is authenticated.
         assertNull(loginContext.getSubject());
 
         // Private field set to true when login succeeds.
         Deencapsulation.setField(loginContext, "loginSucceeded", true);
      }
 
      @Mock
      void logout(Invocation invocation)
      {
         // Gets the invoked instance.
         LoginContext loginContext = invocation.getInvokedInstance();
 
         assertSame(testSubject, loginContext.getSubject());
      }
   };
 
   LoginContext theMockedInstance = new LoginContext("test", testSubject);
   theMockedInstance.login();
   theMockedInstance.logout();
}
```

#### Proceeding into the real implementation
@Mock方法一旦执行就会被从定向到mock方法。如果想执行mock方法的真实实现，可以将`Invocation`当做mock方法的第一个参数，并调用`proceed()`方法。
```
@Test
public void proceedIntoRealImplementationsOfFakedMethods() throws Exception
{
   // Create objects used by the code under test:
   LoginContext loginContext = new LoginContext("test", null, null, configuration);
 
   // Apply mock-ups:
   ProceedingMockLoginContext mockInstance = new ProceedingMockLoginContext();
 
   // Exercise the code under test:
   assertNull(loginContext.getSubject());
   loginContext.login();
   assertNotNull(loginContext.getSubject());
   assertTrue(mockInstance.loggedIn);
 
   mockInstance.ignoreLogout = true;
   loginContext.logout(); // first entry: do nothing
   assertTrue(mockInstance.loggedIn);
 
   mockInstance.ignoreLogout = false;
   loginContext.logout(); // second entry: execute real implementation
   assertFalse(mockInstance.loggedIn);
}
 
static final class ProceedingMockLoginContext extends MockUp<LoginContext>
{
   boolean ignoreLogout;
   boolean loggedIn;
 
   @Mock
   void login(Invocation inv) throws LoginException
   {
      try {
         inv.proceed(); // executes the real code of the mocked method
         loggedIn = true;
      }
      finally {
         // This is here just to show that arbitrary actions can be taken inside
         // the mock, before and/or after the real method gets executed.
         LoginContext lc = inv.getInvokedInstance();
         System.out.println("Login attempted for " + lc.getSubject());
      }
   }
 
   @Mock
   void logout(Invocation inv) throws LoginException
   {
      // We can choose to proceed into the real implementation or not.
      if (!ignoreLogout) {
         inv.proceed();
         loggedIn = false;
      }
   }
}
```

#### Reusing mock-ups between tests
如果需要在测试用例中重复使用`mock-up`类，可以将其定义到`@Before`,`@BeforeClass`注解等方法中，当最后一个`@After`,`@AfterClass`方法执行完毕后，`mock-up`类自动失效。
```
public class MyTestClass
{
   @BeforeClass
   public static void applySharedMockups()
   {
      new MockUp<LoginContext>() {
         // shared @Mock's here...
      };
   }
 
   // test methods that will share the mock-ups applied above...
}
```

#### Global mock-ups
```
@RunWith(Suite.class)
@Suite.SuiteClasses({MyFirstTest.class, MySecondTest.class})
public final class TestSuite
{
   @BeforeClass
   public static void applyGlobalMockUps()
   {
      new LoggingMocks();
 
      new MockUp<SomeClass>() {
         @Mock someMethod() {}
      };
   }
}
```

#### Applying AOP-style advice
Jmocke中还有一个特殊的Mock方法可以在`mock-up`类中出现：`$advice`。如果定义了此方法，它将处理目标类中的每一个方法。与普通的mock方法不同，这个方法有特殊的方法签名、返回值：`Object $advice(Invocation)`
```
public final class MethodTiming extends MockUp<Object>
{
   private final Map<Method, Long> methodTimes = new HashMap<>();
 
   public MethodTiming(Class<?> targetClass) { super(targetClass); }
   MethodTiming(String className) throws ClassNotFoundException { super(Class.forName(className)); }
 
   @Mock
   public Object $advice(Invocation invocation)
   {
      long timeBefore = System.nanoTime();
 
      try {
         return invocation.proceed();
      }
      finally {
         long timeAfter = System.nanoTime();
         long dt = timeAfter - timeBefore;
 
         Method executedMethod = invocation.getInvokedMember();
         Long dtUntilLastExecution = methodTimes.get(executedMethod);
         Long dtUntilNow = dtUntilLastExecution == null ? dt : dtUntilLastExecution + dt;
         methodTimes.put(executedMethod, dtUntilNow);
      }
   }
 
   @Override
   protected void onTearDown()
   {
      System.out.println("\nTotal timings for methods in " + mockedType + " (ms)");
 
      for (Entry<Method, Long> methodAndTime : methodTimes.entrySet()) {
         Method method = methodAndTime.getKey();
         long dtNanos = methodAndTime.getValue();
         long dtMillis = dtNanos / 1000000L;
         System.out.println("\t" + method + " = " + dtMillis);
      }
   }
}
```

### 基于行为的mock方式(mocking)
步骤为：
1. 录制：预期被调用的方法、返回值、调用测试。对应`Expectations`。
2. 回放：执行测试。对应真实测试代码。
3. 验证：检查被mock对象是否按照录制行为被调用。对应`Verifications`。
基本模板如下：
```
   @Test    
   public void testWithBothRecordAndVerify(mock parameters)    
   {    
      //测试前的准备  
     
      new Expectations() { // 期望块     
         {    
            // 一个或者多个mock对象(类型)的调用   
         }    
      };    
     
      // 单元测试业务逻辑
     
      new Verifications() {{ // 验证块    
    
      }};    
     
     // 其他额外的验证代码，JUnit/TestNG 断言等      
   }
```

#### Expectations
expectations是一个给定的测试要调用的方法的集合。在测试阶段，被测试对象调用mock对象的方法过程要与expectations匹配，包括：方法、参数、调用次数。
对于non-void方法，在方法调用后紧跟返回值，`result=xxx;`或者`returns(xxx);`
使用`result=xxx;`模式，还可以返回构造方法，异常对象。
连续调用同一个方法返回不同的值或异常，有以下几种方法：
1. 对`result`多次赋值
2. 调用`returns(Object...)`方法
3. 将`list`或者`array`赋值给`result`
```
        new Expectations(){
            {
                fun.publicMethod(anyString); //期望调用publicMethod
//                result = "mock1"; // 连续赋值
//                result = "mock2";
//                returns("mock1","mock2"); //使用returns()方法返回
//                result = Arrays.asList("mock1","mock2"); //返回list
                result = new String[]{"mock1","mock2"}; //返回array
                fun.withException(anyString);//期望调用withException
                result = new NullPointerException("mock exception");//抛出异常
            }
        };
```
#### Injectable
@Mocked 会作用于类的所有实例上。而@Injectable只作用于对象，不会mock构造方法、static方法、父类。
被测试类：
```
public final class ConcatenatingInputStream extends InputStream
{
   private final Queue<InputStream> sequentialInputs;
   private InputStream currentInput;
 
   public ConcatenatingInputStream(InputStream... sequentialInputs)
   {
      this.sequentialInputs = new LinkedList<InputStream>(Arrays.asList(sequentialInputs));
      currentInput = this.sequentialInputs.poll();
   }
 
   @Override
   public int read() throws IOException
   {
      if (currentInput == null) return -1;
 
      int nextByte = currentInput.read();
 
      if (nextByte >= 0) {
         return nextByte;
      }
 
      currentInput = sequentialInputs.poll();
      return read();
   }
}
```
测试类：
```
@Test
public void concatenateInputStreams(
   @Injectable final InputStream input1, @Injectable final InputStream input2)
   throws Exception
{
   new Expectations() {{
      input1.read(); returns(1, 2, -1);
      input2.read(); returns(3, -1);
   }};
 
   InputStream concatenatedInput = new ConcatenatingInputStream(input1, input2);
   byte[] buf = new byte[3];
   concatenatedInput.read(buf);
 
   assertArrayEquals(new byte[] {1, 2, 3}, buf);
}
```

#### Mocked
@Mocket注解可以修饰类字段或者方法参数上。
- 修饰类字段时
同一个测试用例都会受影响
- 修饰方法参数时
只影响所在方法范围

缺省的，@Mocked 标注的类型及其父类型(除了Object)的所有方法和构造方法都被mock。将类或者对象作为`Expectations`的参数可以mock部分方法，也就是说只有录制过的方法才被mock, 而其他的方法直接调用实际的实现。一个对象在`Expectations`中转化为mock对象后，之前的状态任然保留
- mock类
类、父类的所有方法及构造方法都可以被录制
```
      new Expectations(Collaborator.class) {{
         anyInstance.getValue(); result = 123;
      }};
```
- mock对象
```
      final Collaborator collaborator = new Collaborator(2);
 
      new Expectations(collaborator) {{
         collaborator.getValue(); result = 123;
         collaborator.simpleOperation(1, "", null); result = false;
 
         // Static methods can be dynamically mocked too.
         Collaborator.doSomething(anyBoolean, "test");
      }};
```
#### Capaturing
@Capaturing 使给定类的子孙类被mock
```
public interface Service { int doSomething(); }
final class ServiceImpl implements Service { public int doSomething() { return 1; } }
 
public final class TestedUnit
{
   private final Service service1 = new ServiceImpl();
   private final Service service2 = new Service() { public int doSomething() { return 2; } };
 
   public int businessOperation()
   {
      return service1.doSomething() + service2.doSomething();
   }
}
```
businessOperation() 是我们要测试的目标方法。但service1 和 service2 是 final 的属性，不能通过反射设置。我们需要在实例化被测实例前先 mock `service1`和匿名类`service2`, 但是匿名类，怎么mock呢！这时可以通过`@Capaturing`，来mock`Service`的实现类。
```
public final class UnitTest
{
   @Capturing Service anyService;
 
   @Test
   public void mockingImplementationClassesFromAGivenBaseType()
   {
      new Expectations() {{ anyService.doSomething(); returns(3, 4); }};
 
      int result = new TestedUnit().businessOperation();
 
      assertEquals(7, result);
   }
}
```
`@Capaturing`还可以通过可选属性`maxInstance`限制受影响的实例数量，这样就可以为父类的多个子类录制不同的行为。
```
@Test
public void testWithDifferentBehaviorForFirstNewInstanceAndRemainingNewInstances(
   @Capturing(maxInstances = 1) final Buffer firstNewBuffer,
   @Capturing final Buffer remainingNewBuffers)
{
   new Expectations() {{
      firstNewBuffer.position(); result = 10;
      remainingNewBuffers.position(); result = 20;
   }};
 
   // Code under test creates several buffers...
   ByteBuffer buffer1 = ByteBuffer.allocate(100);
   IntBuffer  buffer2 = IntBuffer.wrap(new int[] {1, 2, 3});
   CharBuffer buffer3 = CharBuffer.wrap("                ");
 
   // ... and eventually read their positions, getting 10 for
   // the first buffer created, and 20 for the remaining ones.
   assertEquals(10, buffer1.position());
   assertEquals(20, buffer2.position());
   assertEquals(20, buffer3.position());
}
```

#### Tested
通常，一个测试类总是操作一个被测试类的对象。通过`@Tested`可以自动实例化并为其注入mocked依赖。
`@Tested`类的属性通过`@Injectable`注入，非mock的数据（原始类型、数组）等也可以通过`Injectable`注入，但要显示赋值。
`@Tested`支持构造方法注入、属性注入。对应构造方法注入，需要有构造方法能与`@Injectable`变量匹配。
如果`@Tested`没有被实例化，则先通过匹配到的构造方法注入，之后通过属性注入；如果在定义的时候就被构造，则为null的属性会通过属性注入。
注入会先通过类型匹配，如果相同类型有多个，则通过名称匹配。
注意，被`final`修饰的属性只能通过构造方法注入。
实例化和依赖注入是在测试方法执行前完成的。
```
public class SomeTest
{
   @Tested CodeUnderTest tested;
   @Injectable Dependency dep1;
   @Injectable AnotherDependency dep2;
   @Injectable int someIntegralProperty = 123;
 
   @Test
   public void someTestMethod(@Injectable("true") boolean flag, @Injectable("Mary") String name)
   {
      // Record expectations on mocked types, if needed.
 
      tested.exerciseCodeUnderTest();
 
      // Verify expectations on mocked types, if required.
   }
}
```

#### 构造对象
在测试代码中创建对象实例，有以下两种方式
方式一：
```
@Test
public void newCollaboratorsWithDifferentBehaviors(@Mocked Collaborator anyCollaborator)
{
   // Record different behaviors for each set of instances:
   new Expectations() {{
      // One set, instances created with "a value":
      Collaborator col1 = new Collaborator("a value");
      col1.doSomething(anyInt); result = 123;
 
      // Another set, instances created with "another value":
      Collaborator col2 = new Collaborator("another value");
      col2.doSomething(anyInt); result = new InvalidStateException();
   }};
 
   // Code under test:
   new Collaborator("a value").doSomething(5); // will return 123
   ...
   new Collaborator("another value").doSomething(0); // will throw the exception
   ...
}
```
anyCollaborator 参数并没有用到，但这里 @Mocked 使得类型 Collaborator 成为Mocked 类型，方可对其方法和构造方法录制。 
方式二：
```
@Test
public void newCollaboratorsWithDifferentBehaviors(
   @Mocked final Collaborator col1, @Mocked final Collaborator col2)
{
   new Expectations() {{
      // Map separate sets of future instances to separate mock parameters:
      new Collaborator("a value"); result = col1;
      new Collaborator("another value"); result = col2;
 
      // Record different behaviors for each set of instances:
      col1.doSomething(anyInt); result = 123;
      col2.doSomething(anyInt); result = new InvalidStateException();
   }};
 
   // Code under test:
   new Collaborator("a value").doSomething(5); // will return 123
   ...
   new Collaborator("another value").doSomething(0); // will throw the exception
   ...
}
```
因为这里@Mocked 作用于两个同类型的变量，从而决定了之后对复发和的录制是关联到实例而非类。 

#### Flexible matching of argument values
在录制、验证阶段，不可能穷尽所有可能得参数值，这是需要使用弹性匹配。
弹性匹配通过两种方式表达，anyXxx 或 withXx() 方法，他们都是 Expectations 和 Verifacations 的基类 mockit.Invocations 的成员，所以对所有的Expectation 和 Verifaction 可用。
- any
提供了原始类型及其包装类型(`anyLong`,`anyInt`,`anyShort`...)，String类型(`anyString`)，通用类型(`any`)。
```
   new Expectations() {{
      // Will match "voidMethod(String, List)" invocations where the first argument is
      // any string and the second any list.
      abc.voidMethod(anyString, (List<?>) any);
   }};
```
- with
    1. 实例： `withSameInstance`, `withEqual/withNotEqual`, `withNull/withNotNull`
    1. 类型： `withAny`,  `withInstanceLike` , `withInstanceOf`
    1. String ： `withMatch`, `withSubString`,` withPrefix/withSuffix`
    1. 自定义匹配：`with` , `withArgThat`

#### Specifying invocation count constraints
在录制、验证阶段，可以通过三个属性指定方法的调用次数：`times`,`minTimes`,`maxTimes`。将其放在方法调用之后，对于每个方法每个属性只能使用一次。如果指定`times = 0` 或`maxTimes = 0`意味着只要方法被调用，测试就失败了。
```
@Test
public void someTestMethod(@Mocked final DependencyAbc abc)
{
   new Expectations() {{
      // By default, at least one invocation is expected, i.e. "minTimes = 1":
      new DependencyAbc();
 
      // At least two invocations are expected:
      abc.voidMethod(); minTimes = 2;
 
      // 1 to 5 invocations are expected:
      abc.stringReturningMethod(); minTimes = 1; maxTimes = 5;
   }};
 
   new UnitUnderTest().doSomething();
}
 
@Test
public void someOtherTestMethod(@Mocked final DependencyAbc abc)
{
   new UnitUnderTest().doSomething();
 
   new Verifications() {{
      // Verifies that zero or one invocations occurred, with the specified argument value:
      abc.anotherVoidMethod(3); maxTimes = 1;
 
      // Verifies the occurrence of at least one invocation with the specified arguments:
      DependencyAbc.someStaticMethod("test", false); // "minTimes = 1" is implied
   }};
}
```
####  验证
- Explicit verification
使用`Verifications`对方法、参数、调用次数做明确验证
```
   // Verifies that Dependency#doSomething(int, boolean, String) was called at least once,
   // with arguments that obey the specified constraints:
   new Verifications() {{ mock.doSomething(anyInt, true, withPrefix("abc")); }};
```
- Verification in order
使用`VerificationsInOrder`对方法的调用顺序做验证
```
@Test
public void verifyingExpectationsInOrder(@Mocked final DependencyAbc abc)
{
   // Somewhere inside the tested code:
   abc.aMethod();
   abc.doSomething("blah", 123);
   abc.anotherMethod(5);
   ...
 
   new VerificationsInOrder() {{
      // The order of these invocations must be the same as the order
      // of occurrence during replay of the matching invocations.
      abc.aMethod();
      abc.anotherMethod(anyInt);
   }};
}
```
- Partially ordered verification
使用`unverifiedInvocations`隔离方法之间顺序的相关性。
```
@Mocked DependencyAbc abc;
@Mocked AnotherDependency xyz;
 
@Test
public void verifyingTheOrderOfSomeExpectationsRelativeToAllOthers()
{
   new UnitUnderTest().doSomething();
 
   new VerificationsInOrder() {{
      abc.methodThatNeedsToExecuteFirst();
      unverifiedInvocations(); // Invocations not verified must come here...
      xyz.method1();
      abc.method2();
      unverifiedInvocations(); // ... and/or here.
      xyz.methodThatNeedsToExecuteLast();
   }};
}
```
上边的代码验证三件事：
    1. `abc.methodThatNeedsToExecuteFirst()`第一个调用
    2. `xyz.methodThatNeedsToExecuteLast()`最后一个调用
    3. `xyz.method1()`必须在`abc.method2()`之前调用

另外一种情况是我们关系某几个方法的调用顺序，其他的方法只验证调用，不验证顺序。这时可以通过两个验证块来实现。
```
@Test
public void verifyFirstAndLastCallsWithOthersInBetweenInAnyOrder()
{
   // Invocations that occur while exercising the code under test:
   mock.prepare();
   mock.setSomethingElse("anotherValue");
   mock.setSomething(123);
   mock.notifyBeforeSave();
   mock.save();
 
   new VerificationsInOrder() {{
      mock.prepare(); // first expected call
      unverifiedInvocations(); // others at this point
      mock.notifyBeforeSave(); // just before last
      mock.save(); times = 1; // last expected call
   }};
 
   // Unordered verification of the invocations previously left unverified.
   // Could be ordered, but then it would be simpler to just include these invocations
   // in the previous block, at the place where "unverifiedInvocations()" is called.
   new Verifications() {{
      mock.setSomething(123);
      mock.setSomethingElse(anyString);
   }};
}
```
- Full verification
通过`FullVerifications`来做完全验证，即所有方法都被调用，且调用次数匹配。如果在`expectation`中已经指定了方法调用最小次数限制，则`FullVerifications`中不需要再次指定该方法。
- Full verification in order
通过`FullVerificationsInOrder`做验证
- Restricting the set of mocked types to be fully verified
通过`FullVerifications(instance/class)`只对指定的 mock 对象或类型做完全验证。参数可以是class或对象。
```
@Test
public void verifyAllInvocationsToOnlyOneOfTwoMockedTypes(
   @Mocked final Dependency mock1, @Mocked AnotherDependency mock2)
{
   // Inside code under test:
   mock1.prepare();
   mock1.setSomething(123);
   mock2.doSomething();
   mock1.editABunchMoreStuff();
   mock1.save();
 
   new FullVerifications(mock1) {{
      mock1.prepare();
      mock1.setSomething(anyInt);
      mock1.editABunchMoreStuff();
      mock1.save(); times = 1;
   }};
}
```
- Verifying that no invocations occurred
使用空的`full verification `代码块可以作为没有任何调用的验证。如果`expectations`中已经指定了`times/minTimes`，则空的`full verification `代码块代表没有其他方法的调用
```
@Test
public void verifyNoInvocationsOnOneOfTwoMockedDependenciesBeyondThoseRecordedAsExpected(
   @Mocked final Dependency mock1, @Mocked final AnotherDependency mock2)
{
   new Expectations() {{
      // These two are recorded as expected:
      mock1.setSomething(anyInt);
      mock2.doSomething(); times = 1;
   }};
 
   // Inside code under test:
   mock1.prepare();
   mock1.setSomething(1);
   mock1.setSomething(2);
   mock1.save();
   mock2.doSomething();
 
   // Will verify that no invocations other than to "doSomething()" occurred on mock2:
   new FullVerifications(mock2) {};
}
```
- Verifying unspecified invocations that should not happen
在`FullVerifications`中指定方法的`minTimes = 0`可验证方法永远没有被调用

#### Delegates: specifying custom results
通过`Delegate`可以根据参数返回不同的结果。`Delegate`接口没有任何方法，被用来告知JMocket方法被调用后在返回结果阶段应该委托给指定的对象。所以实现中方法可以任意命名，但要方法保证不是`private`。方法的参数可以与record的方法参数一致，或者没有参数。
```
@Test
public void delegatingInvocationsToACustomDelegate(@Mocked final DependencyAbc anyAbc)
{
   new Expectations() {{
      anyAbc.intReturningMethod(anyInt, null);
      result = new Delegate() {
         int aDelegateMethod(int i, String s)
         {
            return i == 1 ? i : s.length();
         }
      };
   }};
 
   // Calls to "intReturningMethod(int, String)" will execute the delegate method above.
   new UnitUnderTest().doSomething();
}
```
构造方法同样通过`Delegate`方法来处理。
```
@Test
public void delegatingConstructorInvocations(@Mocked Collaborator anyCollaboratorInstance)
{
   new Expectations() {{
      new Collaborator(anyInt);
      result = new Delegate() {
         void delegate(int i) { if (i < 1) throw new IllegalArgumentException(); }
      };
   }};
 
   // The first instantiation using "Collaborator(int)" will execute the delegate above.
   new Collaborator(4);
```

#### Iterated expectations
`StrictExpectations`可以指定连续调用次数。
`expectations`同样可以指定，但只是作为最大、最小调用次数限制的倍数。
```
@Test
public void recordStrictInvocationsInIteratingBlock(@Mocked final Collaborator mock)
{
   new StrictExpectations(2) {{ //默认1
      mock.setSomething(anyInt);
      mock.save();
   }};
 
   // In the tested code:
   mock.setSomething(123);
   mock.save();
   mock.setSomething(45);
   mock.save();
}
```

#### Verifying iterations
验证中同样可以指定循环次数
```
@Test
public void verifyAllInvocationsInLoop(@Mocked final Dependency mock)
{
   int numberOfIterations = 3;
 
   // Code under test included here for easy reference:
   for (int i = 0; i < numberOfIterations; i++) {
      DataItem data = getData(i);
      mock.setData(data);
      mock.save();
   }
 
   new Verifications(numberOfIterations) {{
      mock.setData((DataItem) withNotNull());
      mock.save();
   }};
 
   new VerificationsInOrder(numberOfIterations) {{
      mock.setData((DataItem) withNotNull());
      mock.save();
   }};
}
```

#### Accessing private members
有时我们需要访问被测对象和mocked对象的私有成员，Deencapsulation 提供了静态方法来解决这个问题：
```
@Test
public void someTestMethod(@Mocked final DependencyAbc abc)
{
   final CodeUnderTest tested = new CodeUnderTest();
 
   // Defines some necessary state on the tested object:
   setField(tested, "someIntField", 123);
 
   new Expectations() {{
      // Expectations still recorded, even if the invocations are done through Reflection:
      newInstance("some.package.AnotherDependency", true, "test"); maxTimes = 1;
      invoke(abc, "intReturningMethod", 45, ""); result = 1;
      // other expectations recorded...
   }};
 
   tested.doSomething();
 
   String result = getField(tested, "result");
   assertEquals("expected result", result);
}
```

参考：
- [The JMockit Testing Toolkit Tutorial][jmockit]
- [JMockit 指南 翻译][yoogo]


[yoogo]: http://www.cnblogs.com/yoogo/p/JMockit.html
[jmockit]: http://jmockit.org/tutorial.html

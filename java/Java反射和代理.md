# JAVA 反射和代理

-----

> [Aliyz 反射代理代码传送门](https://github.com/Aliyzz/aliyz-case/blob/master/practice/src/main/java/com/aliyz/practice/dyproxy/DynamicProxyTest.java)

## JDK 反射机制

#### 要学反射，先从 类加载过程开始

> java 类的执行需要经历以下过程：
>
> ***编译***：`.java` 文件编译后生成 `.class` 字节码文件； <br>
> ***加载***：*类加载器* 负责根据一个类的 *全限定名* 来读取此类的二进制字节流到 JVM，并存储在方法区（元空间），然后将其转换为一个与目标类型对应的 `java.lang.Class` 对象实例； <br>
> ***连接***：细分三步
>> ***验证***：*格式*（class文件规范） *语义*（final类是否有子类） 操作； <br>
>> ***准备***：
>>> 1. 静态变量 *赋初值* 和 *分配内存空间*；
>>> 2. final 修饰的内存空间 *直接赋原值(不是用户指定的初值)* 。
>>
>> ***解析***：符号引用转化为直接引用，分配地址 ；
>
> ***初始化***：
>> 1. 有父类先初始化父类，然后初始化自己；
>> 2. 将 static 修饰代码执行一遍，如果是静态变量，则用用户指定值覆盖原有初值；
>> 3. 如果是代码块，则执行一遍操作。


#### 反射开始

> Java 的反射就是利用上面第二步加载到 jvm 中的 .class 文件来进行操作的；他是在运行状态中使用。
> 
> 对于任意一个类（Class），都能够 ***知道*** 这个类的所有 *属性* 和 *方法*；对于任意一个对象（Object），都能够 ***调用*** 它的任意 *属性* 和 *方法*；并且能改变它的属性。
> 
> 反射就是把 Java 类中的 *各种成分* 映射成一个个的 Java 对象，并且可以进行操作。
> 
> 围绕 `Class` 对象和 `java.lang.reflect` 类库

```java
# 想要使用反射，我先要得到 class 文件对象，其实也就是得到 Class 类的对象
# Class类主要API：
        成员变量  - Field
        成员方法  - Constructor
        构造方法  - Method
        
# 获取class文件对象的方式：
        1：Object类的getClass()方法
        2：数据类型的静态属性class
        3：Class类中的静态方法：public static Class forName(String className)
--------------------------------  
# 通过反射调用 成员变量
        1: 获取Class对象
        2：通过Class对象获取Constructor对象
        3：Object obj = Constructor.newInstance()创建对象
        4：Field field = Class.getField("指定变量名")获取单个成员变量对象
        5：field.set(obj,"") 为obj对象的field字段赋值
        
# 如果需要访问私有或者默认修饰的成员变量
        1:Class.getDeclaredField()获取该成员变量对象
        2:setAccessible() 暴力访问  
---------------------------------          
# 通过反射调用 成员方法
        1：获取Class对象
        2：通过Class对象获取Constructor对象
        3：Constructor.newInstance()创建对象
        4：通过Class对象获取Method对象  ------getMethod("方法名");
        5: Method对象调用invoke方法实现功能
        
# 如果调用的是私有方法那么需要暴力访问
        1: getDeclaredMethod()
        2: setAccessiable();          
```

#### 为什么使用反射

> 提高程序的灵活性 <br>
> 调用不可访问的方法 <br>
> 屏蔽掉实现的细节，让使用者更加方便好用

#### 谁在用

- JDBC：数据库连接驱动
- SpringMVC：各种 JavaBean

## 静态代理

> 静态代理是指在代码编写阶段就已经确定好了 `被代理类`、`代理类`、`接口` 这三个之间的关系。
> 
> 其中 ***代理类*** 实现 ***接口***，***被代理类*** 持有 ***代理类*** 的实例，仅此而已。
> 
> - 代理使的被代理类得到增强；
> - 一个代理类只能为同一个接口的类提供增强，如果有多个不同的接口，就需要定义很多的代理类。（*✨所以动态代理闪亮登场✨*）


## JDK 动态代理

> JDK动态代理是代理模式的一种实现方式，其只能代理接口。
> 
> 代理类继承了 Proxy 类并且实现了要代理的接口，***由于 Java 不支持多继承，所以 JDK 动态代理不能代理类***；<br>

#### 使用方式

> 三个重要的角色：`被代理类`、`代理类`、`接口`
>
> 1、新建一个 ***接口*** <br>
> 2、为接口创建一个实现类，即：***被代理类***<br>
> 3、创建 ***代理类***，并实现 ***`java.lang.reflect.InvocationHandler`*** 接口，即实现 `invoke(...)` 方法 <br>
> 4、使用 ***`Proxy.newProxyInstance(...)`*** 反射出 *代理对象*（这个是我们最终要生成的）。

#### 代理原理（待补充）

https://blog.csdn.net/OnlyloveCuracao/article/details/104889478

## CGLib 动态代理

> JDK 动态代理是基于接口的代理，对于没有实现接口的类，则使用 CGLib 实现动态代理；
> 
> 在 Hibernate 框架中 PO 的字节码生产工作就是靠CGLIB来完成的；
> 
> CGLib 采用非常底层的字节码技术，为一个类创建 ***子类***，并在子类中采用 **方法拦截** 的技术，拦截所有父类方法的调用，顺势织入 **横切逻辑**；

#### 使用方式

> 三个重要的角色：`被代理类`、`代理类`
> 
> 1、新建一个 ***被代理类***； <br>
> 2、创建 ***代理类***，并实现 ***`net.sf.cglib.proxy.MethodInterceptor`*** 接口，即实现 `intercept(...)` 方法； <br>
> 3、新建一个 ***`net.sf.cglib.proxy.Enhancer`*** 对象，然后依次 `setSuperclass(${被代理类})`、`setCallback(${代理类实例})`，最后调用 `create()` 方法创建 *代理对象*（这个是我们最终要生成的）。


## JDK 动态代理与 CGLib 动态代理比较

#### 原理比较

> JDK 动态代理利用 ***拦截器***（实现 InvocationHandler 接口，重写 invoke 方法）加上 ***反射机制*** (基础) 生成一个代理接口的匿名类，在调用具体方法前调用 InvokeHandler 来处理。
> 
> CGLib 原理是利用ASM框架，对指定的目标类生成一个子类，并覆盖其中方法实现增强，但因为采用的是继承，所以不能对 final 修饰的类进行代理。

#### 性能比较

> 从 JDK6 - JDK8 的发展来看，JDK 不断优化和改进，使得 JDK 代理模式性能越来越高（已经超过 CGLib）；相反 CGLib 性能没有跟上；
> 
> 但是早期的 CGLib 性能却高出 JDK 不少（这里就不细究了😊）。

#### 使用场景比较

> JDK 动态代理的类必须要实现一个接口，也就是说只能对该类所实现接口中定义的方法进行代理；
> 
> CGLib 动态代理不受代理类必须实现接口的限制，所以范围更广，但不能代理 final 修饰的类和方法，因为它是动态生成被代理类的子类；
> 
> Spring 的选择：
>> 当 bean 实现接口时，使用 JDK 代理模式；<br>
>> 当 bean 没有实现接口时，使用 CGLib 代理模式（通过配置可以强制使用 CGLib）。
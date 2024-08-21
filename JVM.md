# Java是编译型还是解释型？

1. 首先要知道什么是编译型语言，什么是解释型语言。

   **编译型**：通常我们任务把程序通过编译器把高级语言的源代码直接编译成可以被机器执行的机器码，直接交由机器执行的。如C语言

   **解释型**：解释型一般是在源代码执行的过程中，交由解释器当做中间人的角色进行解释执行，不需要编译成机器语言。如JavaScript。

   所以通常来讲编译型语言因为在执行的过程中是直接对接机器，少了解释的过程，一般执行速度会快不少。

2. Java是哪一类的呢？

   其实现在很多高级语言并不是单一的编译型或者解释型，比如Java。在Java中，首先会对源代码通过javac编译成字节码(class文件)，然后在执行时通过Java虚拟机进行解释执行。

   但是如果Java程序在通过解释器执行的过程中，当JVM发现某个方法或者代码块运行特别频繁的时候，会认为这是热点代码，会对代码进行**即时编译(JIT)**，会把这部分代码直接编译成机器码，这样就可以直接执行了。

   

   **扩展知识**：现在Java除了JIT以外，还支持AOT编译了，就是直接编译成字节码。那为什么有了JIT编译还需要AOT编译呢？

   1. JIT虽然有优点，但是缺点也很明显，他**增加了启动时间**（JIT是在程序运行时编译代码），并且**可能会影响应用性能**（JIT编译需要进行热点代码检测、代码编译等动作，这些都是需要占用运行期间的资源的）
   2. 为了适应现在云原生盛行的情况，AOT编译大大减少了程序的启动速度。AOT就是提前编译，不像JIT一样是在运行期间才生成机器码，而是在编译期间就将字节码转换成机器码，省去了运行时对JVM的依赖，是一种静态编译技术

# JIT编译时有哪些优化技术

1. 首先想要触发JIT，肯定需要先识别出哪些是**热点代码**

   - 基于采样的方式探测：周期性检测各个线程的栈顶，如果某个方法经常出现在栈顶，那么他就是热点代码。优点是比较简单，缺点是无法精准确认，容易受线程阻塞或其他原因的干扰。
   - 基于计数器的热点探测：采用这个方法会的虚拟机会为每个方法甚至是代码块建立计数器，统计方法执行次数，当次数超过阈值时就认为是热点方法，触发JIT编译。

2. **逃逸分析**

   - 全局逃逸：对象超出了方法或线程的范围，比如被存储在静态字段或作为方法的返回值

     ```java
     public class Test {
         // 这里的object就是全局逃逸的
         private static Object object;
         public void createObj(){
             object = new Object();
         }
     }
     ```

   - 参数逃逸：对象被作为参数传递或被参数引用

     ```java
     public void methodA(){
         Object obj = new Object();
     }
     
     public void methodB(Object obj){
         // 这里的对象obj就发生了参数逃逸，从methodA逃逸到了methodB
     }
     ```

   - 无逃逸：对象可以被标量替换，意味着他的内存分配可以从生成的代码中移除。

     ```java
     public static String mergeString(String s1, String s2) {
         StringBuffer sb = new StringBuffer();
         sb.append(s1);
         sb.append(s2);
         return sb.toString();
     }
     ```

     这里的sb对象就没有发生逃逸。

3. **锁消除**

   如果同步块所使用的锁对象通过分析被证实只能够被一个线程访问，那么JIT编译器在编译这个同步块的时候就会取消对这部分代码的同步。这个过程叫同步省略，也叫**锁消除**

4. **标量替换&栈上分配**

   - 标量替换：如果经过分析参数没有发生逃逸，就不会创建Point对象，而进行标量替换。

   ```java
   public void methodA() {
       Point point = new Point(1, 2);
       System.out.println("point.x = " + point.x + ";point.y = " + point.y);
   }
   
   class Point {
       int x;
       int y;
   }
   // 进行了标量替换
   public void methodA() {
       int x = 1;
     	int y = 2;
       System.out.println("point.x = " + x + ";point.y = " + y);
   }
   ```

   - 栈上分配：经过分析Point对象没有逃逸到方法之外，JIT编译就会优化Point到栈上分配，而不是分配到堆内存中，因为这个对象只有这一个方法在使用。

     ```java
     static class Point {
         int x;
         int y;
     }
     
     public static void main(String[] args) {
         for (int i = 1; i < 100000; i++) {
             Point point = new Point();
         }
     }
     ```

5. **方法内联**

   ```java
   public void test() {
       int res = add(1, 2);
   }
   
   public int add(int a, int b) {
       return a + b;
   }
   
   // 上面的方法被JIT优化后会被直接优化成下面的形式
   public void test() {
       int res = 1 + 2;
   }
   ```

# JVM的运行时内存区域是怎样的

JVM运行时内存区域主要由Java堆、虚拟机栈、本地方法栈、程序计数器以及运行时常量池组成。**其中堆、方法区以及运行时常量池是线程共享，而栈(本地方法栈和虚拟机栈)、程序计数器都是线程独享的**。

![image-20240821142633185](/Users/leiyu/Library/Application Support/typora-user-images/image-20240821142633185.png)

# Java的堆分代情况

![image-20240821143031483](/Users/leiyu/Library/Application Support/typora-user-images/image-20240821143031483.png)

Java堆由新生代和老年代组成，新生代存放新分配的对象，老年代存放长期存在的对象。新生成的年轻区(Eden区)和Survivor区(From Survivor、To Survivor)默认空间比例为8:2（其中From和To为1：1）,可以通过-XX:SurvivorRatio参数进行调整。

# 新生代为什么要分为Eden、From Survivor和To Survivor

新生代采用标记整理算法。

# 什么是STW？什么时候会STW

STW：stop the world，是在执行垃圾收集算法的时候，Java应用程序的其他所有线程都会被暂停，Java中的一种全局暂停现象，所有代码都会停止，native代码除外，但是不能与JVM进行交互。

不管哪种GC算法，STW都是无法避免的，只能降低STW的时长。

**扩展知识**：三色标记法的STW

![image-20240821144421760](/Users/leiyu/Library/Application Support/typora-user-images/image-20240821144421760.png)

![image-20240821144450285](/Users/leiyu/Library/Application Support/typora-user-images/image-20240821144450285.png)


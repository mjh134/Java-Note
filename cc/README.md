# CommonsCollections 反序列化利用链分析

## 各条利用链总结

| 链 | Trigger | Gadget | Sink | 核心思想 |
| --- | --- | --- | --- | --- |
| [CC1](cc1.md) | AnnotationInvocationHandler.invoke() | LazyMap | Runtime.exec | 动态代理触发 InvocationHandler |
| [CC2](cc2.md) | PriorityQueue.compare() | TransformingComparator | TemplatesImpl | 堆恢复时调用 Comparator |
| [CC3](cc3.md) | AnnotationInvocationHandler.invoke() | LazyMap | TemplatesImpl | 使用 TemplatesImpl 完成字节码加载 |
| [CC5](cc5.md) | BadAttributeValueExpException.toString() | TiedMapEntry | Runtime.exec | toString 回调触发 LazyMap |
| [CC6](cc6.md) | HashMap.hashCode() | TiedMapEntry | Runtime.exec | HashMap 重建哈希时调用 hashCode |
| [CC7](cc7.md) | Hashtable.equals() | LazyMap | Runtime.exec | Hashtable 检查哈希冲突时调用 equals |

- **CC5、CC6、CC7** 本质上都是围绕 **LazyMap + TiedMapEntry** 展开的，只是 Trigger 不同。
- **CC2、CC3** 的核心在于 **TemplatesImpl**，最终利用恶意字节码加载实现代码执行。
- **CC1** 则利用 AnnotationInvocationHandler 动态代理机制触发整个链条。

---

## 各条链的 Trigger 对比

| 链 | 自动调用的方法 |
| --- | --- |
| CC1 | invoke() |
| CC2 | compare() |
| CC3 | invoke() |
| CC5 | toString() |
| CC6 | hashCode() |
| CC7 | equals() |

> **真正值得学习的不是 Gadget，而是 Trigger。** Trigger 来自 JDK 正常恢复对象状态时自动执行的方法。不同利用链，只是在寻找不同的 Trigger。

---

## Sink 对比

| Sink | 对应链 |
| --- | --- |
| Runtime.exec | CC1、CC5、CC6、CC7 |
| TemplatesImpl | CC2、CC3 |

其中 `Runtime.exec` 利用反射逐步调用最终执行命令；`TemplatesImpl` 利用恶意字节码，在实例化 Translet 时完成代码执行。

---

## 链文件列表

- [CC1 - AnnotationInvocationHandler动态代理](cc1.md)
- [CC2 - PriorityQueue堆恢复触发](cc2.md)
- [CC3 - 动态代理 + TemplatesImpl字节码加载](cc3.md)
- [CC5 - BadAttributeValueExpException.toString()](cc5.md)
- [CC6 - HashMap重建哈希触发](cc6.md)
- [CC7 - Hashtable.equals()触发](cc7.md)

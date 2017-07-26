title: Java里Hashmap序列化的一个坑
date: 2017-05-15 12:18:30
categories: 踩坑总结
tags: Java
---

## 发现问题

在做业务需求的过程中，遇到一个非常奇怪的问题。在一个继承了`Serializable`接口的java bean里按照常规操作添加了一个hashmap和与之对应的getter、setter，就像这样：

```java
...
private HashMap<String, String> mChooseMap;

public HashMap<String, String> getChooseMap() {
        return mChooseMap;
    }

    public void setChooseMap(HashMap<String, String> chooseMap) {
        mChooseMap = chooseMap;
    }
...
```

然后我在某种情况下对含有这个hashmap的java bean进行了deep clone操作，就像这样：

```java
/**
     * 深度拷贝 要求data对象及其引用对象都实现了Serializable接口才可以用
     *
     * @param o 要深拷贝的对象
     */
    public static <T> T deepClone(T o) throws IOException, ClassNotFoundException {
        //将对象写到流里
        ByteArrayOutputStream bo = new ByteArrayOutputStream();
        ObjectOutputStream oo = new ObjectOutputStream(bo);
        oo.writeObject(o);
        //从流里读出来
        ByteArrayInputStream bi = new ByteArrayInputStream(bo.toByteArray());
        ObjectInputStream oi = new ObjectInputStream(bi);
        return (T) oi.readObject();
    }
```

很简单，对吧？在我的意料之中，这件事简直可以小的忽略不计，完全就是一个非常常规的操作。但是我被打脸了，啪啪的打，因为我发现了一个以前没遇到过的问题……为毛我的数据不见了？本来这个bean里的数据应该依照列表的形式在listview里加载出来，但是我发现我的listview一片空白。最关键的问题是，控制台并没有报错信息啊！

## 查错

面对这个问题，我先怀疑了一会儿人生。错误还是要查的，于是我只能先猜猜为什么会出现这个问题。我和之前的代码版本做了对比，发现我只多写了一个hashmap，就出现了问题，于是我怀疑是这个hashmap出现了一个我以前不太了解的坑，导致了现在的问题。

### Debug

首先我要庆幸的是，我的listview是应该拿到后端接口数据之后渲染的，既然控制台没有相关的日志输出，我就debug了一下，看看是不是后端的数据问题。结果我发现了一个让我惊讶的现象，在我处理请求返回的代码中，我正好做了deep clone操作，结果出现了如下错误：

![deep clone error](/images/deep_clone_error.png)

咦，控制台毛错没报，为毛一debug就出现了这个问题？在一开始的代码里，我在方法注释里已经写了，**深度拷贝 要求data对象及其引用对象都实现了Serializable接口才可以用**。这个错太打脸了，我竟然传了一个没有实现序列化接口的对象？再根据刚才的推断，难道说hashmap没有实现序列化接口？

### 追根溯源

非常震惊的我赶紧点开hashmap看了一眼：

```java
public class HashMap<K, V> extends AbstractMap<K, V> implements Cloneable, Serializable
```

我的脸有点疼。代码清清楚楚的写着，明明是序列化了……我不甘心，再次debug发现crash出现在这里：

```java
...
oo.writeObject(o);
...
```

有意思，这不是就是写对象吗，没想明白为什么挂在这里，于是我瞅了一眼hashmap的writeObject方法：

```java
private void writeObject(ObjectOutputStream stream) throws IOException {
        // Emulate loadFactor field for other implementations to read
        ObjectOutputStream.PutField fields = stream.putFields();
        fields.put("loadFactor", DEFAULT_LOAD_FACTOR);
        stream.writeFields();

        stream.writeInt(table.length); // Capacity
        stream.writeInt(size);
        for (Entry<K, V> e : entrySet()) {
            stream.writeObject(e.getKey());
            stream.writeObject(e.getValue());
        }
    }
```

是`private`的？为什么不是public？于是我从crash出现的地方一层层点进去看代码，边看边想是哪里出了问题。看着看着我发现这么一段代码，在`ObjectOutputStream`类的`writeObjectInternal`方法中：

```java
...
if (clDesc.hasMethodWriteReplace()){
                    Method methodWriteReplace = clDesc.getMethodWriteReplace();
                    Object replObj;
                    try {
                        replObj = methodWriteReplace.invoke(object, (Object[]) null);
                    } catch (IllegalAccessException iae) {
                        replObj = object;
                    } catch (InvocationTargetException ite) {
                        // WARNING - Not sure this is the right thing to do
                        // if we can't run the method
                        Throwable target = ite.getTargetException();
                        if (target instanceof ObjectStreamException) {
                            throw (ObjectStreamException) target;
                        } else if (target instanceof Error) {
                            throw (Error) target;
                        } else {
                            throw (RuntimeException) target;
                        }
                    }
          			...
                    }
                }
```

这下我看懂了，这不就是说，如果要这对象有自己的writeObject方法，就在这里会用反射的方式执行对象自己的writeObject方法么，这么一来hashmap里的`private void writeObject`就可以理解了。那我这个问题也就简单了，我在这里加个断点，我看看hashmap到底writeObject除了什么问题不就可以了么。

于是再次debug。不看不知道，一看吓一跳！我发现原来问题出现在这里：

![writeObject_error](/images/writeObject_error.png)

What?!原来错误是从这里报出来的。我仔细看了一下报错后面的信息，居然是一个其他的类，确实不是一个可以序列化的类。

此时此刻我的脸真的很疼。

代码不会骗人，我确实传进来了一个没有序列化的类。于是我赶紧点进去报错提示的类里查看，我发现了我是在这个类里往bean中set我的map。我保证这是我第一次这么set：

```java
contactItem.setChooseMap(new LinkedHashMap() {
                    {
                        put("男", "1");
                        put("女", "2");
                    }
                });
```

明明是new了一个hashmap，但是writeObject的时候传进去的却是当前的这个类。看来确实是我姿势不对。

在我疑惑的时候，我搜到这么一篇文章：

[hashmap-not-serializable](http://stackoverflow.com/questions/32790025/hashmap-not-serializable)

和我差不多的问题嘛，我瞅了一眼回答。

> The exception message tells you exactly what the problem is: you are trying to serialize an instance of class SimpleSerializationTest, and that class is not serializable.

> Why? Well, you have created an anonymous inner class of SimpleSerializationTest, one that extends HashMap, and you are trying to serialize an instance of that class. Inner classes always have references to the relevant instance of their outer class, and by default, serialization will try to traverse those.

嗯？这话大概意思就是，在一个类里创建一个匿名内部类，相当于扩展了hashmap和要序列化的类的实例。内部类会持有外部类的引用，默认情况下会遍历这些进行序列化。

卧槽！原来是这个原因！怪不得我用来new hashmap的这个类被writeObject然后报错了！因为它根本就不能被序列化嘛。

## 解决问题

最后我改成了这样：

```java
LinkedHashMap<String, String> genderMap = new LinkedHashMap<>();
                genderMap.put("男", "1");
                genderMap.put("女", "2");
                contactItem.setChooseMap(genderMap);
```

问题完美解决。

以后再也不会犯这个错误了，脸疼，这回深刻的记住了。
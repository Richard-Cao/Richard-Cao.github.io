title: 记一次Android本地拒绝服务漏洞的追根溯源
date: 2018-08-03 15:28:30
categories: 踩坑总结
tags: Android
---

## 背景

由于一些xxxxoooo的原因，我需要修复一个**跨进程**Service可能会被外部传入intent数据导致拒绝服务的问题。当被告知这个问题的时候，我是一脸懵逼的。

复现这个问题的测试代码如下：

```java
Intent i = new Intent();
i.setClassName("我要调起的应用的包名", "我要调起的对外导出的Service");
// TestSerializeObj是我构造的一个只实现了序列化接口的空类
i.putExtra("瞎写的测试数据", new TestSerializeObj());
startService(i);
```

我看到的报错信息大概是这样的：java.lang.RuntimeException: Parcelable encountered ClassNotFoundException reading a Serializable object，堆栈崩在了Service的onStartCommand方法中。于是我开始对着onStartCommand里的一小段代码冥思苦想：

```java
@Override
public int onStartCommand(Intent intent, int flags, int startId) {
    // 省略无关代码
    ...
    if (intent == null) {
        intent = new Intent();
        // 省略无关代码
        ...
        intent.putExtra("aa", "bb");
    } else if (intent.getAction() == null) {
        // 省略无关代码
        ...
        intent.putExtra("aa", "cc");
    }
    // 省略无关代码
    ...
}
```

和Intent相关的就这些了。根据我看到的堆栈崩溃行数，就是intent.putExtra方法出现了ClassNotFoundException。实在搞不清原因，只能追踪一下看看了。

## 追根溯源

首先是putExtra方法：

```java
public @NonNull Intent putExtra(String name, String value) {
    if (mExtras == null) {
        mExtras = new Bundle();
    }
    mExtras.putString(name, value);
    return this;
}
```

继续看putString方法：

```java
public void putString(@Nullable String key, @Nullable String value) {
    unparcel();
    mMap.put(key, value);
}
```

map的put应该没什么问题，那有可能有问题的就是unparcel方法了。继续看下去：

```java
/* package */ void unparcel() {
    synchronized (this) {
        final Parcel parcelledData = mParcelledData;
        // 省略无关代码
        ...
        try {
            parcelledData.readArrayMapInternal(map, N, mClassLoader);
        } catch (BadParcelableException e) {
            if (sShouldDefuse) {
                Log.w(TAG, "Failed to parse Bundle, but defusing quietly", e);
                map.erase();
            } else {
                throw e;
            }
        } finally {
            mMap = map;
            parcelledData.recycle();
            mParcelledData = null;
        }
        // 省略无关代码
        ...
    }
}
```

如果是这个方法出问题的话，问题应该出在readArrayMapInternal方法上。继续查看这个方法（这个方法在Parcel类中）：

```java
/* package */ void readArrayMapInternal(ArrayMap outVal, int N,
    ClassLoader loader) {
    // 省略无关代码
    ...
    while (N > 0) {
        // 省略无关代码
        ...
        String key = readString();
        Object value = readValue(loader);
        // 省略无关代码
        ...
        N--;
    }
    outVal.validate();
}
```

可以看到，这里做的操作实际上是循环读值。因为是ClassNotFoundException，所以问题应该出在和ClassLoader有关的方法上，这里唯一使用了ClassLoader的就是readValue了。那么看一下readValue方法做了什么：

```java
public final Object readValue(ClassLoader loader) {
    int type = readInt();

    switch (type) {
        // 省略无关代码
        ...
        case VAL_SERIALIZABLE:
            return readSerializable(loader);
        // 省略无关代码
        ...
    }
}
```

由于我测试代码是构造了一个空实现的Serializable对象塞进来，所以相关的方法应该是readSerializable，继续看readSerializable做了什么事：

```java
private final Serializable readSerializable(final ClassLoader loader) {
    String name = readString();
    // 省略无关代码
    ...
    byte[] serializedData = createByteArray();
    ByteArrayInputStream bais = new ByteArrayInputStream(serializedData);
    try {
        ObjectInputStream ois = new ObjectInputStream(bais) {
            @Override
            protected Class<?> resolveClass(ObjectStreamClass osClass)
                    throws IOException, ClassNotFoundException {
                // try the custom classloader if provided
                if (loader != null) {
                    Class<?> c = Class.forName(osClass.getName(), false, loader);
                    if (c != null) {
                        return c;
                    }
                }
                return super.resolveClass(osClass);
            }
        };
        return (Serializable) ois.readObject();
    } catch (IOException ioe) {
        throw new RuntimeException("Parcelable encountered " +
                "IOException reading a Serializable object (name = " + name +
                ")", ioe);
    } catch (ClassNotFoundException cnfe) {
        throw new RuntimeException("Parcelable encountered " +
                "ClassNotFoundException reading a Serializable object (name = "
                + name + ")", cnfe);
    }
}
```

重头戏来了！崩溃日志在这里出现了！这里通过传进来的ClassLoader寻找是否有我传进来的空数据类，显然不可能有，于是ClassNotFoundException就非常稳定的出现了。

至此，问题的根源已经找到了。

## 总结

修复方式也简单，在**对外暴露（跨进程）**的Activity、Service或者Broadcast等需要用到intent的地方，不管是putXXX还是getXXX（getXXX也需要执行unparcel方法）操作，都加上try-catch抓住这个错误就可以了。说实话，这个问题我第一次遇到，这是一个安全性问题，算是让我记忆深刻了，以后不能再踩这个坑了。

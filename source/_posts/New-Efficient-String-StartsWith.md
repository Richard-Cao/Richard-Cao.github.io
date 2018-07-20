title: 一种高效通用的字符串前置匹配方法
date: 2018-07-20 10:34:30
categories: 踩坑总结
tags: Java
---

## 现状

一般情况下，我们使用字符串的前置匹配方法，如果要做的通用一些，忽略大小写，可能会这么写：

```java
public boolean startsWithIgnoreCase(String str, String prefix) {
    if (!TextUtils.isEmpty(str) && !TextUtils.isEmpty(prefix)) {
        String tempS = str.toLowerCase();
        String tempP = prefix.toLowerCase();
        return tempS.startsWith(tempP);
    }
    return false;
}
```

从功能上说，这么写当然没问题。但是，我们忽略了一个问题：**toLowerCase(toUpperCase)方法是耗时的**。为什么耗时？以toLowerCase为例：

```java
public static String toLowerCase(Locale locale, String s) {
    // Punt hard cases to ICU4C.
    // Note that Greek isn't a particularly hard case for toLowerCase, only toUpperCase.
    String languageCode = locale.getLanguage();
    if (languageCode.equals("tr") || languageCode.equals("az") || languageCode.equals("lt")) {
        return ICU.toLowerCase(s, locale);
    }

    String newString = null;
    for (int i = 0, end = s.length(); i < end; ++i) {
        char ch = s.charAt(i);
        char newCh;
        if (ch == LATIN_CAPITAL_I_WITH_DOT || Character.isHighSurrogate(ch)) {
            // Punt these hard cases.
            return ICU.toLowerCase(s, locale);
        } else if (ch == GREEK_CAPITAL_SIGMA && isFinalSigma(s, i)) {
            newCh = GREEK_SMALL_FINAL_SIGMA;
        } else {
            newCh = Character.toLowerCase(ch);
        }
        if (ch != newCh) {
            if (newString == null) {
                newString = StringFactory.newStringFromString(s);
            }
            newString.setCharAt(i, newCh);
        }
    }
    return newString != null ? newString : s;
}
```

**会遍历字符串并且重新赋值给newString**，最终调用的是Character.toLowerCase方法：

```java
public static int toLowerCase(int codePoint) {
    // Optimized case for ASCII
    if ('A' <= codePoint && codePoint <= 'Z') {
        return (char) (codePoint + ('a' - 'A'));
    }
    if (codePoint < 192) {
        return codePoint;
    }
    return toLowerCaseImpl(codePoint);
}
```

最后这里的toLowerCaseImpl就是真正实现的方法，这是一个Native方法，也是由C/C++完成的。

toLowerCaseImpl效率高，然而在调用toLowerCaseImpl之前的操作，会耗时。那我们最开始的startsWithIgnoreCase方法的性能瓶颈就在这里了。如果想要减少耗时，应该怎么办呢？

## 分析

耗时关键点：toLowerCase。这个方法，当然是没什么替代的空间， 只能另辟蹊径。关于String的忽略大小写比对，除了toLowerCase和toUpperCase之后再比对，我们还会想到一个方法：equalsIgnoreCase。

这个方法的效率会比先toLowerCase之后再比对的效率高。为什么呢？

```java
public boolean equalsIgnoreCase(String string) {
    if (string == this) {
        return true;
    }
    if (string == null || count != string.count) {
        return false;
    }
    for (int i = 0; i < count; ++i) {
        char c1 = charAt(i);
        char c2 = string.charAt(i);
        if (c1 != c2 && foldCase(c1) != foldCase(c2)) {
            return false;
        }
    }
    return true;
}
```

**并不会对字符串进行全拷贝**，foldCase就直接调用到上文中Character.toLowerCase方法，最后调用到toLowerCaseImpl方法。

## 实现

说了这么多，肯定有小伙伴会问：我们不是在聊字符串前置匹配吗？equals速度快了和前置匹配有什么关系呢？

让我们想想前置匹配的逻辑：处理好忽略大小写后的字符串，进行startsWith操作，是不是可以等同于，**字符串str截取从下标为0到prefix.length的位置，这部分前置字符串再和prefix进行equalsIgnoreCase**？如果str的长度小于等于prefix的长度，其实startsWith和equals是一样的。

逻辑已经梳理清楚了，那么就简单了，动手写一下新的忽略字符串大小写的前置匹配通用方法：

```java
public static boolean startsWithIgnoreCase(String str, String prefix) {
    if (!TextUtils.isEmpty(str) && !TextUtils.isEmpty(prefix)) {
        if (str.length() > prefix.length()) {
            str = str.substring(0, prefix.length());
        }
        return str.equalsIgnoreCase(prefix);
    }
    return false;
}
```

方法写好了，让我们写个Demo验证一下是否真的有优化效果吧。

## 验证

上Demo代码：

```java
long start = System.currentTimeMillis();
for (int i = 0; i < 10000; i++) {
    startsWithIgnoreCase("sdasdasdasdasdasdhgjhgkturyetrtuuuii",
                    "sdasdasdasdasdasdhgjhgkturyetrtuuuii");
}
long diff = System.currentTimeMillis() - start;
Log.d("test", "test startsWithIgnoreCase: " + diff + "ms");

long startNew = System.currentTimeMillis();
for (int i = 0; i < 10000; i++) {
    startsWithIgnoreCaseNew("sdasdasdasdasdasdhgjhgkturyetrtuuuii",
                    "sdasdasdasdasdasdhgjhgkturyetrtuuuii");
}
long diffNew = System.currentTimeMillis() - startNew;
Log.d("test", "test startsWithIgnoreCaseNew: " + diffNew + "ms");
```

这里的startsWithIgnoreCaseNew方法就是新实现的startsWithIgnoreCase方法。当str和prefix完全一样的时候，验证结果如下：

![startsWithIgnoreCaseEqual](/images/startsWithIgnoreCaseEqual.png)

如果在str和prefix中随机加一些大写字符，验证结果如下：

![startsWithIgnoreCaseNotEqual](/images/startsWithIgnoreCaseNotEqual.png)

**高下立判！**

**高下立判！**

**高下立判！**
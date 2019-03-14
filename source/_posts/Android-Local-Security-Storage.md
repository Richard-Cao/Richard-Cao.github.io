title: Android纯本地安全存储方案
date: 2019-03-14 14:28:30
categories: 踩坑总结
tags: Android
---

## 背景

在业务中，我们可能需要面对在Android本地存储用户token、email等敏感数据。本文将讲述一种安全系数较高的Android本地存储方案。

## 思路

整个方案的核心思路围绕KeyStore展开，如果不太了解KeyStore的小伙伴，请先阅读[Android密钥库系统](https://developer.android.com/training/articles/keystore?hl=zh-CN)。

由于KeyStore在Android 6.0前后差异较大，并且Android 4.3以下系统并不支持KeyStore，方案需要根据不同的Android版本做不同的处理，以及需要提供降级方案。**加解密算法采用AES/CBC/PKCS7Padding。**

### Android 6.0或更高版本系统

这种情况下方案最简单，随机生成128位AES key和iv，key存储在KeyStore中，设置alias，iv存储在SharedPreferences中。需要加解密的时候通过alias从KeyStore中取key，从SharedPreferences中取iv。

首先初始化KeyStore：

```java
private static final String ANDROID_KEY_STORE = "AndroidKeyStore";

// 初始化
if (Build.VERSION.SDK_INT >= Build.VERSION_CODES.JELLY_BEAN_MR2) {
    try {
        mKs = KeyStore.getInstance(ANDROID_KEY_STORE);
        mKs.load(null);
    } catch (Exception e) {
        // 降级方案
        return;
    }
}
```

然后就是生成key了，这里贴出核心代码：

```java
@RequiresApi(api = Build.VERSION_CODES.JELLY_BEAN_MR2)
public void createNewKeys(Context context, String alias)
        throws KeyStoreException, NoSuchProviderException, NoSuchAlgorithmException,
        InvalidAlgorithmParameterException, IllegalBlockSizeException, InvalidKeyException,
        BadPaddingException, NoSuchPaddingException {
    try {
        if (TextUtils.isEmpty(alias) || mKs.containsAlias(alias)) {
            return;
        }
    } catch (NullPointerException e) {
        return;
    }

    // 省略无关紧要的代码...

    AlgorithmParameterSpec spec;
    if (Build.VERSION.SDK_INT < Build.VERSION_CODES.M) {
        // 这里是第二种情况，下面会详细讲
    } else {
        // generate aes key
        KeyGenerator kGenerator = KeyGenerator.getInstance(
                KeyProperties.KEY_ALGORITHM_AES, ANDROID_KEY_STORE);
        spec = new KeyGenParameterSpec.Builder(alias,
                KeyProperties.PURPOSE_ENCRYPT | KeyProperties.PURPOSE_DECRYPT)
                .setCertificateSubject(new X500Principal("CN=" + alias))
                .setDigests(KeyProperties.DIGEST_SHA256, KeyProperties.DIGEST_SHA512)
                .setBlockModes(KeyProperties.BLOCK_MODE_CBC)
                .setKeySize(128)
                .setEncryptionPaddings(KeyProperties.ENCRYPTION_PADDING_PKCS7)
                .setRandomizedEncryptionRequired(false)
                .build();
        kGenerator.init(spec);
        kGenerator.generateKey();
        // generate aes iv and save
        // 省略无关紧要的代码...
    }
}
```

那加解密的时候怎么取得key呢？很简单：

```java
Key key = mKs.getKey(alias, null);
```

取出来key作为Cipher的参数在init的时候传入就可以了，剩下的就是常规的加解密，这里就不贴代码了。

### Android 4.3 - 5.1

由于KeyStore在Android 4.3 - 5.1版本不支持AES算法，所以需要随机生成RSA的key pair，**算法采用RSA/ECB/PKCS1Padding**，设置alias并存储在KeyStore中。然后随机生成128位AES key和iv，使用前面生成的RSA公钥加密key，key和iv一起存储在SharedPreferences中。

需要加解密的时候，从SharedPreferences中读取加密后的key，从KeyStore中取出RSA私钥，使用RSA私钥解密得到真正的AES key，再进行加解密。

这里贴出第二种情况下的代码：

```java
private static final String TYPE_RSA = "RSA";

// 下面这段代码就是在if (Build.VERSION.SDK_INT < Build.VERSION_CODES.M)条件中的
// generate rsa key pair
KeyPairGenerator kpGenerator = KeyPairGenerator.getInstance(TYPE_RSA,
        ANDROID_KEY_STORE);
spec = new KeyPairGeneratorSpec.Builder(context)
        .setAlias(alias)
        .setSubject(new X500Principal("CN=" + alias + " ,O=Android Authority"))
        .setSerialNumber(BigInteger.valueOf(1337))
        .build();
kpGenerator.initialize(spec);
try {
    kpGenerator.generateKeyPair();
    // generate aes key and iv
    generateAESKeyAndIV(alias);
} catch (IllegalStateException e) {
    // keystore locked
    // I guess no lock screen pin password
    // https://issuetracker.google.com/issues/37051017
    // 降级方案
} catch (IllegalArgumentException e) {
    // When keystore generates the key pair, it generates a self signed cert.
    // The ASN1 parser used internally by Android Keystore doesn't correctly take in
    // the locale and it causes the failures for device locale with language from
    // right to left
    // https://issuetracker.google.com/issues/37095309
    // 降级方案
} catch (RuntimeException e) {
    // Sometimes throw RuntimeException (Samsung Android 4)
    // error:0D07207B:asn1 encoding routines:ASN1_get_object:header too long
    // I guess maybe related to keystore being locked
    // Demote temporarily
    // 降级方案
}
```

代码一贴，一切就明朗起来了。这里要注意处理代码中catch的几种异常，会在某些三星手机或者其他手机上出现。

那这里怎么取key呢？

```java
private static final String AES_CBC_PKCS7_PADDING = "AES/CBC/PKCS7Padding";

// 从SharedPreferences中取出RSA加密过的aes key
String encryptAESKey = xxx
byte[] aesKey = decryptRSA(alias, encryptAESKey); // 进行RSA解密
if (aesKey == null) {
    return null;
}
return new SecretKeySpec(aesKey, AES_CBC_PKCS7_PADDING);
```
取出来key作为Cipher的参数在init的时候传入就可以了，剩下的就是常规的加解密，这里就不贴代码了。这种情况下，加解密会比第一种情况下耗时，因为需要经历一次RSA的解密操作。

### Android 4.3以下版本以及降级方案

KeyStore不支持Android 4.3以下的系统。这里提供一种简单思路：可以写一个so库，通过种子字符串生成固定的128位AES key和iv，将iv存储在SharedPreferences中。在so库中内置应用签名，在JNI_OnLoad函数中进行签名校验，检验不通过直接Crash。**生成的key不保存在本地**，需要加解密的时候调用so方法实时获取key。

注：种子字符串是一串随机的字符串，内置在so库中，用于通过计算生成key。如果签名验证失败，就无法生成key。它不是绝对安全的，只是尽可能保证安全。

## 补充

建议提供root检测工具，如果检测到设备已经root，直接提示用户存在安全风险。

## 总结

围绕KeyStore的方案大致思路讲完了，其中一些坑也是线上踩了总结出来的，核心代码其实很少，思路最关键，欢迎交流。

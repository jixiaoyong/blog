---
title: java实现AES加密解密应用
date: 2018-01-22 23:51:19
tags: 加密
---

# 简介

最近手机中涉及到用户账户密码保存的问题，选用AES加密算法进行加密后，再通过SharedPreferences保存在手机端。

本文主要介绍AES的加密、解密用法。

# 代码

初始化秘钥

```java
    private static final String AES = "AES";
    private static final String PASSWPRD = "123456";

    public static SecretKeySpec initKey(){
        SecretKeySpec key = null;

        try {

            KeyGenerator kg = KeyGenerator.getInstance(AES);
            kg.init(128,new SecureRandom(PASSWPRD.getBytes()));//通过这种算法，每次生成的key都是一样的
          //也可以kg.init(128),这样每次生成的key都不一样
            SecretKey securityKey = kg.generateKey();
            byte[] encodedKey = securityKey.getEncoded();
            key = new SecretKeySpec(encodedKey, AES);//AES也可以替换为"AES/CBC/PKCS5PADDING"

        } catch (NoSuchAlgorithmException e) {
            e.printStackTrace();
        }

        return key;
    }
```

加密

```java
//核心代码
//source是要加密的内容
Cipher cipher = Cipher.getInstance(AES);//创建密码器
byte[] byteContent = source.getBytes("utf-8");
cipher.init(Cipher.ENCRYPT_MODE, key);//创建密码器
byte[] result = cipher.doFinal(byteContent);//加密
```

解密

```java
//核心代码
Cipher cipher = Cipher.getInstance(AES);
cipher.init(Cipher.DECRYPT_MODE, key);
byte[] result = cipher.doFinal(source);
```

加密和解密的结果都是二进制的，无法直接转化为字符串，所以还需要将二进制与十六进制互转

```java
    public static String parseByte2HexStr(byte buf[]) {
        StringBuffer stringBuffer = new StringBuffer();
        for (int i = 0; i < buf.length; i++) {
            String hex = Integer.toHexString(buf[i] & 0xff);
            if (hex.length() == 1) {
                hex = '0' + hex;
            }
            stringBuffer.append(hex.toUpperCase());
        }
        return stringBuffer.toString();
    }

    public static byte[] parseHexStr2Byte(String hexStr){
        if (hexStr.length() < 1) {
            return null;
        }

        byte[] result = new byte[hexStr.length() / 2];
        for (int i = 0; i < hexStr.length() / 2; i++) {
            int high = Integer.parseInt(hexStr.substring(i * 2, i * 2 + 1), 16);
            int low = Integer.parseInt(hexStr.substring(i * 2 + 1, i * 2 + 2), 16);
            result[i] = (byte) (high * 16 + low);
        }

        return result;
    }
```

这样就可以在初始化一个key后，对文本进行加密和解密

```java
//初始化key
SecretKeySpec key = initKey();
//加密文本并转化为16进制，方便保存
String eStr = parseByte2HexStr(encrypt(resource,key));
//将加密16进制文本转为二进制，进行解密
String dStr = decrypt(parseHexStr2Byte(estr));
```



# 参考文献

[JAVA实现AES加密 - CSDN博客](http://blog.csdn.net/hbcui1984/article/details/5201247)

[源码github链接](https://github.com/jixiaoyong/AndroidNote/tree/master/code/2018-1-22/encryption)
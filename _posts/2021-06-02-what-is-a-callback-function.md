---
title: Why are public static final array a security hole?
author: Mista
date: 2021-06-01 00:32:00 +0800
categories: [Question]
tags: [static,final]
---

Effective Java说：

> // Potential security hole!
>
> static public final Thing[] VALUES = {...};

## public static final array

声明`static final`公共字段标志着类常量，非常适合原始数据类型`（int、double等）`和不可变类。

对于数组，问题在于即使数组引用是常量，数组元素仍然是可变的；由于它是一个字段，不受更改保护、不可控，所以不常用。

为了解决这个问题，可以将数组字段限制为私有/包私有，这样可以缩小可疑代码的范围。通常，更好的做法是取消数组使用`List`或者集合类型。通过集合，只有通过方法才能更新字段，因此变化可控。

考虑下面的例子：

```Java
public class SafeSites {
    // a trusted class with permission to create network connections
    public static final String[] ALLOWED_URLS = new String[] {
        "http://amazon.com", "http://cnn.com"};

    // this method allows untrusted code to connect to allowed sites (only)
    public static void doRequest(String url) {
        for (String allowed : ALLOWED_URLS) {
            if (url.equals(allowed)) {
                 // send a request ...
            }
        }
    }
}

public class Untrusted {
     // An untrusted class that is executed in a security sandbox.

     public void naughtyBoy() {
         SafeSites.ALLOWED_URLS[0] = "http://myporn.com";
         SafeSites.doRequest("http://myporn.com");
     }
}
```



>Note that a nonzero-length array is always mutable, so **it is wrong for a class to have a public static final array field, or an accessor that returns such a field.** If a class has such a field or accessor, clients will be able to modify the contents of the array.
>
>-- *Effective Java*, 2nd Ed. (page 70)

## Reference

[Why are public static final array a security hole?](https://stackoverflow.com/questions/2842169/why-are-public-static-final-array-a-security-hole/2842175)


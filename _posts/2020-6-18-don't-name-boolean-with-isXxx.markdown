---
layout: post
title:  "POJO类中布尔类型为啥不让用isXxx命名"
date:   2020-6-18 08:00:00 +0800
categories: Java
---
> 源码面前，了无秘密

《阿里开发规范泰山版》(2020.04.22)-->编程规约-->(一) 命名风格-->第8条规定：

>【强制】POJO 类中的任何布尔类型的变量，都不要加 is 前缀，否则部分框架解析会引起序列化错误。

对于这样一条【强制】级别的规定，虽然规范中做了简单的说明，但依然显得很不起眼，以至于我虽然规范背的很熟，依然踩到了这个坑。

## 0 起因

最近写了一个钉钉告警工具类，对于这种需求明确，开发文档清晰的任务，写代码信手拈来，很快就写完了。

但是自测的时候却发现@所有人这个接口始终过不了单元测试，经过一番追踪，终于找到原因，于是有了本文。

## 1 问题重现

本身发钉钉消息就是一个post请求的事，而且钉钉有阿里背书，调用不通那自然是我这个开发者的问题喽。

下面是钉钉的接口说明，参数`isAtAll`表示是否@所有人

```json
{
    "msgtype": "text", 
    "text": {
        "content": "我就是我, 是不一样的烟火@156xxxx8827"
    }, 
    "at": {
        "atMobiles": [
            "156xxxx8827", 
            "189xxxx8325"
        ], 
        "isAtAll": false
    }
}
```

根据这个json串，我编写了如下的POJO类与之对应，并使用fastjson进行json序列化

```java
public class Message {
    /**
     * 消息类型
     **/
    private String msgtype = "text";
    /**
     * 消息内容对象
     **/
    private Text text;
    /**
     * 被@对象
     **/
    private At at;

    public static class At{
        /**
         * 被@人电话
         **/
        private List<String> atMobiles;

        /**
         * 是否@所有人
         **/
        private boolean isAtAll;
        // 省略getter、setter
        }
    // 省略无关代码...
}
```

问题就出在`private boolean isAtAll;`这个字段上，有没有发现阿里的这个参数违背了开篇提到的开发规范？使用fastjson序列化之后，该属性实际转化为了:

```json
"atAll": true
```

这个坑爹货，把前面的is给吃了！导致无法@所有人

## 2 追根溯源

为什么序列化之后把is吃了呢？带着这个问题我追踪了一下fastjson的源码，发现在序列化的时候，其使用的是属性的getter方法名，而`isAtAll`字段自动生成的getter、setter为：

```java
public boolean isAtAll() {
    return isAtAll;
}

public void setAtAll(boolean atAll) {
    isAtAll = atAll;
 }
```

对应的fastjson中对方法名的处理在`com.alibaba.fastjson.util.TypeUtils.computeGetters`中,源码摘录如下：

```java
if(methodName.startsWith("is")){
    if(methodName.length() < 3){
        continue;
    }
    if(method.getReturnType() != Boolean.TYPE
            && method.getReturnType() != Boolean.class){
        continue;
    }
    String propertyName;
    Field field = null;
    if(Character.isUpperCase(c2)){
        if(compatibleWithJavaBean){
            propertyName = decapitalize(methodName.substring(2));
        } else{
            propertyName = Character.toLowerCase(methodName.charAt(2)) + methodName.substring(3);
        }
        propertyName = getPropertyNameByCompatibleFieldName(fieldCacheMap, methodName, propertyName, 2);
    }
```

也就是说，对于is开头的方法，fastjson先把第3个字符截取出来，如果该字符是大写，就转换为小写，并且拼装剩余的方法名组成属性名。

例如：`isAtAll`方法拼装出来的属性名就是`atAll`，最终结果就是把我们的`is`
给吃了！

## 3 解决办法

既然问题根源已经找到了，那我们只需要对症下药就可以了，这里针对不同应用场景，课代表给大家总结三种解决办法：

1. 对于有修改权限的代码，要严格遵守开发规范，POJO类中的布尔类型属性不要用`is`开头命名，如果有，改掉
2. 对于第三方接口，参数里有类似`isXXX`这样的参数，可以在对应属性字段上使用fastjson的`@JSONField(name = "anotherName")`来定制属性名
3. 可以手动修改getter和setter方法名

## 4 引申

SpringBoot集成了jackson，默认使用jackson来进行json序列化，经过测试，jackson也存在吃掉`is`的情况，原理与fastjson类似，就不做过多解释了，开发过程中遵守文中提及的解决办法即可

## 参考

* fastjson源码
* 《阿里开发规范泰山版》(2020.04.22)

---

![qr-code-1270x300](https://zhengxl5566.github.io/img/javaHelper/qr-code-1270x300.png)
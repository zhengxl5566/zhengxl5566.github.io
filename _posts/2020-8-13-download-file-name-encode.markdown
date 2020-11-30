---
layout: post
title:  "下载的附件名总乱码？你该去读一下 RFC 文档了！"
date:   2020-8-13 08:00:00 +0800
categories: Java
---

> 纸上得来终觉浅，绝知此事要躬行

Web 开发过程中，相信大家都遇到过附件下载的场景，其中，各浏览器下载后的文件名中文乱码问题或许一度让你苦恼不已。

网上搜索一下，大部分都是通过`Request Headers`中的`UserAgent`字段来判断浏览器类型，根据不同的浏览器做不同的处理，类似下面的代码：

```java
// MicroSoft Browser
if (agent.contains("msie") || agent.contains("trident") || agent.contains("edge")) {
  // filename 特殊处理
}
// firefox
else if (agent.contains("firefox")) {
  // filename 特殊处理
}
// safari
else if (agent.contains("safari")) {
  // filename 特殊处理
}
// Chrome
else if (agent.contains("chrome")) {
  // filename 特殊处理
}
// 其他
else{
 // filename 特殊处理
}
//最后把特殊处理后的文件名放到head里
response.setHeader("Content-Disposition",
                    "attachment;fileName=" + filename);
```

不过，这样的代码看起来很魔幻，为什么每个浏览器的处理方式都不一样？难道每次新出一个浏览器都要做兼容吗？就没有一个统一标准来约束一下这帮浏览器吗？

带着这个疑惑，我翻阅了 RFC 文档，最终得出了一个优雅的解决方案：

```java
// percentEncodedFileName 为百分号编码后的文件名
response.setHeader("Content-disposition",
        "attachment;filename=" + percentEncodedFileName +
        ";filename*=utf-8''" + percentEncodedFileName);
```

经过测试，这段响应头可以兼容市面上所有主流浏览器，由于是 HTTP 协议范畴，所以语言无关。只要按这个规则设置响应头，就能一劳永逸地解决恼人的附件名中文乱码问题。

接下来课代表带大家抽丝剥茧，通过阅读 RFC 文档，还原一下这个响应头的产出过程。

## 1. Content-Disposition

一切要从 [RFC 6266](https://tools.ietf.org/html/rfc6266) 开始，在这份文档中，介绍了`Content-Disposition`响应头，其实它并不属于`HTTP`标准，但是因为使用广泛，所以在该文档中进行了约束。它的语法格式如下：

```java
content-disposition = "Content-Disposition" ":"
                            disposition-type *( ";" disposition-parm )

     disposition-type    = "inline" | "attachment" | disp-ext-type
                         ; case-insensitive
     disp-ext-type       = token

     disposition-parm    = filename-parm | disp-ext-parm

     filename-parm       = "filename" "=" value
                         | "filename*" "=" ext-value
```

其中的`disposition-type`有两种：

* inline 代表默认处理，一般会在页面展示
* attachment 代表应该被保存到本地，需要配合设置`filename`或`filename*`

注意到`disposition-parm`中的`filename`和`filename*`，文档规定：这里的信息可以用于保存的文件名。

它俩的区别在于，filename 的 value 不进行编码，而`filename*`遵从 [RFC 5987](https://tools.ietf.org/html/rfc5987)中定义的编码规则：

```java
Producers MUST use either the "UTF-8" ([RFC3629]) or the "ISO-8859-1" ([ISO-8859-1]) character set.
```

由于`filename*`是后来才定义的，许多老的浏览器并不支持，所以文档规定，当二者同时出现在头字段中时，需要采用`filename*`，忽略`filename`。

至此，响应头的骨架已经呼之欲出了，摘录 [RFC 6266] 中的示例如下：

```java
 Content-Disposition: attachment;
                      filename="EURO rates";
                      filename*=utf-8''%e2%82%ac%20rates
```

这里对`filename*=utf-8''%e2%82%ac%20rates`做一下说明，这个写法乍一看可能会觉得很奇怪，它其实是用单引号作为分隔符，将等号右边分成了三部分：第一部分是字符集(`utf-8`)，中间部分是语言(未填写)，最后的`%e2%82%ac%20rates`代表了实际值。对于这部分的组成，在[RFC 2231](https://tools.ietf.org/html/rfc2231).section 4 中有详细说明：

```java
 A single quote is used to
   separate the character set, language, and actual value information in
   the parameter value string, and an percent sign is used to flag
   octets encoded in hexadecimal.
```

## 2.PercentEncode

PercentEncode 又叫 Percent-encoding 或 URL encoding.

正如前文所述，`filename*`遵守的是[RFC 5987] 中定义的编码规则，在[RFC 5987] 3.2中定义了必须支持的字符集：

```java
recipients implementing this specification
MUST support the character sets "ISO-8859-1" and "UTF-8".
```

并且在[RFC 5987] 3.2.1规定，百分号编码遵从 [RFC 3986](https://tools.ietf.org/html/rfc3986).section 2.1中的定义，摘录如下：

```java
A percent-encoding mechanism is used to represent a data octet in a
component when that octet's corresponding character is outside the
allowed set or is being used as a delimiter of, or within, the
component.  A percent-encoded octet is encoded as a character
triplet, consisting of the percent character "%" followed by the two
hexadecimal digits representing that octet's numeric value.  For
example, "%20" is the percent-encoding for the binary octet
"00100000" (ABNF: %x20), which in US-ASCII corresponds to the space
character (SP).  Section 2.4 describes when percent-encoding and
decoding is applied.
```

注意了，**[RFC 3986]** 明确规定了**空格 会被百分号编码为`%20`**

而在另一份文档 [RFC 1866](https://tools.ietf.org/html/rfc1866).Section 8.2.1 *The form-urlencoded Media Type* 中却规定：

```java
The default encoding for all forms is `application/x-www-form-
   urlencoded'. A form data set is represented in this media type as
   follows:

        1. The form field names and values are escaped: space
        characters are replaced by `+', and then reserved characters
        are escaped as per [URL]
```

这里要求`application/x-www-form-urlencoded`类型的消息中，空格要被替换为`+`,其他字符按照[URL]中的定义来转义，其中的[URL]指向的是[RFC 1738](https://tools.ietf.org/html/rfc1738) 而它的修订版中和 URL 有关的最新文档恰恰就是 **[RFC 3986]**

这也就是为什么很多文档中描述空格(white space)的百分号编码结果都是 `+`或`%20`，如：

w3schools:`URL encoding normally replaces a space with a plus (+) sign or with %20.`

MDN:`Depending on the context, the character ' ' is translated to a '+' (like in the percent-encoding version used in an application/x-www-form-urlencoded message), or in '%20' like on URLs.`

那么问题来了，开发过程中，对于空格符的百分号编码我们应该怎么处理？

课代表建议大家遵循最新文档，因为 [RFC 1866] 中定义的情况仅适用于`application/x-www-form-urlencoded`类型， 就百分号编码的定义来说，我们应该以 **[RFC 3986]** 为准，所以，任何需要百分号编码的地方，都应该将空格符 百分号编码为`%20`，stackoverflow 上也有支持此观点的答案：[When to encode space to plus (+) or %20?](https://stackoverflow.com/questions/2678551/when-to-encode-space-to-plus-or-20)

## 3. 代码实践

有了理论基础，代码写起来就水到渠成了，直接上代码：

```java
@GetMapping("/downloadFile")
public String download(String serverFileName, HttpServletRequest request, HttpServletResponse response) throws IOException {

    request.setCharacterEncoding("utf-8");
    response.setContentType("application/octet-stream");

    String clientFileName = fileService.getClientFileName(serverFileName);
    // 对真实文件名进行百分号编码
    String percentEncodedFileName = URLEncoder.encode(clientFileName, "utf-8")
            .replaceAll("\\+", "%20");

    // 组装contentDisposition的值
    StringBuilder contentDispositionValue = new StringBuilder();
    contentDispositionValue.append("attachment; filename=")
            .append(percentEncodedFileName)
            .append(";")
            .append("filename*=")
            .append("utf-8''")
            .append(percentEncodedFileName);
    response.setHeader("Content-disposition",
            contentDispositionValue.toString());
    
    // 将文件流写到response中
    try (InputStream inputStream = fileService.getInputStream(serverFileName);
         OutputStream outputStream = response.getOutputStream()
    ) {
        IOUtils.copy(inputStream, outputStream);
    }

    return "OK!";
}
```

代码很简单，其中有两点需要说明一下：

1. `URLEncoder.encode(clientFileName, "utf-8")`方法之后，为什么还要`.replaceAll("\\+", "%20")`。

   正如前文所述，我们已经明确，任何需要百分号编码的地方，都应该把 空格符编码为 `%20`，而`URLEncoder`这个类的说明上明确标注其会将空格符转换为`+`:

   > The space character &quot; &nbsp; &quot; is converted into a plus sign &quot;{@code +}&quot;.

   其实这并不怪 JDK，因为它的备注里说明了其遵循的是`application/x-www-form-urlencoded`( PHP 中也有这么一个函数，也是这么个套路)

   > Translates a string into {@code application/x-www-form-urlencoded} format using a specific encoding scheme. This method uses the

   所以这里我们用`.replaceAll("\\+", "%20")` 把`+`号处理一下，使其完全符合 **[RFC 3986]** 的百分号编码规范。这里为了方便说明问题，把所有操作都展现出来了。当然，你完全可以自己实现一个`PercentEncoder`类，丰俭由人。

2. [RFC 6266] 标准中`filename=`的`value`是不需要编码的，这里的`filename=`后面的 value 为什么要百分号编码？

   回顾 [RFC 6266] 文档， `filename`和`filename*`同时出现时取后者，浏览器太老不支持新标准时取前者。

   目前主流的浏览器都采用自升级策略，所以大部分都支持新标准------除了老版本IE。老版本的IE对 value 的处理策略是 进行百分号解码 并使用。所以这里专门把`filename=`的`value`进行百分号编码，用来兼容老版本 IE。

   PS：课代表实测 IE11 及 Edge 已经支持新标准了。

## 4. 浏览器测试

根据下图 statcounter 统计的 2019 年中国市场浏览器占有率，课代表设计了一个包含中文，英文，空格的文件名 `下载-down test .txt`用来测试

![browser-ranking.png](https://zhengxl5566.github.io/img/download-file-name-encode/browser-ranking.png)

测试结果：

| Browser         | Version        | pass |
| --------------- | -------------- | ---- |
| Chrome          | 84.0.4147.125  | true |
| UC              | V6.2.4098.3    | true |
| Safari          | 13.1.2         | true |
| QQ Browser      | 10.6.1(4208)   | true |
| IE              | 7-11           | true |
| Firefox         | 79.0           | true |
| Edge            | 44.18362.449.0 | true |
| 360安全浏览器12 | 12.2.1.362.0   | true |
| Edge(chromium)  | 84.0.522.59    | true |

根据测试结果可知：基本已经能够兼容市面上所有主流浏览器了。

## 5.总结

回顾本文内容，其实就是浏览器兼容性问题引发的附件名乱码，为了解决这个问题，查阅了两类标准文档：

1. HTTP 响应头相关标准

   [RFC 6266]、[RFC 1866]

2. 编码标准

   [RFC 5987]、[RFC 2231]、[3986]、[1738]

我们以 [RFC 6266] 为切入点，全文总共引用了 6 个 [RFC] 相关文档，引用都标明了出处，感兴趣的同学可以跟着文章思路阅读一下原文档，相信你会对这个问题有更深入的理解。文中代码已上传 [github](https://github.com/zhengxl5566/springboot-demo)

最后不禁要感叹一下：规范真是个好东西，它就像 Java 语言中的 `interface`，只制定标准，具体实现留给大家各自发挥。

如果觉得本文对你有帮助，欢迎收藏、分享、在看三连

## 6.参考资料

[1]RFC 6266: *https://tools.ietf.org/html/rfc6266*

[2]RFC 5987: *https://tools.ietf.org/html/rfc5987*

[3]RFC 2231: *https://tools.ietf.org/html/rfc2231*

[4]RFC 3986: *https://tools.ietf.org/html/rfc3986*

[5]RFC 1866: *https://tools.ietf.org/html/rfc1866*

[6]RFC 1738: *https://tools.ietf.org/html/rfc1738*

[7]When to encode space to plus (+) or %20?: *https://stackoverflow.com/questions/2678551/when-to-encode-space-to-plus-or-20*

---

2020年8月19日更新：

文中内容已合并进入开源框架 [若依](https://gitee.com/y_project/RuoYi) （pull request：[196](https://gitee.com/y_project/RuoYi/pulls/196)）

---
![](https://zhengxl5566.github.io/img/javaHelper/qr-code-1270x300.png)


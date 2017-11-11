## web开发常见安全问题

不是所有 Web 开发者都有安全的概念，甚至可能某些安全漏洞从来都没听说过。这就是这篇科普文章的存在意义，希望 Web 开发者在开发时能依此逐条检查代码中的安全问题。

> 注：服务器运维相关的安全注意事项不在本文之列

## 前言 {#-}

水桶底部只要有一个洞，水就能全部流光。Web 安全同理。

## 前端安全 {#-}

前端安全主要表现为通过浏览器间接影响到用户数据的安全问题。

### 1. XSS 漏洞 {#xss-}

[XSS \(Cross-Site Scripting\)](https://en.wikipedia.org/wiki/Cross-site_scripting)，是一个我觉得耳熟能详的前端安全问题。通过构造特殊数据，在用户浏览器上执行特定脚本，从而造成危害（如以用户身份发帖、转账等）。

由于页面内 JavaScript 几乎可以完成各种事情，因此一旦网站上有 XSS 漏洞，那些没有验证码等确认措施的操作大多都能不知情地完成，其危害甚大。

来看看几个存在 XSS 漏洞的例子吧：

#### 1. Case A: HTML DOM {#case-a-html-dom}

```
<a href="/user/1">{{ user_name }}</a>
```

**Exploit:**

```
<script>
   alert(1)
</script>
```

**Result:**

```
<a href="/user/1"><script>alert(1)</script></a>
```

最基本的例子，如果此处不对 `user_name` 中的特殊符号进行 escape，就会造成 XSS。

#### 2. Case B: HTML Attribute {#case-b-html-attribute}

```
<img src="{{ image_url }}">
```

**Exploit:**

```
"onerror="alert(1)

```

**Result:**

```
<img src=""onerror="alert(1)">
```

这个例子表明，如果只对尖括号进行 escape 是不够的，很多时候引号也需要被 escape。简单来说，对不同输出场景，需要使用不同的 escape 规则。

#### 3. Case C: JavaScript {#case-c-javascript}

```
<script>var user_data ={{ user_data|json_encode }};</script>
```

**Exploit:**

```
{"exploit": "</script><script>alert(1);//"}
```

**Result:**

```
<script>var user_data ={"exploit":"</script><script>alert(1);//"};</script>
```

这是一个特别的例子，大多数人觉得，对于输出在 `<script>` 中的内容，`json_encode` 一下就安全了，其实不然。在这个例子中，XSS 仍然发生了。

更可怕的是，不少编辑器的代码高亮方案中甚至无法看出上面的 Result 中存在 XSS 漏洞。如 Sublime Text 下，代码高亮结果是这样的，看上去没有任何问题：

![](https://dn-breeswish-org.qbox.me/web-sec-xss-1.png)

但是浏览器解析出来的结果是这样的：

![](https://dn-breeswish-org.qbox.me/web-sec-xss-2.png)

#### 解决方法： {#-}

1. 在不同上下文中，使用合适的 escape 方式

2. 不要相信 任何 来自用户的输入（不仅限于 POST Body，还包括 QueryString，甚至是 Headers）

### 2. CSRF 漏洞 {#csrf-}

[CSRF \(Cross-site request forgery\)](https://en.wikipedia.org/wiki/Cross-site_request_forgery)，是一个知名度不如 XSS 但是却同样具有很大杀伤力的安全漏洞。它的杀伤力大正是因为很多开发者不知道这个漏洞。

举个栗子，如果你网站 A 上的「登出」功能是这样实现的：

```
<a href="http://a.com/logout.php">登出</a>
```

则存在 CSRF 漏洞。假设网站 B（当然也可以是网站 A 本身）中有这么一段代码：

```
<img src="http://a.com/logout.php">
```

那么当用户访问的时候，就会导致网站 A 上的会话被登出。

需要注意的是，不只是 GET 类请求，POST 类请求同样会存在 CSRF 漏洞，例如网站 B 中：

```
<form action="http://a.com/transaction" method="POST" id="hack">
<input type="hidden" name="to" value="hacker_account">
<input type="hidden" name="value" value="100000">
</form>
<script>
document.getElementById("hack").submit();
</script>
```

那么用户访问网站 B 的时候，便会自动携带 A 的 SESSION 信息，成功 POST `/transaction` 到网站 A。

这个漏洞危害很大，例如以前某些 BTC 交易所就存在这个漏洞，一旦用户被诱骗访问了黑客精心布置的网站就会造成资金损失；又例如，以前某中国著名社交网站也存在这个漏洞，更糟糕的是该网站还允许用户递交自己的 `<img>` 显示在所有人的信息流中，且某些传播性操作可以用 `GET` 方式达成，这就导致黑客可以进行病毒式扩散，当然，仅仅是把 `<img src="logout.do">` 嵌入信息流也是有足够大杀伤力的 :\)

#### 解决方法： {#-}

1. 给所有请求加上 token 检查。token 一般是随机字符串，只需确保其不可预测性即可。token 可以在 QueryString、POST body 甚至是 Custom Header 里，但千万不能在 Cookies 里。

2. 检查 `referer` （请注意，这往往不能防御来自网站自身的 CSRF 攻击，如用户评论中的 `<img>` 就是一个常见触发点）

## 后端安全 {#-}

在这里，后端安全指的是对服务器数据等可能造成直接影响的安全问题。黑客一般会直接构造数据进行攻击。

### 1. SQL 注入漏洞 {#sql-}

SQL 注入漏洞应该也是一个大多数开发者都知道的漏洞。

考察以下 PHP 代码：

```
<?php
$user = mysql_query('SELECT * FROM USERS WHERE UserName="'.$_GET['user'].'"');
```

那么当请求中 user 参数为 `";DROP TABLE USERS;--` 时，合成的 SQL 语句是：

```
SELECT 
*
 FROM USERS WHERE 
UserName
=
""
;
DROP TABLE USERS
;--
"
```

这里产生什么结果就不需要解释了吧 :\) 另外，通过 SQL 注入，往往还能拿到系统权限。

#### 解决方法： {#-}

所有 SQL 语句都使用参数化查询（推荐）或对参数进行 escape（不推荐）

2. ### 权限控制漏洞 {#-}

   这个就不仔细展开了，未经授权可以进行的操作都是权限控制漏洞。

   例如，某些网站的后台操作就仗着「以为用户不知道入口地址」不进行任何权限检查，又例如，某些操作可能出现不允许更改的字段被用户递交更改（往往是那些网页上标记为 `disabled` 或者 `hidden` 的字段），再例如，允许通过 `../` 访问到不应该被访问的文件等（一般存在于 `include` 中）。

   #### 解决方法： {#-}

   所有地方都要进行权限检查（如是否已登录、当前用户是否有足够权限、该项是否可修改等），总之，不要相信任何来自用户的数据，URL 当然也是。

3. ### SESSION 与 COOKIE {#session-cookie}

   Session 和 Cookie 是两种用于存储用户当前状态的工具。某些开发者不了解 Session 与 Cookie 的区别，误用或者混用，导致敏感信息泄露或者信息篡改。

   Cookie 存储在浏览器上，用户可以查看和修改 Cookie。  
   Session 是存储在服务端的数据，一般来说安全可靠；大多数 Session 都是基于 Cookie 实现的（在 Cookie 中存储一串 SESSION\_ID，在服务器上存储该 SESSION\_ID 对应的内容）。

4. ### IP 地址 {#ip-}

   首先，用户的 IP 地址一般存储在 `REMOTE_ADDR` 中，这是唯一的可信的 IP 地址数据（视不同语言而定）。然后某些代理服务器，会将用户的真实 IP 地址附加在 header 的 `VIA` 或 `X_FORWARDED_FOR` 中（因为`REMOTE_ADDR` 是代理服务器自身的 IP）。所以，要获取用户 IP 地址，一般做法是，判断是否存在 `VIA` 或者 `X_FORWARDED_FOR` 头，如果存在，则使用它们，如果不存在则使用 `REMOTE_ADDR`。这也是网上大多数所谓教程提供的方法。

   这就产生问题了，`X_FORWARDED_FOR` 或 `VIA` 是 HTTP Header，换句话说，它们是可以被伪造的。例如，在投票中，如果采信了 `X_FORWARDED_FOR`，往往意味着被刷票。

   #### 解决方法： {#-}

   只使用 `REMOTE_ADDR` 作为获取 IP 的手段。

5. ### 验证码 {#-}

   验证码里常见的问题有：非一次性、容易被识别。

   非一次性指的是，同一个验证码可以一直被用下去。一般来说，每进行一次验证码校对（无论正确与否），都应该强制更换或清除 Session 中的验证码。

   关于识别问题，在当前科技水平下，不加噪点不加扭曲的验证码几乎是 100% 可识别的。所以大家自己看着办吧…

## 后记 {#-}

本文仅仅列举了一些常见的 Web 开发安全问题，还有一些很少出现但确实有危害的安全问题没有出现在这里（例如，点击劫持），读者可以自行了解。

总之，这是一篇科普文，希望开发者可以了解并重视 Web 开发安全问题。

  


另附篇讲的比较好的文章

http://www.haomou.net/2015/10/30/2015\_webxss/




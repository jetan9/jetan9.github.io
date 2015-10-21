---
layout: post
category : leetcode
title: LeetCode代码下载脚本
tagline: ""
tags : [python, fiddler, leetcode]
---
{% include JB/setup %}

## 概述
自从接触LeetCode，也做了一些题目，于是考虑把解出来的题目代码提交到GitHub上，这样也方便以后复习或者提高代码运行的速度。第一步是把所有的代码下载下来，直接手工操作显然比较繁琐，于是考虑写脚本来完成这个任务。这样，以后解出了新的题目，只需要重新运行脚本，然后提交生成的代码即可。

## 思路与分析
LeetCode已提交的代码页面需要登录之后才能查看，所以首先需要登录。其次需要获取所有正确答案的链接，然后跳转到相应的页面提取提交的代码并写入本地文件。

## 实现过程
由于近期工作中Python用得比较多，所以这次也用Python来做。

### 登录LeetCode
LeetCode的登录页面是<https://oj.leetcode.com/accounts/login/>。

开始时为了省事起见，直接用了一个人人发帖机的登录代码，结果总是报403错误，显然并没有登录成功。反复检查也没有发现问题所在，于是只能老老实实祭出[Fiddler][]抓包分析登录过程。

打开登录页面发出的请求：

```
GET https://oj.leetcode.com/accounts/login/ HTTP/1.1
Accept: text/html, application/xhtml+xml, */*
Referer: https://oj.leetcode.com/accounts/login/
Accept-Language: zh-Hans-CN,zh-Hans;q=0.8,en-US;q=0.5,en;q=0.3
User-Agent: Mozilla/5.0 (Windows NT 6.3; WOW64; Trident/7.0; Touch; rv:11.0) like Gecko
Accept-Encoding: gzip, deflate
Host: oj.leetcode.com
DNT: 1
Connection: Keep-Alive
Cookie: _ga=GA1.2.1448150336.1424922143; _gat=1; csrftoken=lEyElcazhTP555owuvMqz5pDllgUXZ3p; __atuvc=7%7C8
```

填入用户名和密码之后，表单被重新提交到登录页面（请求中的用户名和密码已被替换为\****）：

```
POST https://oj.leetcode.com/accounts/login/ HTTP/1.1
Accept: text/html, application/xhtml+xml, */*
Referer: https://oj.leetcode.com/accounts/login/
Accept-Language: zh-Hans-CN,zh-Hans;q=0.8,en-US;q=0.5,en;q=0.3
User-Agent: Mozilla/5.0 (Windows NT 6.3; WOW64; Trident/7.0; Touch; rv:11.0) like Gecko
Content-Type: application/x-www-form-urlencoded
Accept-Encoding: gzip, deflate
Host: oj.leetcode.com
Content-Length: 93
DNT: 1
Connection: Keep-Alive
Cache-Control: no-cache
Cookie: _ga=GA1.2.1448150336.1424922143; _gat=1; csrftoken=lEyElcazhTP555owuvMqz5pDllgUXZ3p; __atuvc=7%7C8

csrfmiddlewaretoken=lEyElcazhTP555owuvMqz5pDllgUXZ3p&login=****&password=****
```

可以看到提交表单时除了用户名和密码之外，还需要提交一项csrfmiddlewaretoken。通过查看网页源代码得知这是一个隐藏域，并且其值和Cookie中的csrftoken的值是相同的。个人猜测由于之前使用的发帖机登录代码只提交了用户名和密码，并没有处理csrfmiddlewaretoken，所以被后台代码判定为恶意登录从而予以拒绝。上网搜了一下CSRF，发现原来是跨站请求伪造（cross site request forgery）的缩写，而在表单中加入隐藏域并随机赋值则是一种抵御攻击的通行做法。这也证实了我之前的猜测。

解决方法看来很简单了，即在第一次打开登录页面之后，自动提取csrfmiddlewaretoken的值，然后和帐户名/密码一并提交。不过，改好的脚本仍然报403错误，这不由得让我对自己的判断力产生了些许怀疑。对脚本进行抓包分析之后发现，脚本生成的请求头和浏览器生成的请求头相比缺失了好几项，于是尝试逐项分析。结果在添加了Referer之后就成功登录了。看来后台代码对提交表单时的Referer也做了检查，如果不是指定的值则直接予以拒绝。

### 提取代码链接
已提交代码页面是分页的，第1页的URL为<https://oj.leetcode.com/submissions/1/>，第2页的URL为<https://oj.leetcode.com/submissions/2/>，依此类推。所以可以从第1页开始构造URL，访问页面并提取代码链接，直到通过某个构造出来的URL访问的页面中没有任何代码链接为止。

已提交代码页面的源文件中，表示一个正确提交代码的部分类似：

```html
<tr>
	<td>1 month, 1 week ago</td>
	<td>
		<a class="inline-wrap" href="/problems/department-highest-salary/">Department Highest Salary</a>
	</td>
	<td>
		<a class="text-danger status-accepted" href="/submissions/detail/19601352/"><strong>Accepted</strong></a>
	</td>
	<td>
		1101 ms
	</td>
	<td>mysql</td>
</tr>
```

由于一道题目可能提交了多个正确答案，这里需要选择运行时间最少的，同时在保存代码时希望以题目名称命名文件，所以需要提取的信息除了代码对应的链接之外，还需要提取题目名称（在题目链接中）以及运行时间。可以采用如下的正则表达式来完成：

```
'href="/problems/(.*)/".*\s*</td>\s*<td>\s*.*href="/(submissions/detail/[0-9]*/).*Accepted.*\s*</td>\s*<td>\s*(.*) ms'
```
	
提取出来的信息保存在一个字典中，其键为题目名称，值为一个列表。列表的第1个元素为代码链接，第2个元素为运行时间。
	
### 提取代码
通过代码链接打开代码页面之后，可以查看页面源文件：

```html
...
scope.code.cpp = 'class Solution {\u000D\u000Apublic:\u000D\u000A    int removeDuplicates(int A[], int n) {\u000D\u000A\u0009\u0009if (n \u003D\u003D 0) {\u000D\u000A\u0009\u0009\u0009return 0\u003B\u000D\u000A\u0009\u0009}\u000D\u000A        int len \u003D 1\u003B\u000D\u000A\u0009\u0009for(int i \u003D 1\u003B i \u003C n\u003B i++) {\u000D\u000A\u0009\u0009\u0009if (A[i] !\u003D A[len \u002D 1]) {\u000D\u000A\u0009\u0009\u0009\u0009A[len++] \u003D A[i]\u003B\u000D\u000A\u0009\u0009\u0009}\u000D\u000A\u0009\u0009}\u000D\u000A\u0009\u0009return len\u003B\u000D\u000A    }\u000D\u000A}\u003B';
...
```

代码部分可以通过如下的正则表达式来提取：

```
"scope.code.*'([\s\S]*)';"
```
	
通过转码之后即可得到原始内容：

```c++
class Solution {
public:
	int removeDuplicates(int A[], int n) {
		if (n == 0) {
			return 0;
		}
		int len = 1;
		for(int i = 1; i < n; i++) {
			if (A[i] != A[len - 1]) {
				A[len++] = A[i];
			}
		}
		return len;
	}
};
```
	
写入文件的时候需要使用'wb'选项，否则会插入多余的空行。

## 总结
最终的脚本已提交至[GitHub][]。

这个脚本还有比较多可以改进的地方。例如：

* 将不同语言的答案存为特定格式的源文件而非统一的txt文件。
* 应用多线程提高下载代码的速度，等等。

另外，围绕这个脚本也可以做一些其他的事情。例如将脚本放到某个长期开机的机器上作为定时任务来跑，这样以后做题的时候就不用手动跑脚本去同步了。

[Fiddler]: http://www.telerik.com/fiddler
[GitHub]: https://github.com/jetan9/LeetCode/blob/master/spider.py
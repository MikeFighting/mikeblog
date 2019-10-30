---
title: 正则表达式最佳实践
date: 2017-05-26 16:15:00
tags: Objective-C
---

![Regular-Expressions](http://upload-images.jianshu.io/upload_images/1513759-47f810d76025ba94.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
作为一名开发人员，无论是前端，后端，移动端，都可能会接触到正则表达式，最常见的场景就是注册登录了，我们需要对电话号码或者邮箱做校验，如果对用户名有特殊字符有限制的话还会对特殊字符做校验。我们通常的做法就是**百度**或者**谷歌**，最后复制粘贴完事，对于那一串奇奇怪怪的字符串感觉很头大。接下来的这篇文章会向你详细解释正则表达式的语法，分析正则表达式，并且附带上常用的表格，最后给出IOS中用到正则表达式的两个类`NSRegularExpress`，`NSPredicate`，NSString。

## 主要内容包括

一、简介
二、正则表达式的PlayGround
三、基本语法表及简介
四、正则表达式实例
五、正则表达式在IOS中
六、常用的正则表达式

## 简介

正则表达式就是一个用来检索或者替换合乎某种条件的一串字符集。

## 正则表达式的PlayGround

在学习正则表达式时，面对很多的长篇大论，可能比较枯燥无味，写代码跑项目又比较费时费力。[regexpal网站](http://www.regexpal.com/)（**需要翻墙**）刚好可以解决我们的问题，它可以随时测试我们写的正则是否有误，并且有语法检查和语法提示。在接下来的说明中，可以边看边操作了。
![正则表达式的PlayGround](http://upload-images.jianshu.io/upload_images/1513759-a0b9b72716437239.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

## 基本语法表及简介

![正则表达式常用指令集](http://upload-images.jianshu.io/upload_images/1513759-c6cb378e6098d705.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

* 纯文本形式，比如`a`就将匹配文本中的a,如果`Mike`就会匹配文本中的Mike，文本之间是`与`的关系。
* **\** 其中`\`是转意字符，表示该字符后面的字母有特殊含义，比如下面要说的`\d`,`\b`等，因为在很多语言中，比如0C，Swift中`\`已经是转意字符，所以需要`\\b`来表示`\b`的含义。
* **[]**匹配里面的任何一个字符，比如`p[abcde]`，将匹配pa，pb，pc，pd，pe。当然可以换成`p[a-e]`,其中`-`表示“至”的意思，`[0-9]`表示0到9间的任何一个数字。
* **{}**如上表，表示的是匹配的次数，如{6}，表示匹配六次，{5,}表示匹配5次以上。比较难以理解的是{2,4}?，这表示最少匹配两次，最多匹配四次，但是，如果四个字母同时出现了，就算两个匹配，如果三个字母出现了，就匹配两次。如：正则`[A-Z]{2,3}?`，检测`MIKEF`,就会产生两个匹配`MI`和`KE`，可以在上文的网站中练习。
* **.**匹配任何*一个字符*，比如`M.M`，匹配MuM，MdM，M@M，等。
* **\w**匹配`很想单词的字符`，包含字母，数字，下划线，但是不包含标点符号，及其他字符，比如：`hello\w`，匹配hello_，hello8，但是不匹配`hello!`。
* **\d**匹配数字，其和[0-9]是同意的。例如`\d\d?:\d\d`就是可以匹配时间，比如`12:30`和`9:20`等。digital单词的首字母
* **\b**表示文字的边界，比如空格和标点符号。如`go\b`将会匹配go home和go!但是不会匹配gone，在需要匹配整个单词的时候往往有用。boundary单词的首字母
* **\s**表示空格以及新的一行。比如`Hey\s`将会匹配`Hey man!`中的`Hey`。
* **^**表示一行的开头，比如`^Hello`将会匹配`Hello Everyone!`但是不会匹配`He said Hello`。注意：在[]里面的`^`表示的非的意思，如：`[^DE]`表示的是：不是DE的任何字符。
* **$**表示行的结尾，例如end$将会匹配`it was the end`但是不会匹配`the end is comming`。
* **\***表示匹配其前面的字符0次或者很多次，如：`go\*d`,将会匹配good,goood,gooood,goooooood,gd等。
* **+**表示匹配其前面的字符一次或者很多次，如：`go+d`将不会匹配`gd`。

注：关于强匹配的概念，如果想了解的可以！[在正则ExpressionsInfo网站上学习](http://www.regular-expressions.info/possessive.html,shang),`捕获`的意思就是被捕获的信息可以利用`$n`的形式来获取并且用来做替换。由于其使用不是很多，就不在赘述。

## 正则表达式实例

经过上面对正则表达式基本语法的讲解及练习，我们来使用试着写几个正则表达式。

### 英文名字校验，规则如下：

* 名字: 标准的英文字母，1到10个字母组成，首字母大写
* Middle Name简写：标准英文字母，1个字母，大小写都可以
* 姓：标准的英文字母，可能有`'`(只能出现一个)，比如： O’Brien，长度在2到10个字母，首字母大写'

根据上面的表格我们很容易写出这样的

名字：`^[A-Z][a-z]{1,9}$`,其中^表示一行的开始[A-Z]表示第一个字母大写[a-z]表示中间的是小写字母，最后{1,9}表示1到9个字母，最后结尾是$
MiddleName: `^[a-x]|[A-Z]$`,其中|表示或的意思
姓：`^[A-Z]'?[a-z]{1,9}$`，其中'?表示'可以出现一次也可以不出现

### 日期，规则如下：

日期应该在1/1/1900到31/12/2099年之间，并且日期的格式必须是`dd/mm/yyyy`或`dd-mm-yyyy`或者`dd.mm.yyyy`这三种格式：参考上面的速查表，我们可以写出如下的正则表达式来

```bash
^0[1-9]|([1-2]\d)|3[01][/-.]0[1-9]|[1][012][/-.](19|20)\d\d$
```

其中日期：`0[1-9]|([1-2]\\d)|3[01]`,也就是穷举了所有的可能01,02,03...9,然后1或者2拼上\d，最后是3可能是30和31
月份：`0[1-9]|[1][012]`,和前面的日期类似，并且更少了，大于10的之后10,11,12几种情况
年份：`(19|20)\d\d`前面是可能出现的年分19和20，后面就是任意0-9的组合
分隔符：`[/-.]`只可能出现这三种情况，所以用[]括起来即可
  
### 日期加强版，规则如下：

格式是xx/xx/xx或xx.xx.xx或xx-xx-xx，分别是月，日，年，如：10-05-12表示12年10月5日，其中月分可以是英文全拼也可能是缩写，比如January->Jan，February->Feb，日期可能第几天，比如1st，2nd之类的，月日年之间可以有不定的几个空格，如March 13th, 2001：

```bash
(\d{1,2}[-/.]\d{1,2}[-/.]\d{1,2})|(Jan(uary)?|Feb(ruary)?|Mar(ch)?|Apr(il)?|May|Jun(e)?|Jul(y)?|Aug(ust)?|Sep(tember)?|Oct(ober)?|Nov(ember)?|Dec(ember)?)\s*\d{1,2}(st|nd|rd|th)?+[,]\s*\d{4}
```

我们可以先用（）将其切开，在用|将其切开
先看全是数字的：`(\d{1,2}[-/.]\d{1,2}[-/.]\d{1,2})`表示两个数字，两个数字的组合，如：`10-05-12`
然后看字母类型的`(Jan(uary)?|Feb(ruary)?|Mar(ch)?|Apr(il)?|May|Jun(e)?|Jul(y)?|Aug(ust)?|Sep(tember)?|Oct(ober)?|Nov(ember)?|Dec(ember)?`穷举了所有的月份信息，然后`\s*`表示任意多个空格，然后是日期`\d{1,2}(st|nd|rd|th)?`再加若干个空格：`\s*`最后是四位数字。

### 时间，规则如下：

时间可以一位或者两位，数字，然后可以有若干个空格，最后是am或者pm，经过以上几个例子，我们不难写出：`\d{1,2}\s*[ab]m`这样的正则表达式

## 正则表达式在iOS中的应用

### NSRegularExpress

在IOS开发中，我们经常使用这个类来做有关文本校验筛选工作，对于对象的校验经常使用`NSPredicate`，其使用非常简单，主要包含创建，查找，替换几个方法：

```objc
NSError *error = NULL;
NSString *pattern = @"正则表达式";
NSString *string = @"需要校验的文本";
NSRange range = NSMakeRange(0, string.length);
NSRegularExpression *regex = [NSRegularExpression regularExpressionWithPattern:pattern options:NSRegularExpressionCaseInsensitive error:&error]; // 创建RegularExpression对象
NSArray *matches = [regex matchesInString:string options:NSMatchingProgress range:range]; // 找到校验的结果matches中是NSTextCheckingResult对象，该对象中包含各种被查找对象的信息
// 下面两个方法还可以帮们替换掉找到的字符
- (NSString *)stringByReplacingMatchesInString:(NSString *)string options:(NSMatchingOptions)options range:(NSRange)range withTemplate:(NSString *)templ;
- (NSUInteger)replaceMatchesInString:(NSMutableString *)string options:(NSMatchingOptions)options range:(NSRange)range withTemplate:(NSString *)templ;
```

### NSPredicate和正则表达式结合

```objc
NSString    *regularExpression = @"正则表达式";
NSPredicate *numberPre = [NSPredicate predicateWithFormat:@"SELF MATCHES %@",regularExpression];
return [numberPre evaluateWithObject:textString];
```

### NSString的方法

```objc
-(NSRange)rangeOfString:(NSString *)aString options:(NSStringCompareOptions)mask;
NSRange range = [searchedText rangeOfString:@"正则表达式" options:NSRegularExpressionSearch];
```

## 常用的正则表达式

1.验证用户名和密码：”^[a-zA-Z]\w{5,15}$”
2.验证电话号码：（”^(\\d{3,4}-)\\d{7,8}$”）
eg：021-68686868  0511-6868686；
3.验证手机号码：”^1[3|4|5|7|8][0-9]\\d{8}$”；
4.验证身份证号（15位或18位数字）：”\\d{14}[[0-9],0-9xX]”；
5.验证Email地址：(“^\\w+([-+.]\\w+)*@\\w+([-.]\\w+)*\.\\w+([-.]\\w+)*$”)；
6.只能输入由数字和26个英文字母组成的字符串：(“^[A-Za-z0-9]+$”) ;
7.整数或者小数：^[0-9]+([.]{0,1}[0-9]+){0,1}$
8.只能输入数字：”^[0-9]*$”。
9.只能输入n位的数字：”^\\d{n}$”。
10.只能输入至少n位的数字：”^\\d{n,}$”。
11.只能输入m~n位的数字：”^\\d{m,n}$”。
12.只能输入零和非零开头的数字：”^(0|[1-9][0-9]*)$”。
13.只能输入有两位小数的正实数：”^[0-9]+(.[0-9]{2})?$”。
14.只能输入有1~3位小数的正实数：”^[0-9]+(\.[0-9]{1,3})?$”。
15.只能输入非零的正整数：”^\+?[1-9][0-9]*$”。
16.只能输入非零的负整数：”^\-[1-9][]0-9″*$。
17.只能输入长度为3的字符：”^.{3}$”。
18.只能输入由26个英文字母组成的字符串：”^[A-Za-z]+$”。
19.只能输入由26个大写英文字母组成的字符串：”^[A-Z]+$”。
20.只能输入由26个小写英文字母组成的字符串：”^[a-z]+$”。
21.验证是否含有^%&’,;=?$\”等字符：”[^%&',;=?$\x22]+”。
22.只能输入汉字：”^[\u4e00-\u9fa5]{0,}$”。
23.验证URL：”^http://([\\w-]+\.)+[\\w-]+(/[\\w-./?%&=]*)?$”。
24.验证一年的12个月：”^(0?[1-9]|1[0-2])$”正确格式为：”01″～”09″和”10″～”12″。
25.验证一个月的31天：”^((0?[1-9])|((1|2)[0-9])|30|31)$”正确格式为；”01″～”09″、”10″～”29″和“30”~“31”。
26.获取日期正则表达式：\\d{4}[年|\-|\.]\\d{\1-\12}[月|\-|\.]\\d{\1-\31}日?
评注：可用来匹配大多数年月日信息。
27.匹配双字节字符(包括汉字在内)：[^\x00-\xff]
评注：可以用来计算字符串的长度（一个双字节字符长度计2，ASCII字符计1）
28.匹配空白行的正则表达式：\n\s*\r
评注：可以用来删除空白行
29.匹配HTML标记的正则表达式：<(\S*?)[^>]*>.*?</>|<.*? />
评注：网上流传的版本太糟糕，上面这个也仅仅能匹配部分，对于复杂的嵌套标记依旧无能为力
30.匹配首尾空白字符的正则表达式：^\s*|\s*$
评注：可以用来删除行首行尾的空白字符(包括空格、制表符、换页符等等)，非常有用的表达式
31.匹配网址URL的正则表达式：[a-zA-z]+://[^\s]*
评注：网上流传的版本功能很有限，上面这个基本可以满足需求
32.匹配帐号是否合法(字母开头，允许5-16字节，允许字母数字下划线)：^[a-zA-Z][a-zA-Z0-9_]{4,15}$
评注：表单验证时很实用
33.匹配腾讯QQ号：[1-9][0-9]\{4,\}
评注：腾讯QQ号从10 000 开始
34.匹配中国邮政编码：[1-9]\\d{5}(?!\d)
评注：中国邮政编码为6位数字
35.匹配ip地址：((2[0-4]\\d|25[0-5]|[01]?\\d\\d?)\.){3}(2[0-4]\\d|25[0-5]|[01]?\\d\\d?)。

## 延伸阅读：

1. https://www.raywenderlich.com/30288/nsregularexpression-tutorial-and-cheat-sheet
2. http://www.regexpal.com/
3. http://nshipster.com/nspredicate/
4. http://nshipster.com/nssortdescriptor/

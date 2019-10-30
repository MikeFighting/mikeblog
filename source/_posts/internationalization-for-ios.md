---
title: iOS中的国际化
date: 2016-06-01 08:05:00
tags: Translation
---

IOS中，如果系统的语言或者地区变化了，我们怎样让App中显示的语言, 日期，数字，货币单位的格式随着变化呢？下面将介绍IOS中简单的国际化的方法：

在GitHub上下载一个[需要国际化的工程:](https://github.com/MikeFighting/Bilingual)
打开这个工程你可以当看到如下的一个界面：

<img src="http://upload-images.jianshu.io/upload_images/1513759-5f34f732ca292932.png" width="320" heigt="568" style="display:block;margin:0 auto">

然后点开`StoryBoard`，你会发现里面的控件都非常简单。为了国际话，我们需要往项目中再添加一门语言。添加语言的方式是，`Project--->Info--->Localizations`点击"+"来添加相应语言，这里我们选择Chinese(simplified)简体中文。然后将弹出的对话框中的LaunchScreen.strings和main.storyBoard都勾选了。这样我们的基本工作就完成了，下面正式开始:

## 创建.strings文件

创建`.strings`文件，点击command + N 新建文件，在Resouce中选择Strings File文件，命名为Localizable，这样系统就会在不同语言环境下选择不同的.strings文件进行加载，显示不同的语言。

## 处理Localizable.strings

点击`Localizable.strings`文件，点击右侧的Localization按钮，然后选择需要本地化的语言，选择汉语，然后再次点击Localizable.strings，你会发现右侧的Localizattion下面多了几个选择语言的选项，Base,English,Chinese(simplified)选中English，然后就会发现又多了一个.strings文件。

## 配置.string文件

对不同的.string文件配置不同的Value值，在.string中配置的格式是"KEY" = "VALUE";,  注意最后的分号。我们在localizable.strings(English)中加上

```bash
"I write %@ lines every day." = "I write %@ lines every day.";
"Do you like coding ?" = "Do you like coding ?";
```

在Localizable.strings(Chinese simplified)中添加

```bash
"I write %@ lines every day." = "我每天写%@行代码。";
"Do you like coding ?" = "你喜欢编程吗？";
```

## 使用方法

在ViewController.m中改写相应的加载方法将原来的

```bash
_viewControllerNumLabel.text = @"I write 1000000000 lines every day.";
[_viewControllerLikeButotn setTitle:@"Do you like coding ?" forState:UIControlStateNormal];
```

替换为:

```bash
_viewControllerNumLabel.text = [NSString stringWithFormat:NSLocalizedString(@"I write %@ lines every day.", nil),@1000000000];
[_viewControllerLikeButotn setTitle:NSLocalizedString(@"Do you like coding ?", nil) forState:UIControlStateNormal];
```

这时候将系统的语言设置成简体中文，`General -> International -> Language -> Chinese`，然后再重新运行App，你会发现变其中的一个Label和一个Button上的文字改变了。

## 图片的处理

修改不同的图片。这时你会发现，有些时候，我们的图片上会有文字，这些文字是不能用代码改变的，所以这时要加载不同的图片，利用步骤4中的方法，我们给图片名用一个Key来表示，然后将不同的图片名作为value写在不同的.strings文件中，我们在English中加入：

`"imageName" = "english";`
我们在Chinese中加入：`"imageName" = "chinese";`
其中english和chinese都是图片的文字，然后在ViewControlelr中添加如下代码

```bash
_viewControllerImageView.image = [UIImage imageNamed:NSLocalizedString(@"imageName",nil)];
```

这时，修改相应的语言就将会出现不同的图片。

## 处理StoryBoard

修改没有被引出的控件的显示。点击Main.StoryBoard,你会发现下面有一个Main.strings(Chinese Simplified)文件，这时将里面有关Lable的代码：

```bash
/* Class = "UILabel"; text = "Hello I am a Lable"; ObjectID = "zki-n6-dit"; */
"zki-n6-dit.text" = "Hello I am a Lable";
```

修改为：

```bash
/* Class = "UILabel"; text = "Hello I am a Lable"; ObjectID = "zki-n6-dit"; */
"zki-n6-dit.text" = "您好，我是一个Label";
```

再次运行代码，你会发现这个时候没有用代码修改的Lable显示的内容也变了。

## 数字的处理

还有一个细节问题，数字的格式，运行App时候你会发现原来显示的格式是1000000000，但是我国用的数字表示方式应该是1,000,000,000，西班牙用的数字表示方式是:1.000.000.000,这个怎么国际化呢？这个时候需要在ViewController中添加如下代码:

```bash
NSNumberFormatter *numberFormatter = [NSNumberFormatter new];
numberFormatter.numberStyle =NSNumberFormatterDecimalStyle;
NSString *numberString = [numberFormatter stringFromNumber:@1000000000];
_viewControllerNumLabel.text = [NSString stringWithFormat:NSLocalizedString(@"I write %@ lines every day.", nil),numberString];
```

这里利用NSNumberFormatter来进行对数字格式的转化,这里需要注意，再运行App的时候需要将相应的地区也设置成相应的  General -> International -> Region Format -> China

## App显示国际化

如何让App显示的名字也国际化？这时需要添加一个plist文件InfoPlist.strings，然后将这个文件进行Localizable，在生成的相应的english和chinese中添加需要显示的App的名字，如：在Chinese的文件中添加:"CFBundleDisplayName" = "双语者",重新运行App就会发现App的名字也随着语言的改变也改变了，最后App国际化的结果为:

<img src="http://upload-images.jianshu.io/upload_images/1513759-8391f5cf97d6771b.png" width="320" height="568" style="display:block; margin:0 auto">

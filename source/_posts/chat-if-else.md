---
title: 聊聊if else
date: 2017-09-05 07:04:01
tags: Others
---

![if else](http://upload-images.jianshu.io/upload_images/1513759-12027b6db51d7301.png)
if else 是我们学习C语言开始用的流程控制语句。还记得大学老师说过一句话，任何复杂的业务逻辑都可以用if else去解决。然而就像面向对象中的继承一样，如果用的过多就会造成代码的腐烂。下面我们就来聊聊if else。

## 为什么太多的if else不好？

我们先看一个例子：

```swift
       public void OnMessage(Push.PushMessage pushMessage) {
        try {
            String message = pushMessage.messageContent;
            Log.v("keyes", "pushMessage = " + message);
            if (message != null) {
                JsonObject data = new JsonParser().parse(message).getAsJsonObject();
                if (data.get("type") != null) {
                    String type = data.get("type").getAsString();
                    if ("100".equals(type)) {//XX消息
                        EventBus.getDefault().post(new GrabActionEvent());
                        PopBean bean = PushUtils.dealPushMessage(mContext, message);
                        IDealWithPush dealWithPush = AppStateUtils.getAppState(mContext);
                        dealWithPush.dealPush(mContext, bean);
                    } else if ("106".equals(type)) {// 
                        String nick = data.get("nick").getAsString();
                        WPushNotify.notification(106, nick, "正在访问您的信息，请立即回复");
                    } else {
                        Gson temp = new Gson();
                        final SystemNotification bean = temp.fromJson(pushMessage.messageContent, SystemNotification.class);
                        if (bean.getType() == 103) {
                            //应用在后台，不需要刷新UI,通知栏提示新消息
                            if (!AppInfoUtils.isRunningForeground(HyApplication.getApplication())) {
                                WPushNotify.notification(bean);
                            }
                            saveDataToDB(bean);
                            EventBus.getDefault().post(bean);
                        } else if (bean.getType() == 104) {
                            List<Activity> list = HyApplication.getInstance().getActivityList();
                            if (list != null && list.size() > 0) {
                                Activity activity = list.get(list.size() - 1);
                                if (!TextUtils.isEmpty(bean.getDescribe())) {
                                    new LogoutDialog(activity, bean.getDescribe());
                                } else {
                                    new LogoutDialog(activity, activity.getString(R.string.force_exit));
                                }
                            }
                        } else if (bean.getType() == 108) {//沉默用户唤醒
                            String title = bean.getTitle();
                            String describe = bean.getDescribe();
                            WPushNotify.notification(108, title, describe);
                        }
                    }
                }
            }
        } catch (Exception e) {
            LogUtils.e("push", e.getMessage());
        }

        Log.i("song", pushMessage.messageContent);

    }
```                 

这个方法里面仅仅嵌套了好多层if else，看上去会比较复杂难懂，再看一个例子：

```swift
if(isSkipPPUForQA && StringUtils.isNotBlank(request.getParameter("userId"))){
    super.doFilter(request, response, chain);
}else {
    try {
    	if("/app/school/article/share".equals(request.getRequestURI())){
    		super.doFilter(request, response, chain);
    	}else{
    		if(!filterReqUrl(request)) {
                long ppuUserId = PassportService.passportService.getLoginUserId(RemoteValid.SAPCE_ONE_HOUR, request, response);
                if (ppuUserId < 2) {
response.getWriter().write(this.generateResponse(AppResultStateEnum.PPU_UNVALID.getCodeStr(), "登录认证信息已过期，请重新登录"));
                    log.error("ppu返回的userId:" + ppuUserId + ",ppu过期,ppu=" + PPUCookieUtil.getPpuCookie(request) + ",url=" + request.getRequestURI());
                    log.error("imei=" + request.getParameter("imei") + ",version=" + request.getParameter("version") + ",platform=" + request.getParameter("platform"));
                } else {
                    boolean flag = isTouchSingleDeviceLimitWithoutLogin(ppuUserId, request);
                    if (flag) {
                        String singleDeviceLoginContent = configComp.getValueByConfigTable(ConfigEnum.APP_SINGLE_DEVICE_LOGIN_CONTENT);
                        boolean isH5 = "1".equals(request.getParameter("isH5"));
                        if(isH5){
                            String jsAjaxHeader = request.getHeader("X-Requested-With");
                            if("XMLHttpRequest".equals(jsAjaxHeader)){response.getWriter().write(this.generateResponse(AppResultStateEnum.SINGLE_DEVICE_LOGIN.getCodeStr(), singleDeviceLoginContent));
                            }else {
                             renderSingleDeviceHtml(response,"/single_device_error");
                            }
                        }else {
                            response.getWriter().write(this.generateResponse(AppResultStateEnum.SINGLE_DEVICE_LOGIN.getCodeStr(), singleDeviceLoginContent));
                        }
                        log.error("触发单设备登录限制错误");
                    } else {
                        request.addParameter("userId", new String[]{ppuUserId + ""});
                        super.doFilter(request, response, chain);
                    }
                }
            }
    	}
    } catch (Exception e) {
        logger.error("业务处理异常,url=",e);
    }
}
```          

第一，这个方法中嵌套了七层的if else，层次太多。第二这个方法太长。嵌套层次过多和方法过长都是Bad Smell。那么究竟很多的if else有哪些弊端呢？

* **僵化**：如果再有更多情况的时候，我们需要在原来的地方写更多的if...else if条件。也就是说你需要去改动原来的代码，然后重新编译，重新部署，这是很浪费时间的。并且这违背了面向对象中的**开放封闭原则**：对扩展开放，对修改封闭。同时由于这个类需要处理各种业务，职责太多，所以也违背了**职责单一原则。**

* **效率低下**：很多系统的类，比如`HashMaps`，`Properties`等，都非常注意基于数据的条件判断。

* **难阅读**：像这种层层if else嵌套的情况，如果其他人需要来看，并且维护这份代码，由于难阅读，他们会感觉吃力。试想下，如果段代码很长，一个屏看不完，那肯定是维护的灾难。

* **难维护**：if else不像switch case，它的每个分支都和其它分支有关系，如果需求变更，在修改某个分支之前要看懂其它所有分支，确保不会对其它分支造成影响。

* **难调试**：很多if else，调试过程中需要一步步跟进，会影响调试效率。

* **难测试**：每次我们写测试用例的Case，针对每个有很多if else的方法，我们要对每个分支都写一个测试，这样下来这个测试用例将会变得非常长。

>在任何面向对象语言中，都需要考虑移除分支控制逻辑（`if`以及`switch`，`case`）。移除的常用做法是将这些控制逻辑的方法移到一个类中。 [Quora,Simon Hayes](https://www.quora.com/Why-should-Java-programmers-try-to-avoid-if-statements)

那么我们怎样来解决这种情况，因为遇到不同的情况需要用不同的解决方案，我们逐个来分析：

## 常见的嵌套类if else处理方式

比如第一个if else的例子，通常

## 平行类的if else处理方式

**如果我们有几个判断条件是平级if，那么我们可以使用命令模式来解决这种问题。**比如我们现在有如下的if else：

```swift       
if (value.equals("A")) { doCommandA() }
else if (value.equals("B")) { doCommandB() } 
else if etc.
```    

这时，我们可以使用**命令模式**来解决，先创建一个接口：

```swift
public interface Command {
     void exec();
}
```       

然后`CommandA`和`CommandB`类实现这个接口：

```swift
public class CommandA() implements Command {

     void exec() {
          // ... 
     }
}
public class CommandB() implements Command {
     
     void exec() {
          // ...
     }
}
```       

然后创建一个`Map<String,Command>`,并且往其中添加Command实例：

```swift 
commandMap.put("A", new CommandA());
commandMap.put("B", new CommandB());
```   
                
然后所有的**if/else if**，就都会变成：

```swift
commandMap.get(value).exec();
```     

如果某个Command有任何的改变只需要改动某个具体的类即可，如果有新加的Command，那么只需要添加响应的Command即可。命令模式就是加了一个**中间件：命令容器**(就是这里的Map，根据情况可能会是List或者其它)来实现解耦。

## 复杂处理算法的if else

如果我们if之后的代码处理的业务逻辑很相似，并且这种处理算法可能会经常变动，比如：

```swift
public class IfElseDemo {
    public double calculateInsurance(double income, InputType type) {
        if (type == smallType) {
            return income*0.365;
        } else if (type == mediumType) {
            return (income-10000)*0.2+35600;
        } else if (type == bigType) {
            return (income-30000)*0.1+76500;
        } else {
            return (income-60000)*0.02+105600;
        }
    }
}
```         

那么我们就可以将每个if分支中的代码单独分离到各个类中，然后再抽出一个父类，这样我们每个条件分支中就不会有很多代码了：

```swift     
public abstract class InsuranceStrategy {
    public double calculateInsuranceVeryHigh(double income) {
        return (income - getAdjustment()) * getWeight() + getConstant();
    }
    public abstract int getConstant();
    public abstract double getWeight();
    public abstract int getAdjustment();
}

public class InsuranceStrategyMedium extends InsuranceStrategy {
    @Override
    public int getConstant() {
        return 35600;
    }
    @Override
    public double getWeight() {
        return 0.2;
    }
    @Override
    public int getAdjustment() {
        return 10000;
    }
}

/*InsuranceStrategyLow和InsuranceStrategyHigh的处理方式相似，此处略去*/
class IfElseDemo {
    private InsuranceStrategy strategy;
    public double calculateInsurance(double income, InputType type) {
        if (type == smallType) {
            strategy = new InsuranceStrategyLow();
        } else if (type == mediumType) {
            strategy = new InsuranceStrategyMedium();
        } else if (type == bigType) {
            strategy = new InsuranceStrategyHigh();
        } else {
            strategy = new InsuranceStrategyVeryHigh();
        }
        
        return strategy.calculate(income);
    }
}
```      

这样最终不还是有if else吗？是的，最终还是有if else，但是if else的逻辑变得非常清晰，只是用于创建一个新的类。并且我们将经常变化的算法部分封装到了子类中，如果某个子类中的算法变了，只需要变动某个子类（**封装变化**），然后重新编译就可以了，不需要将整个项目重新编译，部署。

## 区间类的if else

如果客户端的if条件表示的是不同的范围，然后根据不同范围来选择不同的对象来处理，比如：

```java
public class Client {

    public  static  void  main(String[] args) {
    
        Request request = new Request();
        request.addSalaryAmount = 9999;
        if (request.addSalaryAmount <= 100){
            DivisionManager divisionManager = new DivisionManager();
            divisionManager.accept();
        }else if (request.addSalaryAmount <= 1000){
            Chief chief = new Chief();
            chief.accept();
        }else if (request.addSalaryAmount <= 10000){
            GeneralManager generalManager  = new GeneralManager();
            generalManager.accept();
        }else {
            System.out.println("金额太大没人能批准");
        }
    }
}
```                   
  
上面这个例子中，不同的条件分支是让不同的对象来处理这种条件。并且以后可能Request对象会添加其他的请求属性，比如offWork（请假），并且这种请求属性同样需要`DivisionManager`，`Chief`，`GeneralManager`。然而其中的处理顺序变了，并不是现在的请求等级。可能是先由`Chief`处理，再有`GeneralManager`处理，最后有`DivisionManager`来处理，那怎么办呢？难道还要写一套if else吗？
这时候我们就可以用责任链模式来将这一长串if else嵌套进每一个对象中去，我们可以这样做：

```java
interface ManagerCommand{
    public void accept(int requestAmount);
}

abstract class CommManager implements ManagerCommand {
    public CommManager superior;
    public void setSuperior(CommManager superior) {
        this.superior = superior;
    }
}

public class DivisionManager extends CommManager {

    @Override
    public void accept(int requestAmount) {

        if (requestAmount<100){
            System.out.println("部门经理批准");
        }else if ( this.superior != null){
            this.superior.accept(requestAmount);
        }
    }
}

public class Chief extends CommManager{

    public void  accept(int requestAmount)  {
        if (requestAmount < 1000){
        System.out.println("总监同意");
        }else if (this.superior != null){
            this.superior.accept(requestAmount);
        }
    }
}
public class GeneralManager extends CommManager{

    @Override
    public void accept(int requestAmount) {

        if (requestAmount<10000){
            System.out.println("总经理批准");
        }else if (this.superior != null){
            this.superior.accept(requestAmount);
        }
    }
}
```        

最后在Client端调用的时候，我们可以这样写：

```java
public class Client {

    public  static  void  main(String[] args) {

        Request request = new Request();
        request.addSalaryAmount = 999;
        DivisionManager divisionManager = new DivisionManager();
        Chief chief = new Chief();
        GeneralManager generalManager = new GeneralManager();
        divisionManager.setSuperior(chief);
        chief.setSuperior(generalManager);
        divisionManager.accept(request.addSalaryAmount);
    }
}
```    

这种写法的好处是：**将条件和处理该条件的对象解耦，每个处理条件的对象都不知道其他对象，我们可以随时地增加或者修改处理一个请求的结构。这增加了给对象指派职责的灵活性**。

>小结：其实上述的每种方式都是利用**多态**来解决分支带来的僵化，[谷歌有一个视频对这个问题阐述得很好。](https://www.youtube.com/watch?v=4F72VULWFvc)。


## 参考资料

1. https://stackoverflow.com/questions/10175805/how-to-avoid-a-lot-of-if-else-conditions
2. https://stackoverflow.com/questions/271526/avoiding-null-statements?rq=1
3. https://stackoverflow.com/questions/14136721/converting-many-if-else-statements-to-a-cleaner-approach
4. https://www.youtube.com/watch?v=4F72VULWFvc
5. https://www.quora.com/Why-should-Java-programmers-try-to-avoid-if-statements
6. https://stackoverflow.com/questions/1199646/long-list-of-if-statements-in-swift/1199677#1199677
7. https://industriallogic.com/xp/refactoring/conditionalWithStrategy.html
8. 大话设计模式
9. 重构改善既有代码的设计



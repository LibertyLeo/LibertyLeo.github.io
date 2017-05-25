---
layout: post
title: Runtime常用实例
date: 2017-03-14 13:36:25 +08:00
tags: 舞刀弄枪
---

## 管中窥豹
本篇主要围绕以下九个方面进行用法介绍：

| 用法 | 关键函数 |
| --- | --- |
动态获取类名 | `const char *class_getName(Class cls)`
动态获取类的成员变量 | `Ivar *class_copyIvarList(Class cls, unsigned int *outCount)`、<br />`const char *ivar_getName(Ivar v)`、<br />`const char *ivar_getTypeEncoding(Ivar v)`
动态获取类的属性列表 | `objc_property_t *class_copyPropertyList(Class cls, unsigned int *outCount)`、<br />`const char *property_getName(objc_property_t property)`
动态获取类的实例方法列表 | `Method *class_copyMethodList(Class cls, unsigned int *outCount)`、<br />`SEL method_getName(Method m)`
动态获取类所遵循的协议列表 | `Protocol * __unsafe_unretained *class_copyProtocolList(Class cls, unsigned int *outCount)`、<br />`const char *protocol_getName(Protocol *p)`
动态添加新的方法 | `Method class_getInstanceMethod(Class cls, SEL name)`、<br />`IMP method_getImplementation(Method m)`、<br />`const char *method_getTypeEncoding(Method m)`、<br />`BOOL class_addMethod(Class cls, SEL name, IMP imp, const char *types)`
类的实例方法实现交换 | `Method class_getInstanceMethod(Class cls, SEL name)`、<br />`void method_exchangeImplementations(Method m1, Method m2)`
动态属性关联 | `void objc_setAssociatedObject(id object, const void *key, id value, objc_AssociationPolicy policy)`、<br />`id objc_getAssociatedObject(id object, const void *key)`
消息发送与消息转发机制 | `+ (BOOL)resolveInstanceMethod:(SEL)sel`、<br />`- (id)forwardingTargetForSelector:(SEL)aSelector`、<br />`- (NSMethodSignature *)methodSignatureForSelector:(SEL)aSelector`、<br />`- (void)forwardInvocation:(NSInvocation *)anInvocation`

## 磨刀霍霍
> 我们首先封装一个测试类TestClass，其中需要包含遵守协议，并添加公有属性、私有属性、私有成员变量、公有实例方法、私有实例方法、类方法等。    
> 这些内容的添加主要便于之后的测试。

![TestClass方法变量声明.png](http://upload-images.jianshu.io/upload_images/3074427-19267050508b29d9.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

![TestClass私有变量.png](http://upload-images.jianshu.io/upload_images/3074427-22b00ab1f0769c54.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

![TestClass方法实现.png](http://upload-images.jianshu.io/upload_images/3074427-d356cd39dd97b548.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

## 一、动态获悉类结构
### 1. 动态获取类名
        采用`class_getName(cls)`在运行时获取类的名称。将char类型的指针转换成NSString类型进行返回。

![获取类名.png](http://upload-images.jianshu.io/upload_images/3074427-6e8f46d99e270e37.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

### 2. 动态获取成员变量
        采用`class_copyIvarList(cls, count)`获取成员变量列表。使用`ivar_getName(variable)`来输出成员变量名称，`ivar_getTypeEncoding(variable)`来输出成员变量类型。    
        我们通过将所得数据组合成NSDictionary来存储单个变量，若干个字典组成NSArray作为属性列表的返回。

![获取成员变量.png](http://upload-images.jianshu.io/upload_images/3074427-ba236c4ae0886f17.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

        使用TestClass进行用例测试。由于是调用上述方法获取TestClass的成员变量，到了运行时阶段实际就不存在公有私有之分。OC中的类在ARC情况下添加的属性，其实就是自动生成其get方法与set方法。    
        所有获取的成员列表中肯定带有成员属性，成员属性的名称前方带有下划线用于成员变量进行区分。    
        下方中各基本类型由特殊字母代替，可以看出i代表int类型，c代表bool类型，d表示double类型，f表示float类型。而如果是引用类型则直接是一个字符串显示，比如NSString类型就是@"NSString"。

![测试成员列表打印.png](http://upload-images.jianshu.io/upload_images/3074427-7d103a20f9e43f71.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

### 3. 动态获取成员属性列表
        上方获取了类的成员变量，那么下方进行属性列表的获取。属性区分于变量主要是它们拥有完整的set方法和get方法。    
        我们使用`class_copyPropertyList(cls, count)`来获取属性列表，通过`property_getName(property)`来获取属性名称。

![获取属性列表.png](http://upload-images.jianshu.io/upload_images/3074427-c6b057ad72869487.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

        下方dynamic的属性是我们使用runtime进行动态添加的。

![测试属性列表打印.png](http://upload-images.jianshu.io/upload_images/3074427-a994e7f820add3a6.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

### 4. 获取类的实例方法
        我们通过`class_copyMethodList(cls, count)`来获取实例方法列表，通过`method_getName(method)`来获取实例方法名称。

![获取实例方法.png](http://upload-images.jianshu.io/upload_images/3074427-69e8c49ef4437ad1.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

        下方打印了所有TestClass类的实例方法，当然包括成员属性的set方法和get方法。**其中`.cxx_destruct`方法不确认归属于何处，也许dealloc方法的自我实现？**

![测试实例方法列表打印.png](http://upload-images.jianshu.io/upload_images/3074427-66f763ed020ceaa7.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

### 5. 获取类的协议列表
        我们使用`class_copyProtocolList(cls, count)`来获取协议列表，使用`protocol_getName(protocol)`来获取协议名称

![获取协议列表.png](http://upload-images.jianshu.io/upload_images/3074427-11dd45e18315f535.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

--- 

## 二、动态操作类方法
### 1. 动态添加方法实现
        其添加原理旨在使用`class_getInstanceMethod(cls, methodName)`获取相关的方法声明以及使用`method_getImplementation(method)`获取相关的方法实现。将它们进行组合后，使用`class_addMethod(cls, methodName, method, type)`进行方法的添加。

![动态添加方法实现.png](http://upload-images.jianshu.io/upload_images/3074427-9df1842bea2f9984.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

### 2. 实现方法交换
        通过`class_getInstanceMethod(cls, methodName)`获取到需要交换的两个方法，直接使用`method_exchangeImplementation(methodA, methodB)`进行方法替换即可。

![实现方法交换.png](http://upload-images.jianshu.io/upload_images/3074427-5c4422c03177acdc.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

        通过类目为测试类封装一个针对交换方法的测试用例。
- 如果是普通情况下，没有交换。在replaceMethod中调用本身势必会造成死循环。
- 如是如果交换方法成功，那么此时在replaceMethod中调用replaceMethod，**其实此时调用的是exchangeMethodA**。由于exchangeMethodA不存在死循环，故在测试时，调用了封装的交换方法后，进一步又调用了replaceMethod，其实只是调用了exchangeMethodA而已。
![交换方法的封装.png](http://upload-images.jianshu.io/upload_images/3074427-2bf0d28ff56ea180.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

## 三、属性关联
        属性关联可以说是runtime最普通的打开方式了。通过为属性声明一个静态名称，调用`void objc_setAssociatedObject(id object, const void *key, id value, objc_AssociationPolicy policy)`实现新增属性的set方法，调用`id objc_getAssociatedObject(id object, const void *key)`实现新增属性的get方法即可。
![动态添加属性.png](http://upload-images.jianshu.io/upload_images/3074427-6e5efa33f0326869.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

## 四、消息处理与消息转发
- **消息处理过程:**
- 当你调用一个类的方法时，先在本类中的方法缓存列表中进行查询，如果在缓存列表中找到了该方法的实现，就执行；如果找不到就在本类中的方法列表中进行查找。
- 在本类方法列表中查找到相应的方法实现后就进行调用，如果没找到，就去父类中进行查找；如果在父类中的方法列表中找到了相应方法的实现。

当在方法缓存列表，本类中的方法列表以及父类中的方法列表中都找不到相应的实现，到程序崩溃以前还会经历以下过程：

1. 消息处理

> - 如果一直寻找方法直到父类中都找不到方法实现时会执行`+ (BOOL)resolveInstanceMethod:(SEL)sel`类方法。
- 如果返回NO，则表明不做任何处理，继续下一步。如果返回YES，就说明该方法中对找不到实现的方法进行了处理。    
- 我们就可以在此方法中为找不到实现的SEL动态添加一个方法实现，添加完毕后，就会执行我们添加的方法实现。    
- 下一次程序再找不到该类某个方法的实现时，就不会因为找不到而崩溃了。

![消息处理.png](http://upload-images.jianshu.io/upload_images/3074427-267321a3f488a9f4.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

2.消息转发

> - 如果不对上述消息进行处理的话，也就是`+ (BOOL)resolveInstanceMethod:(SEL)sel`方法返回NO时。便进入了下一步消息转发。
- 即执行`- (id)forwardingTargetForSelector:(SEL)aSelector`方法。该方法会返回一个类对象，该类的对象有SEL对应的实现，当调用这个找不到方法时，就会转发到ExtClass中进行处理。
- 此时完成消息转发。如果该方法返回self或者nil，说明不对相应的方法进行转发，那就再走下一步。

![消息转发.png](http://upload-images.jianshu.io/upload_images/3074427-725069652b745e84.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

3.消息常规转发

> - 如果不将消息转发给其他类的对象，则此时代表自己进行处理。即上述的方法中返回self或者nil。
- 此时执行`- (NSMethodSignature *)methodSignatureForSelector:(SEL)aSelector`来获取方法的参数以及返回数据类型，即可以理解为该方法的签名。
- 如果此时再次返回nil，那么消息转发结束。程序崩溃，报出找不到相应的方法实现的崩溃消息。

- 下方方法执行的先决条件，是要在`+ (BOOL)resolveInstanceMethod:(SEL)sel`中返回NO。然后下方也是进行将方法转给ExtClass的实现。

![消息常规转发.png](http://upload-images.jianshu.io/upload_images/3074427-f7ab8e6f3bfe6b95.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

        本文项目Github链接地址:https://github.com/LibertyLeo/Runtime-Usage
---
layout: post
title: Runtime常用实例
date: 2017-03-14 13:36:25 +08:00
tags: 舞刀弄枪
---

## 管中窥豹
本篇主要根据以下方法进行相关功能的实现：

| 实现目的 | 相关函数 |
| ------ | ------ |
| 动态获取类名 | const char *class_getName(Class cls) |
| 动态获取类的成员变量 | Ivar *class_copyIvarList(Class cls, unsigned int *outCount), <br />const char *ivar_getName(Ivar v), <br />const char *ivar_getTypeEncoding(Ivar v) |
| 动态获取类的属性列表 | objc_property_t *class_copyPropertyList(Class cls, unsigned int *outCount), <br />const char *property_getName(objc_property_t property) |
动态获取类的实例方法列表 | Method *class_copyMethodList(Class cls, unsigned int *outCount), <br />SEL method_getName(Method m)
动态获取类所遵循的协议列表 | Protocol * __unsafe_unretained *class_copyProtocolList(Class cls, unsigned int *outCount), <br />const char *protocol_getName(Protocol *p)
动态添加新的方法 | Method class_getInstanceMethod(Class cls, SEL name), <br />IMP method_getImplementation(Method m), <br />const char *method_getTypeEncoding(Method m), <br />BOOL class_addMethod(Class cls, SEL name, IMP imp, const char *types)
类的实例方法实现交换 | Method class_getInstanceMethod(Class cls, SEL name), <br />void method_exchangeImplementations(Method m1, Method m2)
动态属性关联 | void objc_setAssociatedObject(id object, const void *key, id value, objc_AssociationPolicy policy), <br />id objc_getAssociatedObject(id object, const void *key)
消息发送与消息转发机制 | + (BOOL)resolveInstanceMethod:(SEL)sel, <br />- (id)forwardingTargetForSelector:(SEL)aSelector, <br />- (NSMethodSignature *)methodSignatureForSelector:(SEL)aSelector, <br />- (void)forwardInvocation:(NSInvocation *)anInvocation

## 磨刀霍霍
封装测试类TestClass。遵循了`NSCoding`、`NSCopying`协议，并添加了公私有属性、公私有实例方法、私有成员变量及类方法等。    
方法的实现以反馈清晰为主。

- 声明公有属性、公有实例方法、类方法

```objc
#import <Foundation/Foundation.h>

@interface TestClass : NSObject<NSCoding, NSCopying>

@property (nonatomic, copy) NSArray *publicPropertyA;
@property (nonatomic, copy) NSString *publicPropertyB;

+ (void)classMethod:(NSString *)value;

- (void)publicInstanceMethodA:(NSString *)valueA withValueB:(NSString *)valueB;
- (void)publicInstanceMethodB;
- (void)exchangeMethodA;

@end
```

- 声明**"私有"**属性、成员变量、实例方法、

```objc
#import "TestClass.h"

@interface TestClass () {
    NSInteger _varA;
    int _varB;
    BOOL _varC;
    float _varD;
    double _varE;
}

@property (nonatomic, strong) NSNumber *privatePropertyA;
@property (nonatomic, strong) NSMutableArray *privatePropertyB;
@property (nonatomic, copy) NSDictionary *privatePropertyC;

@end
```

- 方法实现

```objc
@implementation TestClass

+ (void)classMethod:(NSString *)value {
    NSLog(@"publicClassMethodWithValue: <%@>", value);
}

- (void)publicInstanceMethodA:(NSString *)valueA withValueB:(NSString *)valueB {
    NSLog(@"publicInstanceMethodA valueA: <%@> valueB: <%@>", valueA, valueB);
}

- (void)publicInstanceMethodB {
    NSLog(@"publicInstanceMethodB");
}

- (void)privateInstanceMethodA {
    NSLog(@"privateInstanceMethodA");
}

- (void)privateInstanceMetehodB {
    NSLog(@"privateInstanceMethodB");
}

#pragma mark - 动态交换方法时的实现
- (void)exchangeMethodA {
    NSLog(@"exchangeMethodA");
}
```

## 一、动态获悉测试类结构
### 1. 动态获取类名
调用`class_getName(cls)`在运行时获取类的名称，将char类型的指针转换成NSString类型进行返回。

- 类名的获取

```objc
/**
 获取类名
 
 @param cls 对应类
 @return 类名
 */
+ (NSString *)captureClassName:(Class)cls {
    const char *className = class_getName(cls);
    return [NSString stringWithUTF8String:className];
}
```

- 控制台输出

```
测试类的类名为TestClass
```

### 2. 动态获取成员变量
调用`class_copyIvarList(cls, count)`返回成员变量列表。    
调用`ivar_getName(variable)`返回成员变量名称。    
调用`ivar_getTypeEncoding(variable)`返回成员变量类型。

我们使用字典的方式进行单个变量的信息存储，若干个字典数据构成数组返回属性列表。

- 获取成员变量

```objc
/**
 获取类的成员变量
 
 @param cls 对应类
 @return 成员变量数组, 由成员变量类型与名称构成的若干个字典
 */
+ (NSArray *)captureIvarList:(Class)cls {
    unsigned int count = 0;
    //  该函数返回一个由Ivar指针组成的数组
    Ivar *ivarList = class_copyIvarList(cls, &count);
    
    NSMutableArray *mutableList = [NSMutableArray arrayWithCapacity:count];
    for (unsigned int i = 0; i < count; i++) {
        NSMutableDictionary *dic = [NSMutableDictionary dictionaryWithCapacity:2];
        //  分别获取变量名称与变量类型
        const char *ivarName = ivar_getName(ivarList[i]);
        const char *ivarType = ivar_getTypeEncoding(ivarList[i]);
        dic[@"ivarName"] = [NSString stringWithUTF8String:ivarName];
        dic[@"ivarType"] = [NSString stringWithUTF8String:ivarType];
        //  添加到可变字典中
        [mutableList addObject:dic];
    }
    free(ivarList);
    return [NSArray arrayWithArray:mutableList];
}
```

使用TestClass完成用例测试。    
测试上述方法时，由于运行时的成员变量便不存在公有私有之分。Objc中的类在ARC情况下添加的属性，其实就是自动添加其get方法与set方法。故获取到的成员列表中肯定包含成员属性，不过成员属性的名称前方通过下划线与成员变量进行区分。    
而在控制台打印出的基本类型通过特殊字母表示，其含义为：

| 字母 | 含义 |
| :---: | :---: |
| q | NSInteger |
| i | int |
| c | bool |
| f | float |
| d | double |

如果是引用类型则直接是一个字符串显示，比如`NSString`类型就是`@"NSString"`。

- 成员列表一览

```
测试类的成员变量列表(
        {
        ivarName = "_varA";
        ivarType = q;
    },
        {
        ivarName = "_varB";
        ivarType = i;
    },
        {
        ivarName = "_varC";
        ivarType = c;
    },
        {
        ivarName = "_varD";
        ivarType = f;
    },
        {
        ivarName = "_varE";
        ivarType = d;
    },
        {
        ivarName = "_publicPropertyA";
        ivarType = "@\"NSArray\"";
    },
        {
        ivarName = "_publicPropertyB";
        ivarType = "@\"NSString\"";
    },
        {
        ivarName = "_privatePropertyA";
        ivarType = "@\"NSNumber\"";
    },
        {
        ivarName = "_privatePropertyB";
        ivarType = "@\"NSMutableArray\"";
    },
        {
        ivarName = "_privatePropertyC";
        ivarType = "@\"NSDictionary\"";
    }
)
```

### 3. 动态获取成员属性列表
进一步进行属性列表的获取，属性于变量的主要区分点是它们拥有完整的set方法和get方法。    
调用`class_copyPropertyList(cls, count)`返回属性列表。    
调用`property_getName(property)`返回属性名称。

- 获取属性列表

```objc
/**
 获取类的属性列表, 包括公有属性、私有属性、延展中增加的属性
 
 @param cls 对应类
 @return 属性列表数组
 */
+ (NSArray *)capturePropertyList:(Class)cls {
    unsigned int count = 0;
    objc_property_t *propertyList = class_copyPropertyList(cls, &count);
    
    NSMutableArray *mutableLlist = [NSMutableArray arrayWithCapacity:count];
    for (unsigned int i = 0; i < count; i++) {
        const char *propertyName = property_getName(propertyList[i]);
        [mutableLlist addObject:[NSString stringWithUTF8String:propertyName]];
    }
    free(propertyList);
    return [NSArray arrayWithArray:mutableLlist];
}
```

- 属性列表一览

```
测试类的属性列表(
    privatePropertyA,
    privatePropertyB,
    privatePropertyC,
    publicPropertyA,
    publicPropertyB
)
```

### 4. 获取类的实例方法
调用`class_copyMethodList(cls, count)`返回实例方法列表。    
调用`method_getName(method)`返回实例方法名称。

- 获取实例方法

```objc
/**
 获取类的实例方法列表: 包括setter、getter, 对象方法等, 但不包括类方法
 
 @param cls 对应类
 @return 实例方法数组
 */
+ (NSArray *)captureMethodList:(Class)cls {
    unsigned int count = 0;
    Method *methodList = class_copyMethodList(cls, &count);
    
    NSMutableArray *mutableList = [NSMutableArray arrayWithCapacity:count];
    for (unsigned int i = 0; i < count; i++) {
        Method method = methodList[i];
        SEL methodName = method_getName(method);
        [mutableList addObject:NSStringFromSelector(methodName)];
    }
    free(methodList);
    return [NSArray arrayWithArray:mutableList];
}
```

- 实例方法一览

```
测试类的方法列表(
    "unknowMethod:",
    "publicInstanceMethodA:withValueB:",
    publicInstanceMethodB,
    privateInstanceMethodA,
    privateInstanceMetehodB,
    exchangeMethodA,
    publicPropertyA,
    "setPublicPropertyA:",
    publicPropertyB,
    "setPublicPropertyB:",
    privatePropertyA,
    "setPrivatePropertyA:",
    privatePropertyB,
    "setPrivatePropertyB:",
    privatePropertyC,
    "setPrivatePropertyC:",
    categoryMethod,
    "setDynamicProperty:",
    dynamicProperty,
    replaceMethod,
    swapMethod,
    ".cxx_destruct",
    "methodSignatureForSelector:",
    "forwardInvocation:",
    "forwardingTargetForSelector:"
)
```

> 上方控制台打印结果罗列了TestClass类的实例方法名，包括成员属性的set和get。    
> 其中`.cxx_destruct`方法经试验发现：    
- 只有ARC情况下该方法才会出现。不免猜想是否是Dealloc方法中未经释放的内存自行实现的机制。
- 只有当前类拥有实例变量时才会出现，且父类的实例变量不会导致子类拥有该方法。
- 出现该方法与变量是否被赋值，赋何值并无关系。

### 5. 获取类的协议列表
调用`class_copyProtocolList(cls, count)`返回协议列表。    
调用`protocol_getName(protocol)`返回协议名称。

- 获取协议列表

```objc
/**
 获取类的协议列表
 
 @param cls Class
 @return 协议列表数组
 */
+ (NSArray *)captureProtocolList:(Class)cls {
    unsigned int count = 0;
    __unsafe_unretained Protocol **protocolList = class_copyProtocolList(cls, &count);
    
    NSMutableArray *mutableList = [NSMutableArray arrayWithCapacity:count];
    for (unsigned int i = 0; i < count; i++) {
        Protocol *protocol = protocolList[i];
        const char *protocolName = protocol_getName(protocol);
        [mutableList addObject:[NSString stringWithUTF8String:protocolName]];
    }
    return [NSArray arrayWithArray:mutableList];
}
```

> 测试类中遵守NSCoding，NSCopying两个基本协议，由于无关内部实现，故在项目中不对相关协议方法进行实现。

- 遵守协议一览

```
测试类的协议列表(
    NSCoding,
    NSCopying
)
```

--- 

## 二、动态操作测试类方法
### 1. 动态添加方法实现
实现原理：
1. 调用`class_getInstanceMethod(cls, methodName)`返回相关的方法声明。
2. 调用`method_getImplementation(method)`返回相关的方法实现。
3. 综上两个条件，调用`class_addMethod(cls, methodName, method, type)`完成方法添加。

- 动态添加方法-实现

```objc
/**
 向类添加新的方法与实现
 
 @param cls Class
 @param methodName 方法名
 @param methodIMPName 实现方法的方法名
 */
+ (void)addIntoClass:(Class)cls method:(SEL)methodName methodIMP:(SEL)methodIMPName {
    Method method = class_getInstanceMethod(cls, methodIMPName);
    IMP methodIMP = method_getImplementation(method);
    const char *methodType = method_getTypeEncoding(method);
    class_addMethod(cls, methodName, methodIMP, methodType);
}
```

- 动态添加方法-测试

```objc
    [RuntimeKit addIntoClass:[TestClass class] method:@selector(lookAtMe) methodIMP:@selector(lookAtMe)];
    NSArray *methodList = [RuntimeKit captureMethodList:[TestClass class]];
    NSLog(@"\n测试类的方法列表%@\n", methodList);
```

> 项目中并未对`lookAtMe`方法进行任何声明实现，该测试通过打印测试类方法列表中是否新增`lookAtMe`方法名来判断是否已添加成功。

### 2. 实现方法交换
实现原理：
1. 调用`class_getInstanceMethod(cls, methodName)`返回两个方法待交换。
2. 调用`method_exchangeImplementation(methodA, methodB)`完成交换操作即可。

- 交换方法-实现

```objc
/**
 在同一个类中进行方法的替换
 
 @param cls Class
 @param methodAName 待替换A的方法名
 @param methodBName 待替换B的方法名
 */
+ (void)swapMethod:(Class)cls methodA:(SEL)methodAName methodB:(SEL)methodBName {
    Method methodA = class_getInstanceMethod(cls, methodAName);
    Method methodB = class_getInstanceMethod(cls, methodBName);
    method_exchangeImplementations(methodA, methodB);
}
```

- 交换方法-测试

我们在`swapMethod:methodA:methodB`方法上进一步进行测试用例的封装。

```
#import "TestClass+SwapMethod.h"
#import "RuntimeKit.h"

@implementation TestClass (SwapMethod)

- (void)swapMethod {
    [RuntimeKit swapMethod:[self class]
                   methodA:@selector(exchangeMethodA)
                   methodB:@selector(replaceMethod)];
}

- (void)replaceMethod {
    //  此时调用的不是本身, 而是exchangeMethodA
    [self replaceMethod];
    NSLog(@"Now you can add something in methodA!");
}

@end

```

一般情况下，无任何其他操作。在`replaceMethod`中调用自身会势必造成死循环而导致程序崩溃。    
但如果交换方法调用成功，此时在`replaceMethod`中调用`replaceMethod`，**其实完成的是调用`exchangeMethodA`**。且`exchangeMethodA`是一个健全函数的实现。    
故在测试时，经调用二次封装的交换方法，唤起`replaceMethod`时便完成`exchangeMethodA`的调用。

## 三、属性关联
属性关联可以说是runtime最普通的打开方式。
1. 通过为属性声明一个静态名称。
2. 调用`void objc_setAssociatedObject(id object, const void *key, id value, objc_AssociationPolicy policy)`实现新增属性的set方法。
3. 调用`id objc_getAssociatedObject(id object, const void *key)`实现新增属性的get方法即可。

```objc
#import "TestClass+AssociatedObejct.h"
#import "RuntimeKit.h"

@interface TestClass ()

/** 添加的动态属性*/
@property (nonatomic, copy) NSString *dynamicProperty;

@end

static char dynamicKey;

@implementation TestClass (AssociatedObejct)

#pragma mark - 关联动态属性

/**
 setter方法
 
 @param dynamicProperty 需要关联的属性
 */
- (void)setDynamicProperty:(NSString *)dynamicProperty {
    objc_setAssociatedObject(self, &dynamicKey, dynamicProperty, OBJC_ASSOCIATION_COPY_NONATOMIC);
}

/**
 getter方法
 
 @return 关联的属性的值
 */
- (NSString *)dynamicProperty {
    return objc_getAssociatedObject(self, &dynamicKey);
}

@end
```

## 四、消息处理与消息转发
消息处理过程:    
- 当你调用一个类的方法时，先在本类中的方法缓存列表中进行查询，如果在缓存列表中找到了该方法的实现，就执行。
- 如果找不到就在本类中的方法列表中进行查找，在本类方法列表中查找到相应的方法实现后就进行调用。
- 如果依旧没找到，就去父类中进行查找；如果在父类中的方法列表中找到了相应方法的实现。
- 当在方法缓存列表，本类中的方法列表以及父类中的方法列表中都找不到相应的实现，到程序崩溃以前还会经历以下过程：

#### I. 消息处理
执行`+ (BOOL)resolveInstanceMethod:(SEL)sel`类方法，如果当一直寻找方法直到父类中都找不到方法实现时。    
该类方法返回值为NO，则表明不做任何处理，继续下一步；如果返回YES，就说明该方法中对找不到实现的方法进行了处理。我们在们就可以在此方法中为找不到实现的SEL动态添加一个方法实现，添加完毕后，就会执行我们添加的方法实现。下一次程序再找不到该类某个方法的实现时，就不会因为找不到而崩溃了。

- 消息处理

```objc
#pragma mark - 运行时方法拦截
- (void)unknowMethod:(NSString *)value {
    //  运行时找不到新添加的方法实现, 对其进行拦截, 并进行替换
    NSLog(@"Method is unknown, insert some words here: <%@>", value);
}

/**
 未找到SEL的IML实现时会执行的方法
 
 @param sel 方法选择器, 当前对象调用但是找不到IML的SEL
 @return 找到其他执行方法, 如自定义方法, 会返回YES, 否则返回NO
 */
+ (BOOL)resolveInstanceMethod:(SEL)sel {
    //  如果返回NO, 则继续执行forwardingTargetForSelector:方法
    [RuntimeKit addIntoClass:[self class] method:sel methodIMP:@selector(unknowMethod:)];
    return YES;
}
```

#### II. 消息转发
如果不对上述消息进行处理的话，也就是`+ (BOOL)resolveInstanceMethod:(SEL)sel`方法返回NO时。    
进入下一步消息转发：    
执行`- (id)forwardingTargetForSelector:(SEL)aSelector`方法。该方法会返回一个类对象，该类的对象有SEL对应的实现，当调用这个找不到方法时，就会转发到`ExtClass`中进行处理，完成消息转发。

- 消息转发

```objc
/**
 如果对象不存在SEL而传给其他对象时, 进入消息转发
 
 @param aSelector 当前类中不存在的SEL
 @return 消息转发该SEL所在的类
 */
- (id)forwardingTargetForSelector:(SEL)aSelector {
    //  如果resolveInstanceMethod:方法中不进行处理, 即返回NO, 此时进入消息转发
    return [ExtClass new];
}
```

如果该方法返回self或者nil，说明不对相应的方法进行转发，继续下一步。

#### III. 消息常规转发
如果不将消息转发给其他类的对象，则此时代表自己进行处理。即上述的方法中返回self或者nil。    
此时执行`- (NSMethodSignature *)methodSignatureForSelector:(SEL)aSelector`来获取方法的参数以及返回数据类型，即可以理解为该方法的签名。    
如果此时再次返回nil，那么消息转发结束。程序崩溃，报出找不到相应的方法实现的崩溃消息。

**下方方法执行的先决条件，是要在`+ (BOOL)resolveInstanceMethod:(SEL)sel`中返回NO。然后下方也是进行将方法转给ExtClass的实现**

- 消息不处理，即将崩溃

```objc
/**
 如果不实现消息转发, 则交由自身进行处理, 即消息转发时返回的是self, 则执行本签名方法
 
 @param aSelector 需要获取参数以及返回数据类型的方法
 @return 方法的签名, 如果返回nil, 则程序崩溃
 */
- (NSMethodSignature *)methodSignatureForSelector:(SEL)aSelector {
    //  查找父类的方法签名
    NSMethodSignature *signature = [super methodSignatureForSelector:aSelector];
    if (signature == nil) {
        signature = [NSMethodSignature signatureWithObjCTypes:"@@;"];
    }
    return signature;
}

/**
 消息接收对象无法正常响应消息时会被调用
 
 @param anInvocation 调用者, 即需要响应相应方法的对象
 */
- (void)forwardInvocation:(NSInvocation *)anInvocation {
    ExtClass *forwardClass = [ExtClass new];
    SEL sel = anInvocation.selector;
    /*
     如果消息转发的类响应了实现方法, 则对其调用, 否则,
     发现不了可以实现方法的调用者, 程序崩溃
     */
    if ([forwardClass respondsToSelector:sel]) {
        [anInvocation invokeWithTarget:forwardClass];
    } else {
        [self doesNotRecognizeSelector:sel];
    }
}
```

本项目Github链接地址[点击跳转](https://github.com/LibertyLeo/BlogReinforcements/tree/master/Runtime_Usage)

---
layout: post
title: 数据存储持久化相关知识
date: 2016-11-29 22:25:31 +08:00
tags: 数据存储
---

## 一、App应用结构

1. 每个应用都拥有独立且唯一的沙盒，即文件系统目录，除自身App外不允许对其进行访问。
2. 应用沙盒结构分析：

- 应用程序包: 包含了当前App所有的资源文件和可执行文件。
    - `Documents`：保存应用运行时生成的需要持久化的数据，iTunes同步设备时会备份该目录。例如游戏应用会将游戏存档保存在该目录。
    - `tmp`：保存应用运行时所需的临时数据，使用完毕后再将相应的文件从该目录删除。应用没有运行时，系统也可能会清除该目录下的文件。**iTunes同步设备时不会备份该目录**。
    - `Library/Caches`：保存应用运行时生成的需要持久化的数据，一般存储体积大、不需要备份的非重要数据。**iTunes同步设备时不会备份该目录**。
    - `Library/Preference`：保存应用的所有偏好设置，iOS的Settings（设置）应用会在该目录中查找应用的设置信息。**iTunes同步设备时会备份该目录**。

## 二、保存路径的生成

**根据保存数据类型不同，采取不同的保存路径，以下示例均统一放在`Documents`文件夹下**。

### 1. 沙盒路径拼接

通过拼接沙盒根目录与`Documents`文件夹路径而生成对应的保存路径（**不推荐**）

```objc
NSString *homeDirectory = NSHomeDirectory();
NSString *documentPath = [homeDirectory stringByAppendingPathComponent:@"Documents"];
```

### 2. 用户文件夹查找

通过用户文件夹进行往下查找，可理解为`NSUserDomainMask`参数的使用（**推荐**）

```objc
//  由于程序有且只有独立拥有一个Documents文件夹, 所以数组中仅仅只有一个元素, 无论是第一个对象还是最后一个对象均可
NSArray *applicationFilesArray = NSSearchPathForDirectoriesInDomains(NSDocumentDirectory, NSUserDomainMask, YES);
//  对要存储的的内容添加文件名, 第一种方法会自动添加'/', 故可以直接添加文件名, 而第二种需要手动添加
NSString *documentPath = [applicationFilesArray firstObject];
    
NSString *filePath1 = [documentPath stringByAppendingPathComponent:@"newInfo.plist"];
NSString *filePath2 = [documentPath stringByAppendingString:@"/newInfo.plist"];

//  可以发现这两个路径相同
NSLog(@"filePath1 Address:%@", filePath1);
NSLog(@"filePath2 Address:%@", filePath2);
```

### 3. 其他路径示例

- `tmp`：`NSTemporaryDirectory()`是`NSString`类型，可直接追加相应文件名称即保存路径。

```objc
[NSTemporaryDirectory() stringByAppendingPathComponent:@"newInfo.plist"]
```

- `Library/Caches`：`NSCachesDirectory`是`NSSearchPathDirectory`的枚举值，故对用户文件夹查找方式中的首个参数进行替换即可。
- `Library/Preference`：通过`NSUserDefaults`进行该目录下的存储设置。

## 三、应用数据的存储方式

一般来说，应用存储数据的方式不外乎五种方式：    
XML属性列表`plist`、偏好设置`Preference`、NSKeyedArchiver归档`NSCoding`、`SQLite3`、 `Core Data`。

### I、属性列表`plist`: 

对象支持的主要类型包含`NSString`、`NSDictionary`、`NSArray`、`NSData`、`NSNumber`等，其他基本数据类型可以通过`NSData`进行转换存储。

```objc
//  创建文件方法如下:
//  创建可变字典, 初始化设置值, 如果后期需要修改值, 只需要重新设置, 再保存一次即可
//  如果采用不可变字典, 首次写入依然可以生成, 但是后期不可更改信息
NSArray *applicationFilesArray = NSSearchPathForDirectoriesInDomains(NSDocumentDirectory, NSUserDomainMask, YES);
NSString *documentPath = [applicationFilesArray firstObject];
NSString *filePath = [documentPath stringByAppendingString:@"/newInfo.plist"];

NSMutableDictionary *saveInfoDic = [NSMutableDictionary dictionary];
[saveInfoDic setObject:@"Leo" forKey:@"name"];
[saveInfoDic setObject:@"Male" forKey:@"sex"];
[saveInfoDic setObject:@"25" forKey:@"age"];
[saveInfoDic writeToFile:filePath atomically:YES];
    
[saveInfoDic setObject:@"Female" forKey:@"sex"];
[saveInfoDic writeToFile:filePath atomically:YES];

//  读取文件方法如下:
//  从目标路径下获取字典, 获取单个属性键入键值即可
NSDictionary *readInfoDic = [NSDictionary dictionaryWithContentsOfFile:filePath];
NSLog(@"age:%@", readInfoDic[@"age"]);
NSLog(@"name:%@", readInfoDic[@"name"]);
```

### II、偏好设置`Preference`：

对象支持类型包含`NSString`、`NSDictionary`、`NSArray`、`NSData`、`NSNumber`等，其他基本数据类型可以通过`NSData`进行转换存储。

```objc
NSUserDefaults *defaults = [NSUserDefaults standardUserDefaults];
NSMutableDictionary *testDic = [NSMutableDictionary dictionary];
    
//  可以对键值对进行初始化
[defaults setObject:@"Leo" forKey:@"userName"];
[defaults setFloat:18.0f forKey:@"age"];
[defaults setBool:YES forKey:@"auto_login"];
    
//  该操作是对数据进行一个登记注册
[defaults registerDefaults:testDic];
    
//  取得之前设置的偏好设置
NSUserDefaults *getDefaults = [NSUserDefaults standardUserDefaults];
    
//  对应上式初始化, 再取得偏好设置文件后, 已经在册的键将被更新, 不存在的键将被创建
[getDefaults setFloat:25.0f forKey:@"age"];
[getDefaults setObject:@"Male" forKey:@"sex"];
    
//  将数据保存到本地磁盘当中进行保存, 此时为立即执行保存操作, 如果不执行该方法, 也许会造成数据还未保存完毕的结果
[getDefaults synchronize];

//  读取偏好设置的键值
NSLog(@"age:%@", [defaults objectForKey:@"age"]);
NSLog(@"sex:%@", [defaults objectForKey:@"sex"]);
```

### III、 NSKeyedArchiver归档`NSCoding`：

支持类型同`plist`，如果自定义类对象要采取该方法，则须遵守`NSCoding`协议。

- 协议中包含两个方法：
    - 方法一：每次**归档**均会调用该方法，一般在方法中指定如何归档对象中的每个实例变量。
        - 采取`encodeObject:forKey:`进行归档实例变量。
        - `- (void)encodeWithCoder:(NSCoder *)aCoder;`
    - 方法二：每次从文件中**恢复（解码）**对象时，调用该方法。一般在方法中指定如何解码对象中的每个实例变量。
        - 采取`decodeObjectForKey:`进行实例变量归档。
        - `- (nullable instancetype)initWithCoder:(NSCoder *)aDecoder;`

#### 1. 归档一个已存在的对象到`Documents`下

```objc
NSString *documentPath = NSSearchPathForDirectoriesInDomains(NSDocumentDirectory, NSUserDomainMask, YES).firstObject;
//  后缀名为archive (默认archive文件是加密的, 而设置为plist后缀的话可以进行明文阅读)
NSString *filePath = [documentPath stringByAppendingPathComponent:@"archiverExp.archive"];

//  1. 归档一个已有对象如数组对象到Documents下
NSArray *keyArchiveArray = @[@"testA", @"testB"];
[NSKeyedArchiver archiveRootObject:keyArchiveArray toFile:filePath];
    
//  解码恢复对象
NSArray *unarchiveArray = [NSKeyedUnarchiver unarchiveObjectWithFile:filePath];
//  打印解码信息
NSLog(@"unarchiveArray:%@", unarchiveArray);
```

#### 2. 归档自定义对象

> 首先创建自定义对象，以下几种归档均采用该对象为示例。

```objc
#import <Foundation/Foundation.h>

@interface Example : NSObject<NSCoding>

@property (nonatomic, copy) NSString *name;
@property (nonatomic, assign) NSInteger number;

@end
```

```objc
#import "Example.h"

@implementation Example

- (instancetype)initWithCoder:(NSCoder *)aDecoder {
    //  如果该类为子类, 父类遵守协议, 则需self = [super initWithCoder:aDecoder];
    self.name = [aDecoder decodeObjectForKey:@"name"];
    self.number = [aDecoder decodeIntegerForKey:@"number"];
    return self;
}

- (void)encodeWithCoder:(NSCoder *)aCoder {
    //  如果该类为子类, 父类遵守协议, 则需[super encodeWithCoder:aCoder];
    [aCoder encodeObject:self.name forKey:@"name"];
    [aCoder encodeInteger:self.number forKey:@"number"];
}

@end
```

> 进行归档操作。

```objc
NSString *documentPath = NSSearchPathForDirectoriesInDomains(NSDocumentDirectory, NSUserDomainMask, YES).firstObject;
//  后缀名为archive (默认archive文件是加密的, 而设置为plist后缀的话可以进行明文阅读)
NSString *filePath = [documentPath stringByAppendingPathComponent:@"archiverExp.archive"];

//  2. 归档自定义对象
Example *archiveExample = [[Example alloc] init];
archiveExample.name = @"eg";
archiveExample.number = 1;
[NSKeyedArchiver archiveRootObject:archiveExample toFile:filePath];
    
//  对象类型匹配, 解码恢复对象
Example *unarchiveExample = [NSKeyedUnarchiver unarchiveObjectWithFile:filePath];
NSLog(@"unarchiveExample name:%@", unarchiveExample.name);
NSLog(@"unarchiveExample number:%ld", unarchiveExample.number);
```

#### 3. 归档多个自定义对象

```objc
NSString *documentPath = NSSearchPathForDirectoriesInDomains(NSDocumentDirectory, NSUserDomainMask, YES).firstObject;
//  后缀名为archive (默认archive文件是加密的, 而设置为plist后缀的话可以进行明文阅读)
NSString *filePath = [documentPath stringByAppendingPathComponent:@"archiverExp.archive"];

Example *archiveExample = [[Example alloc] init];
archiveExample.name = @"eg";
archiveExample.number = 1;
Example *archiveExample2 = [[Example alloc] init];
archiveExample2.name = @"eg2";
archiveExample2.number = 2;

//  3. 归档多个自定义对象
//  建立可变数据区, 并连接到一个NSKeyedArchiver对象
NSMutableData *archiveData = [NSMutableData data];
NSKeyedArchiver *archive = [[NSKeyedArchiver alloc] initForWritingWithMutableData:archiveData];

//  开始对对象存档, 存档的数据都会储存在NSMutableData中
[archive encodeObject:archiveExample forKey:@"eg1"];
[archive encodeObject:archiveExample2 forKey:@"eg2"];
    
//  指定完对象存档, 调用该方法告诉系统存档完毕
[archive finishEncoding];
    
//  将存档好的数据写入文件
[archiveData writeToFile:filePath atomically:YES];
    
//  解码恢复对象
NSData  *unarchiveData = [NSData dataWithContentsOfFile:filePath];
    
//  根据数据, 解析出对应的NSKeyedUnarchive对象
NSKeyedUnarchiver *unarchive = [[NSKeyedUnarchiver alloc] initForReadingWithData:unarchiveData];
Example *unarchiveExample = [unarchive decodeObjectForKey:@"eg1"];
Example *unarchiveExample2 = [unarchive decodeObjectForKey:@"eg2"];
    
//  解码恢复完毕, 调用该方法告诉系统解码完毕, 此时unarchive不能再解码对象
[unarchive finishDecoding];

//  打印解码信息
NSLog(@"unarchiveExample name:%@", unarchiveExample.name);
NSLog(@"unarchiveExample2 name:%@", unarchiveExample2.name);

//  同理也可以将多个对象放到数组中进行归/解档, 在归/解档操作时会默认对数组中的对象自动进行encodeWithCoder:和initWithCoder的方法
```

#### 4. 利用归档来实现深复制

```objc
Example *archiveExample = [[Example alloc] init];
archiveExample.name = @"eg";
archiveExample.number = 1;

//  4、利用归档来实现深复制
//  临时存储数据
NSData *tempData = [NSKeyedArchiver archivedDataWithRootObject:archiveExample];
    
//  解析data, 生成对象
Example *waitForCopyExample = [NSKeyedUnarchiver unarchiveObjectWithData:tempData];
    
//  通过内存地址的打印, 可以发现已经完成了深拷贝操作
NSLog(@"example1 address:0x%x", archiveExample);
NSLog(@"example2 address:0x%x", waitForCopyExample);
```

### IV、SQLite3

最常用的开源数据库，内存开销小，操作都是基于C底层，效果好，除开SQL语句需要多使用熟练以外。

> 1. 常用的5种数据类型: `text`、 `integer`、 `float`、 `boolean`、`blob`等。
> 2. 在iOS开发中要进行其使用，需要添加库文件`libsqlite3.0.tbd`或`libsqlite3.tbd`均可，后者是一个**原始库**，前者是一个**指针指向最新的数据库**，如果更新则无需手动修改。
> 3. 添加完库文件后, 在使用处还需导入`<sqlite3.h>`，如果多处地方需使用，不妨写入预编译头文件进行导入。
> 4. 不排除现在部分开发者使用[fmdb][fmdb-source]等第三方数据库框架可以省去写SQL语句，但是把底层数据库知识储备完善，有百利而无一害。
> 5. 更多更清楚的了解[SQL][SQL-docs]。

#### 1. 创建并打开数据库

```objc
NSString *documentDirectory = NSSearchPathForDirectoriesInDomains(NSDocumentDirectory, NSUserDomainMask, YES).firstObject;
NSString *filePath = [documentDirectory stringByAppendingPathComponent:@"dataBase.sqlite"];
    
sqlite3 *sqliteDB = nil;
//  sqlite3_open()语句: 其将在指定路径下打开数据库, 如果不存在, 则新建数据库
//  通过openResult的返回结果可知道操作是否成功
int openResult = sqlite3_open([filePath UTF8String], &sqliteDB);
if (openResult != SQLITE_OK) {
    NSLog(@"打开数据库文件失败");
    return;
}
```

#### 2. 数据库创建表单

```objc
NSString *documentDirectory = NSSearchPathForDirectoriesInDomains(NSDocumentDirectory, NSUserDomainMask, YES).firstObject;
NSString *filePath = [documentDirectory stringByAppendingPathComponent:@"dataBase.sqlite"];
    
sqlite3 *sqliteDB = nil;
//  sqlite3_open()语句: 其将在指定路径下打开数据库, 如果不存在, 则新建数据库
//  通过openResult的返回结果可知道操作是否成功
int openResult = sqlite3_open([filePath UTF8String], &sqliteDB);
if (openResult != SQLITE_OK) {
    NSLog(@"打开数据库文件失败");
    return;
}

//  创建一个错误信息
char *errorMsg = nil;
    
//  如果在创表语句中去掉if not exists, 将会出现“工作表已存在”的报错语句
//  sqlite3_exec()可以执行任何SQL语句, 对于表创建以及表中数据的增删改都可以进行操作
char *createTableSQL = "create table if not exists t_person(id integer primary key autoincrement, name text, age integer);";
int executeResult = sqlite3_exec(sqliteDB, createTableSQL, NULL, NULL, &errorMsg);
if (executeResult != SQLITE_OK) {
    NSLog(@"数据库创建表格失败, 原因为:%s", errorMsg);
    return;
}

//  执行完操作, 关闭数据库
sqlite3_close(sqliteDB);
```

#### 3. 数据库值绑定

- `SQLite`中的插入语句，带占位符进行插入。
    - `sqlite3_prepare_v2()`用于判断语句是否有误，其参数依次代表：

    ```objc
    sqlite3 *db,            /* Database handle.                     数据库句柄*/
    const void *zSql,       /* SQL statement, UTF-16 encoded.       SQL语句*/
    int nByte,              /* Maximum length of zSql in bytes.     SQL语句的最大字符串长度、一般设-1即可, 从零终止符开始自动计算字符长度*/
    sqlite3_stmt **ppStmt,  /* OUT: Statement handle.               SQLy语句解析后的句柄*/
    const void **pzTail     /* OUT: Pointer to unused portion of zSql.   一个去观察SQL语句是否编译完毕的指针、指向下一个SQL语句开始的地址*/
    ```

    - `sqlite3_bind_text()`用于函数值的绑定，其参数根据不同的数据格式略有不同，基本以`stmt`指针、将绑定的值在表格中的位置，默认为1开始、绑定值、数据长度，设为-1自动计算、可选函数回调执行内存清理工作。

```objc
NSString *documentDirectory = NSSearchPathForDirectoriesInDomains(NSDocumentDirectory, NSUserDomainMask, YES).firstObject;
NSString *filePath = [documentDirectory stringByAppendingPathComponent:@"dataBase.sqlite"];
    
sqlite3 *sqliteDB = nil;
//  sqlite3_open()语句: 其将在指定路径下打开数据库, 如果不存在, 则新建数据库
//  通过openResult的返回结果可知道操作是否成功
int openResult = sqlite3_open([filePath UTF8String], &sqliteDB);
if (openResult != SQLITE_OK) {
    NSLog(@"打开数据库文件失败");
    return;
}

//  根据需求编写SQL插入语句
char *insertSQL = "insert into t_person(name, age) values (?, ?);";
sqlite3_stmt *insertStmt = nil;
if (sqlite3_prepare_v2(sqliteDB, insertSQL, -1, &insertStmt, NULL) == SQLITE_OK) {
    sqlite3_bind_text(insertStmt, 1, "Leo", -1, NULL);
    sqlite3_bind_int(insertStmt, 2, 27);
}
if (sqlite3_step(insertStmt) != SQLITE_DONE) {
    NSLog(@"数据插入错误");
}
    
//  关闭数据库句柄
sqlite3_finalize(insertStmt);
//  关闭数据库
sqlite3_close(sqliteDB);
```

#### 4. 数据查询
`sqlite3_step()`返回`SQLITE_ROW`查询到一条新记录。    
`sqlite3_column_*()`用于获取每个字段对应的值，第二参数为索引。

```objc
NSString *documentDirectory = NSSearchPathForDirectoriesInDomains(NSDocumentDirectory, NSUserDomainMask, YES).firstObject;
NSString *filePath = [documentDirectory stringByAppendingPathComponent:@"dataBase.sqlite"];
    
sqlite3 *sqliteDB = nil;
//  sqlite3_open()语句: 其将在指定路径下打开数据库, 如果不存在, 则新建数据库
//  通过openResult的返回结果可知道操作是否成功
int openResult = sqlite3_open([filePath UTF8String], &sqliteDB);
if (openResult != SQLITE_OK) {
    NSLog(@"打开数据库文件失败");
    return;
}

char *querySQL = "select id, name, age from t_person;";
sqlite3_stmt *queryStmt = nil;
if (sqlite3_prepare_v2(sqliteDB, querySQL, -1, &queryStmt, NULL) == SQLITE_OK) {
    while (sqlite3_step(queryStmt) == SQLITE_ROW) {
        int _id = sqlite3_column_int(queryStmt, 0);
        char *_name = (char *)sqlite3_column_text(queryStmt, 1);
        int _age = sqlite3_column_int(queryStmt, 2);
        NSString * name = [NSString stringWithUTF8String:_name];
        NSLog(@"id:%i, name:%@, age:%i", _id, name, _age);
    }
}
sqlite3_finalize(queryStmt);
sqlite3_close(sqliteDB);
```

### V、Core Data
时间仓促，暂未为Core Data进行整理。

[fmdb-source]: https://github.com/ccgus/fmdb "fmdb源代码"
[SQL-docs]: https://www.sqlite.org/c3ref/prepare.html "SQLite官网介绍"
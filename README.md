<h1 align='center'> ysql框架 </h1>

## 概述

更高效更简洁的sqlite数据库框架，借鉴了安卓Room框架的设计，将框架划分为Entity数据类、Dao数据访问类、Database数据库类三个层次。

框架直接建立在sqlite官方库的基础之上，无其他依赖，性能损失很小，隐藏（自动实现）了许多烦人的中间细节，只需要关心sql语句与python数据应用场景。

- 第三方库依赖：无
- 最新版本：1.3.3
- 项目地址：[ysql (gitee)](https://gitee.com/darlingxyz/ysql)
- 更新日志：[CHANGELOG.md](https://gitee.com/darlingxyz/ysql/blob/master/CHANGELOG.md)
- 使用文档：[ysql (github)](https://darlingxyz.github.io/ysql/#/)

## 结构示例

```python
@Entity
@dataclass
class Student:  # 定义一个数据类
    score: float
    name: str = 'LiHua', Constraint.not_null  # 用逗号分开默认值和约束
    student_id: int = Constraint.auto_primary_key 
    
@Dao(Student)
class DaoStudent:  # 定义一个数据访问类
    @Sql("select * from student where student_id=?;")
    def get_student(self, student_id):
        pass
    
@Path("test.db")
class Database(MetaDatabase):  # 定义顶层数据库类，对外使用的唯一接口
    dao1 = DaoStudent()
    dao2 = ...
    dao3 = ...
```

## 使用方式

**方式1**：通过pip安装

```
pip install ysql
```

**方式2**：通过gitee下载项目源码，将ysql文件夹放置到项目根目录中，如使用pycharm则应与venv目录同级。

## 特性介绍

**特性1**：整体操作分为表定义(Entity)、表操作(Dao)、库操作(Database)三种，由Entity到Dao再到Database，层层向上，不存在越级关系，模块解耦程度高。

**特性2**：Entity和Dao均采用装饰器方式实现，避免了大量的约定规则，仅需添加一行代码。

```python
@Entity
@dataclass  # 指定为原生的dataclass数据类
class Student:
    score: float
    name: str = 'LiHua', Constraint.not_null  # 同时设置默认值和约束，以逗号分开即可
    student_id: int = Constraint.auto_primary_key  # Constraint类提供多种字段约束
```

**特性3**：Dao中实现了@Insert和@Sql两个装饰器。Dao类仅需定义方法名称和参数，并在@Sql装饰器中编写原生sql语句即可，Dao装饰器将自动实现其余代码。

```python
@Dao(Student)  # 绑定对应的数据类
class DaoStudent:

    @Insert
    def insert(self, entity):
        pass  # 装饰器自动实现该方法

    @Sql("select name, score, picture from student where student_id=?;")
    def get_student(self, student_id):
        pass  # 装饰器自动实现该方法
```

**特性4**：@Sql装饰器自动转换查询结果为具名元组，无论编写的sql语句查询的是*还是指定字段，@Sql将解析并转换各个查询字段名，并生成具名元组来存储单条记录（全部查询结果使用列表存储）。避免了在应用层直接使用结果元组或者字典，改动数据结构时可以大大降低对应用层的影响。

## 设计分析

![框架设计](asset/design.svg)

**Entity**：基于python内置的dataclass类和本框架定义的Entity装饰器共同实现。由于采取装饰器和使用元编程、描述符向该类中增添功能，但装饰后的数据类仍然是独立完整的dataclass，与其他结构解耦，可以单独使用。

**Dao**：通过Dao装饰器注册该类对应的Entity类；隐藏了cursor对象的使用，用户只需关心数据库的connection；对类方法提供了Sql装饰器，仅需传递sql语句，无需具体实现该方法；提供了insert装饰器以及默认的insert方法，可以直接插入对硬的数据类对象至数据库中。

**Database**：通过将Dao类写为类中静态变量来集成全部的Dao类，是对外访问的唯一接口；具体通过继承MetaDatabase，以及Path装饰器传递数据库路径实现（由于装饰器是动态生成属性，在静态开发的时候不利于IDE的自动补全，因此采取了继承的策略）。

## 使用建议

![推荐编写结构](asset/advice.svg)

1.分文件编写各个Entity，可避免容易发生的引用问题。

2.将Entity与对应Dao写入同一文件中。

3.Dao类仅实现基础的操作方法，将复杂操作放至DataRepo中进行。

------


**作者**：大风起兮呼呼呼

**邮箱**：dfqxhhh@163.com

**时间**：2023-9-9
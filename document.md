# 使用文档
## Entity
### Entity(check_type: bool)

数据类dataclass的装饰器。
在定义dataclass时，同时将约束条件和默认值赋予类属性。
通过改造原生dataclass的init方法和getattribute方法，实现sql建表时约束条件的解析，以及正常使用被装饰dataclass时属性值的获取。
##### 参数
- check_type(bool): 传入True -> 实例化dataclass时检查属性值和类型注释是否一致，不一致将触发类型错误。否则不检查。
##### Example1:

```python
@Entity
@dataclass
class Student:  # 类名即表名，不区分大小写
    name: str  # 字段名以及字段类型
    score: float = 100
```

##### Example2:
```python
@Entity(check_type=True)  # 启用类型检查
@dataclass
class Student:
    name: str  # 字段名以及字段类型
    score: float = 100.0
    
    # Example1中的score属性虽然注释为float，而实际默认值为int，但仍然可以正常工作（由于python和sqlite都是宽松的类型约束）。
    # 若使用check_type，将进行严格的检查，类型不一致的情况（如Example1）会抛出异常
```
##### 注意
在实用角度上@dataclass应该合并到@Entity内部，这样定义数据类时只需要使用一个装饰器，并且不需要初级使用者关心什么是dataclass，
但经过测试发现，如果不显式的使用@dataclass装饰器，在实例化数据类时Pycharm将无法给出代码提示，这是不可忍受的。
并且经过测试发现只有Pycharm2020及之前的版本可以正确给出代码提示，高于2020版存在代码提示的bug，详见:
https://intellij-support.jetbrains.com/hc/en-us/community/posts/4421575751442-Code-Completion-doesn-t-work-for-class-functions-decorated-with-decorators-that-return-inner-functions


### Constraint

以类的形式对外开放的各种字段约束。

##### 属性：

- primary_key：主键
- auto_primary_key：自增主键
- not_null：非空
- unique：唯一

##### Example:
```python
@Entity
@dataclass
class Student:
    name: str  # 可以不使用约束
    score: float = 100  # 可以只赋予默认值
    address: str = 'HIT', Constraint.not_null  # 同时使用默认值和约束，需要以逗号分隔开
    student_id: int = Constraint.auto_primary_key, Constraint.not_null  # 同时使用多条约束，需要以逗号分隔开
```

##### 注意
建议导入Constraint类时使用别名，可以有效简化存在大量约束的使用场景。
**Example**:

```python
from ysql import Constraint as cs

@Entity
@dataclass
class Student:
    name: str
    score: float = 100
    address: str = 'HIT', cs.not_null
    student_id: int = cs.auto_primary_key, cs.not_null
```

#### Constraint.default(default_value)

默认值约束
##### 参数
- default_value: 该字段在sql中的默认值，与定义数据类时使用默认值作用类似。
##### Example:
```python
@Entity
@dataclass
class Student:
    name: str
    score1: float = 100
    score2: float = Constraint.default(100)  # score1和score2的作用类似
    student_id: int = Constraint.auto_primary_key
```

#### Constraint.check(check_condition: str)

条件约束
##### 参数
- check_condition: 具体条件
##### Example:
```python
@Entity
@dataclass
class Student:
    name: str
    score: float = Constraint.check('score > 60')  # 需要填写字段的字符形式名称
    student_id: int = Constraint.auto_primary_key
```

#### Constraint.foreign_key(entity, field, delete_link, update_link)

外键约束
##### 参数
- entity: 外键所在的数据类（父表）
- field: 外键对应的数据类属性
- delete_link: 级联删除方式
- update_link: 级联更新方式
##### Example:
```python
@Entity
@dataclass
class Student:  # 父表
    name: str = Constraint.not_null
    student_id: int = Constraint.auto_primary_key
    
@Entity
@dataclass
class Score:  # 子表
    score: float
    score_id: int = Constraint.auto_primary_key
    # 对student_id字段设置外键关联
    student_id: int = Constraint.foreign_key(entity=Student,
                                             field='student_id',
                                             delete_link=Constraint.cascade,
                                             update_link=Constraint.cascade)
```

#### Constraint.comment(comment: str)

字段注释
##### 参数
- comment: 具体注释。注意，在sqlite中只能通过DDL(Data Definition Language)查看。
##### Example:
```python
@Entity
    @dataclass
    class Student:
        name: str = Constraint.comment('学生姓名')
        student_id: int = Constraint.auto_primary_key, Constraint.comment('学生id')
```
## Dao

### Dao(entity)

对数据访问类使用的装饰器，用于绑定数据类和数据访问类

##### 参数

- entity: 该数据访问类对应的数据类

##### Example:
```python
@Entity
@dataclass
class Student:  # 定义一个数据类
    name: str
    score: float
    
@Dao(Student)  # 通过Dao装饰器绑定对应的数据类
class DaoStudent:  # 定义一个数据访问类
    ...
```

### Sql(static_sql_statement: str)

执行sql语句的装饰器，传入sql语句的同时会自动实现被装饰函数。

##### 参数

- static_sql_statement: 简单固定的纯字符串sql语句。

##### Example1:
```python
# 对于简单固定的sql语句可以直接传入Sql装饰器。
# 此段代码无法直接运行，需要首先连接数据库并插入记录才可运行。
@Entity
@dataclass
class Student:
    name: str
    score: float
    student_id: int = Constraint.auto_primary_key
    
@Dao(Student)
class DaoStudent:
    @Sql("select * from student where student_id=?;")
    def get_student(self, student_id):
        pass  # 此处并非以pass示意省略，而是Sql装饰器会自动实现该函数，因此实际使用时均只需要pass即可。
```
##### Example2:
```python
# 复杂sql语句可能需要外部传参，此时应该通过隐藏的参数sql_statement来传递复杂sql语句。
# 此段代码无法直接运行，需要首先连接数据库并插入记录才可运行。
@Dao(Student)
class DaoStudent:
    
    @staticmethod  # 借助静态方法装饰器，可以避免烦人且不需要的self参数的传递。也可以将该方法定义在其他任何位置。
    def generate_sql(*arg, **kwargs) -> str:
        print(*arg, **kwargs)
        return "select name from student where student_id=?;"
    
    @Sql("select * from student where student_id=?;")
    def get_student(self, student_id):
        pass
    
    @Sql  # 不再直接传递sql语句
    def get_student_name(self, student_id):
        pass
    
dao = DaoStudent()
result1 = dao.get_student(student_id=1)
result2 = dao.get_student_name(student_id=1,
                               sql_statement=Dao.generate_sql('some args'))  # 通过隐藏的sql_statement参数传递sql语句。

# 将以列表形式返回查询结果，其中每条记录的数据格式都是具名元组（与数据类类似）
# result1: [namedtuple('Record', 'name score student_id'), ...]  查询全部字段时，具名元组与定义的数据类完全一致。
# result2: [namedtuple('Record', 'name'), ...]  # 查询部分字段时，仅以查询的字段生成相应的具名元组。
```

##### 注意

设计思想始终是sql语句与数据传递分离。
1.对于简单sql情况，通过Sql装饰器传递sql语句，被装饰器函数传递数据。
2.对于复杂sql情况，为避免sql注入，请勿直接将数据与sql模板进行拼接，而是通过隐藏的参数sql_statement来传递复杂sql语句，被装饰器函数
仍然负责传递数据。


### Insert(*)

执行插入功能的装饰器，会自动生成插入sql语句，以及自动实现被装饰函数。

##### 返回

自动返回刚插入记录的自增主键（如果使用自增主键）。

##### Example:
```python
# 此段代码无法直接运行，需要首先连接数据库才可运行。
@Entity
@dataclass
class Student:
    name: str
    score: float
    student_id: int = Constraint.auto_primary_key
    
@Dao(Student)
class DaoStudent:
    @Insert  # 无需传递任何参数或者sql语句。
    def insert(self, entity):  # entity是必须的固定参数。
        pass  # 此处并非以pass示意省略，而是Insert装饰器会自动实现该函数，因此实际使用时均只需要pass即可。

dao = DaoStudent()
bob = Student(name='Bob', score=95.5)
dao.insert(entity=bob)  # 将整条记录以数据类的形式插入数据库，避免了同时使用多个参数的麻烦。
```

##### 注意

由于插入部分的结构完全固定，因此对数据访问类使用Dao装饰器后，会自动内置insert(entity)方法，无需额外定义（建议），可直接调用，但缺点是无法获取代码提示。


### insert(*, entity)

默认内置的insert方法，无需额外实现，可以直接调用。

##### 参数

- entity: 数据类的实例

##### 返回

自动返回刚插入记录的自增主键（如果使用自增主键）。

##### Example:
```python
# 此段代码无法直接运行，需要首先连接数据库才可运行。
@Entity
@dataclass
class Student:
    name: str
    score: float
    student_id: int = Constraint.auto_primary_key
    
@Dao(Student)
class DaoStudent:
    pass

dao = DaoStudent()
bob = Student(name='Bob', score=95.5)
student_id = dao.insert(entity=bob)  # 返回刚插入记录的自增主键
```
## MetaDatabase

### MetaDatabase

元数据库类，提供库级的操作
由于动态元编程无法使用代码提示，因此采取继承写法。

##### Example:
```python
@Entity
@dataclass
class Student:  # 定义一个数据类
    name: str
    student_id: int = Constraint.auto_primary_key
    
@Dao(Student)
class DaoStudent:  # 定义一个数据访问类
    @Sql("select * from student where student_id=?;")
    def get_student(self, student_id):
        pass
    
class Database(MetaDatabase):  # 定义一个数据库类，继承元数据库类
    dao1 = DaoStudent()  # 将各个数据访问类实例化为类中静态变量，集中管理，统一对外。
    dao2 = ...
    dao3 = ...
    
db = Database(db_path='test.db')  # 实例化数据库类，并传入数据库路径
db.connect()  # 连接数据库
db.create_tables()  # 创建数据表
db.commit()  # 提交更改
```

#### MetaDatabase.connect(*, use_multithreading)

连接数据库


#### MetaDatabase.disconnect(*)

断开数据库连接


#### MetaDatabase.create_tables(*)

当表不存在时，才会创建数据表。因此可以反复调用该方法，而不会产生错误。


#### MetaDatabase.commit(*)

提交事务


#### MetaDatabase.rollback(*)

回滚事务


#### MetaDatabase.execute(*, statement: str)

给外部提供的直接执行sql的接口，避免了再调用内部的connection

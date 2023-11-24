# 可运行的使用示例
## 快速示例

```python
# -*- coding: utf-8 -*-
import os
from dataclasses import dataclass
from ysql import Entity, Constraint, Dao, Sql, MetaDatabase


# 结构说明：
# Entity定义数据类，对应数据表
# Dao定义数据访问类，提供表级控制
# Database定义数据库类，集成Dao统一对外，提供库级控制


# 数据类的定义形式：
@Entity  # 表明这是数据（表）类
@dataclass
class Student:  # 类名即表名，不区分大小写
    score: float
    name: str = '好家伙', Constraint.not_null  # 同时设置默认值和约束，以逗号分开即可
    student_id: int = Constraint.auto_primary_key  # Constraint类提供多种字段约束


# 数据访问类的定义形式：
@Dao(Student)  # 绑定对应的数据类
class DaoStudent:
    # 内置insert(entity)方法，可直接调用，参数entity应为对应数据类的实例

    @Sql("select * from student where student_id=?;")  # 通过Sql装饰器传递sql语句
    def get_student(self, student_id):
        pass  # 装饰器自动实现该方法


# 数据库类的定义形式：
class Database(MetaDatabase):  # 继承MetaDatabase类，以表明这是数据库类
    # 需将全部Dao类分别定义为类中静态变量，以便集成Dao到一个数据类中对外访问
    dao_student = DaoStudent()


if __name__ == '__main__':
    db_path = "test_simple.db"
    if os.path.exists(db_path):
        os.remove(db_path)  # 删除已存在的数据库文件，便于测试和时自动重置

    db = Database(db_path=db_path)  # 创建数据库类的实例
    db.connect()  # 连接数据库
    db.create_tables()  # 创建表

    student = Student(name="李华", score=95.6)
    db.dao_student.insert(student)  # 插入一条记录
    db.commit()  # 提交改动

    result = db.dao_student.get_student(student_id=1)  # 获取一条记录
    print(f"查询结果：{result}")  # 查询结果以列表嵌套具名元组的形式返回

    db.disconnect()  # 关闭数据库连接

```

## 较完整示例

```py
# -*- coding: utf-8 -*-
import os
from dataclasses import dataclass
from ysql import Entity, Constraint, Dao, Sql, MetaDatabase


# 结构说明：
# Entity定义数据类，对应数据表
# Dao定义数据访问类，提供表级控制
# Database定义数据库类，集成Dao统一对外，提供库级控制


# 数据表类的定义形式
@Entity(check_type=True)  # 开启实例化时的严格类型检查
@dataclass  # 显式的使用dataclass装饰器，可以在IDE中获取代码补全
class Student:  # 类名即表名，不区分大小写
    name: str = Constraint.not_null, Constraint.unique  # 用逗号分个多个约束，Constraint类提供多种字段约束
    weight: float = 50.0, Constraint.not_null  # 同样以逗号分隔，默认值和约束
    student_id: int = Constraint.auto_primary_key


@Entity
@dataclass
class Score:
    score: float
    student_id: int = Constraint.auto_primary_key, \
                      Constraint.foreign_key(entity=Student,
                                             field='student_id',
                                             delete_link=Constraint.cascade,
                                             update_link=Constraint.cascade)  # 设置外键的具体约束


# Dao类的定义形式
@Dao(entity=Student)  # 绑定对应的数据类
class DaoStudent:

    # 内置insert(entity)方法的返回值为刚插入记录的row id
    # 缺点是在IDE的自动补全无法提示该方法的存在
    # @Insert  # 如需重新定义插入方法，可使用sql装饰器并传递相应的sql语句，或者直接使用Insert装饰器，也可自动实现。
    # def insert(self, entity):
    #   pass
    # 如果使用了@Insert（包括内置的insert方法），被装饰方法会自动返回插入记录的主键

    @Sql("select * from student where student_id=? and name=?;")  # 对方法使用Sql装饰器，并传入sql语句即可
    def get_student(self, student_id, name):  # 参数名可任意，但顺序需要与sql语句中?的传值顺序一致
        pass  # 方法本身无需实现
        # 使用select的时候返回结果为list[named_tuple]，其中每条记录会自动解析为具名元组。


@Dao(entity=Score)  # 绑定对应的数据类
class DaoScore:

    # 需要entity的表名时，可以使用__代替符，但__只允许出现一次。
    @Sql(f"select * from __ where student_id=?;")
    def get_score(self, student_id):
        pass  # 方法本身无需实现


# 数据库类的定义形式
class Database(MetaDatabase):  # 继承MetaDatabase类，以表明这是数据库类
    # 需将全部Dao类分别定义为类中静态变量，以便集成Dao到一个数据类中对外访问
    dao_student = DaoStudent()
    dao_score = DaoScore()


if __name__ == '__main__':
    db_path = "test_complex.db"
    if os.path.exists(db_path):
        os.remove(db_path)  # 删除已存在的数据库文件，便于测试和时自动重置

    db = Database(db_path)
    db.connect()
    db.create_tables()

    db.dao_student.insert(Student(name="李华"))  # 插入一条student记录
    db.dao_score.insert(Score(score=59.5, student_id=1))  # 插入一条score记录
    db.commit()  # 提交改动

    result1 = db.dao_student.get_student(student_id=1, name='李华')  # 获取一条student记录
    result2 = db.dao_score.get_score(student_id=1)

    # 查询结果以列表形式返回
    print(f"查询结果1：{result1}\n查询结果2：{result2}")

    db.disconnect()  # 关闭数据库连接
```
### 为什么选用 Flask

Flask 主要有两大优点。第一个优点，它是轻量级 Python 开发框架。意思就是 Flask 核心就是几个文件，不用大量的配置文件，很快就能搭建起来一个小型的 Web 应用并快速启动运行，很适合搞微服务

API 开发在 Flask 中不复杂，几行代码就可以实现一个简单的功能。

```python
from flask import Flask, jsonify
app = Flask(__name__)
def get_weather(city):
  # 根据城市名称查询天气
  # 返回查询结果
  return {'city': city, 'weather': 'sunny'}
@app.route('/weather/<city>')
def weather(city):
  # 调用get_weather函数获取天气情况
  result = get_weather(city)
  # 将查询结果以json格式返回
  return jsonify(result)
if __name__ == '__main__':
  app.run()
```

Django 有着比较严密的架构设计，安全性也比较高，可以保护网站抵御像 SQL 注入这样的恶意操作。

### 框架搭建

* 点击 “Create New Project” 按钮，然后输入项目的名称和存放路径。
* 点击 New environment using，选择 “**Virtualenv**”，然后在 Base interpreter 中选择 Python 版本。**有软件 anaconda 自带的 Python 解释器**，可以使用下载过的 Python 解释器即可。若系统未自动检测到，还需要点击右边省略号图标，找到并选中 python.exe 文件路径。

使用 pip 命令来安装，在下方的终端——Terminal 输入命令行即可

```shell
pip install flask
```

<img src="../images/image-20250314145700273.png" alt="image-20250314145700273" style="zoom:50%;" />

```python
from flask import Flask    # 导入Flask模块
app = Flask(__name__)      # 创建Flask实例，并指定模块名

@app.route('/')                 # 定义路由，即当访问 根目录 时返回下面的函数结果
def hello_world():
    return 'Hello, World!'   # 返回字符串Hello, World!
  
if __name__ == '__main__':
    app.run()                       # 运行Flask应用程序
```

最后的 if name == ‘main’ 方法中 name 表示当前模块名，如果是直接运行的 Python 脚本，则为 main

### 视图函数

Flask 中的视图函数就是用于处理 HTTP 请求并返回响应的 Python 函数。视图函数通常接受 HTTP 方法（如 GET、POST、PUT、DELETE 等）和 URL 路径作为参数，并根据请求的不同和条件来执行不同的操作。

**render_template 函数**

* 视图函数负责处理请求
* 生成需要传递给模板的数据
* 然后使用  render_template 函数将数据传递给模板来渲染响应内容。

在 render_template 函数内可以接受两个参数。**第一个参数是 template_name，代表要渲染的模板文件名称。**

可以在用于生成 HTML 响应的 Jinja2 模板中，使用  {{ }} 语法来引用  context 中的变量。

```python
from flask import Flask, render_template
app = Flask(__name__)

@app.route('/')
def index():
  data={
    'name':'列表',
    'len':'8',
    'list':{1,2,3,4,5,6,7,8}
  }
  return render_template('index.html'，data=data)

if __name__ == '__main__':
  app.run()
```

```html
<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <title>Title</title>
</head>
<body>
名称：{{data['name']}}
<br>
长度：{{data['len']}}
<br>
list：{{data['list']}}
</body>
</html>
```

### ORM

使用 ORM 建表的第一步就是创建一个 UserInfo 类，这里的类名一般就是我们的表名。如果想指定表名，我们可以使用 ___tablename__ 这个属性来进行赋值

```python
class UserInfo(db.Model):
    """用户信息表"""
    __tablename__ = "user_info"  # 表名
```

在创建类时传入了一个参数 db.Model。**db 是我们后面使用的工具 SQLAIchemy 的一个实例化对象**。我们这里用到的 db.Model 就是它的一个基类，用来定义数据库中的模型，并与数据库进行交互。

当一个类继承 db.Model 后，就可以使用 ORM（对象关系映射）的方式来操作数据库。同时，通过继承 db.Model

### 设计数据库表

在设计各种业务模块使用的数据库表之前，我们通常需要先定义一个模型基类，其中包含每个数据库表都需要使用的字段

```python
class BaseModels:
    """模型基类"""
    # 创建时间
    create_time = db.Column(db.DateTime, default=datetime.now)
    # 记录你的更新时间
    update_time = db.Column(db.DateTime, default=datetime.now, onupdate=datetime.now)
    # 记录存活状态
    status = db.Column(db.SmallInteger, default=1)  
    def add(self, obj):
        db.session.add(obj)
        return session_commit()
    def update(self):
        return session_commit()
    def delete(self):
        self.status = 0
        return session_commit()
```

```python
class UserInfo(BaseModels, db.Model):
    """用户信息表"""
    __tablename__ = "user_info"
    id = db.Column(db.Integer, primary_key=True, autoincrement=True)  # 用户id
    nickname = db.Column(db.String(64), nullable=False)  # 用户昵称
    mobile = db.Column(db.String(16))  # 手机号
    avatar_url = db.Column(db.String(256))  # 用户头像路径
    signature = db.Column(db.String(256))  # 签名
    sex = db.Column(db.Enum('0', '1', '2'), default='0')  # 1男  2 女 0 暂不填写
    birth_date = db.Column(db.DateTime)  # 出生日期
    role_id = db.Column(db.Integer)  # 角色id
    last_message_read_time = db.Column(db.DateTime)
```

### 数据库连接

```sh
pip install flask-sqlalchemy
pip install pymysql
```

在 practice 文件下新建 lb_04.py 文件，它主要的作用就是在 Flask 应用中连接到 MySQL 数据库服务器。

```python
import pymysql
from flask import Flask
app = Flask(__name__)
@app.route("/")
def hello():
    connection = pymysql.connect(
        host='127.0.0.1',  # 数据库IP地址
        port=3306,  # 端口
        user='root',  # 数据库用户名
        password='flask_project',  # 数据库密码
        database='flask_databases'  # 数据库名称
    )
    return "恭喜，MySQL数据库已经连接上"

if __name__ == "__main__":
    app.run()
```

数据库的配置我们要对应放在 config 文件下。所以我们第一步要做的，就是直接在 config 文件下新建 config.py。

```python
connection = pymysql.connect(
        host='127.0.0.1',  # 数据库IP地址
        port=3306,  # 端口
        user='root',  # 数据库用户名
        password='flask_project',  # 数据库密码
        database='my_project'  # 数据库名称
    )
```

```python
class Config:
    SQL_HOST = '127.0.0.1'
    SQL_USERNAME = 'root'
    SQL_PASSWORD = 'flask_project'
    SQL_PORT = 3306
    SQL_DB = 'my_project'
    JSON_AS_ASCII = False
    # 数据库的配置
    SQLALCHEMY_DATABASE_URI = f"mysql+pymysql://{SQL_USERNAME}:{SQL_PASSWORD}@{SQL_HOST}:{SQL_PORT}/{SQL_DB}"
    SQLALCHEMY_TRACK_MODIFICATIONS = False
```

项目包括开发、测试、生产这三个阶段，所以我们为了配置的方便，可以事先定义好这三个类。

```python
# 开发环境的config
class DevConfig(Config):
    pass
# 生产环境的config
class ProConfig(Config):
    DEBUG = False
    SQLALCHEMY_DATABASE_URI = "mysql+pymysql://root:my_project@127.0.0.1:3306/aaa"
# 测试环境的config
class TestConfig(Config):
    pass
config_dict = {
    'dev': DevConfig,
    'pro': ProConfig,
    'test': TestConfig,
}
```

### 通过 SQLAlchemy 实现 ORM 应用

```
pip install flask-sqlalchemy
```

```python
from flask import Flask
from flask_sqlalchemy import SQLAlchemy
app = Flask(__name__)
db = SQLAlchemy(app)
```

在项目当中，我们一般把 db 和 app 的实例化，放在 api 文件夹下的 __init__.py 文件中。

```python
from flask import Flask
from flask_sqlalchemy import SQLAlchemy
from config.config import config_dict
db = SQLAlchemy()
def create_app(config_name):
    app = Flask(__name__)
    config = config_dict.get(config_name)
    app.config.from_object(config)
    db.init_app(app)
    return app
```

#### 建立数据库表

```python
from datetime import datetime
from api import db
class BaseModels:
    """模型基类"""
    create_time = db.Column(db.DateTime, default=datetime.now)  # 创建时间
    update_time = db.Column(db.DateTime, default=datetime.now, onupdate=datetime.now)  # 记录你的更新时间
    status = db.Column(db.SmallInteger, default=1)  # 记录存活状态
```

```python
from api import db
from api.models.base import BaseModels
class UserInfo(BaseModels, db.Model):
    """用户信息表"""
    __tablename__ = "user_info"
    id = db.Column(db.Integer, primary_key=True, autoincrement=True)  # 用户id
    nickname = db.Column(db.String(64), nullable=False)  # 用户昵称
    mobile = db.Column(db.String(16))  # 手机号
    avatar_url = db.Column(db.String(256))  # 用户头像路径
    signature = db.Column(db.String(256))  # 签名
    sex = db.Column(db.Enum('0', '1', '2'), default='0')  # 1男  2 女 0 暂不填写
    birth_date = db.Column(db.DateTime)  # 出生日期
    role_id = db.Column(db.Integer)  # 角色id
    last_message_read_time = db.Column(db.DateTime)
    def to_dict(self):
        return {
            'id': self.id,
            'nickname': self.nickname,
            'mobile': self.mobile,
            'avatar_url': self.avatar_url,
            'sex': self.sex,
        }
```

我们只要在 app.py 文件下做好相关函数的导入和初始化工作，就可以把 app.py 作为 Flask 的命令行接口，通过它运行不同的 Flask 命令，高效完成数据库迁移、创建超级用户、导出数据等操作。

```python
from api import db, create_app
from flask_script import Manager
from flask_migrate import Migrate
app = create_app('dev')
# 管理器对象可以用来提供各种管理命令，例如启动应用程序、创建数据库表、导入数据等等
manager = Manager(app)
Migrate(app, db)
def create_db():
    db.create_all()
if __name__ == '__main__':
    manager.run()
```






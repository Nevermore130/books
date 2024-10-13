Flask 中的视图函数就是用于处理 HTTP 请求并返回响应的 Python 函数

和视图函数紧密相关的函数和对象包括 render_template 函数和 request 对象

#### render_template 函数 ####

视图函数负责处理请求并生成需要传递给模板的数据，然后使用  render_template 函数将数据传递给模板来渲染响应内容。

第一个参数是 template_name，代表要渲染的模板文件名称。

第二个参数是 context，可以是一个字典，包含要在模板中使用的变量和数据。context 参数还可以是 Flask 视图函数的返回值，它可以被视图函数中的变量引用。

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

Int 转化器的案例，带你了解一下系统自带转化器的应用方法

```python
from flask import Flask
app = Flask(__name__)
@app.route("/index/<int:id>",)
def index(id):
  if id == 1:
    return 'first'
  elif id == 2:
    return 'second'
  elif id == 3:
    return 'thrid'
  else:
    return 'hello world!'
if __name__=='__main__':
  app.run()
```

#### 自定义转化器 ####

自定义转换器可以将 URL 中的字符串转换成指定类型的变量，并将其传递给视图函数

```python
from werkzeug.routing import BaseConverter #导入转换器的基类，用于继承方法
# 自定义转换器类
class Flask_DIY_Converter(BaseConverter):
  def to_python(self, value):
    # 将字符串转换成指定类型的变量
    return int(value)  # 这里指定转换成int类型
  def to_url(self, value):
    # 将变量转换成字符串
    return str(value)
```

* BaseConverter 是一个基础转换器，主要用于处理 URL 中的字符串
* to_python(value) 方法是将 URL 中的字符串转换成 int 类型的变量
* to_url(value) 方法将变量转换成字符串，用于生成 URL。
* 在 Flask 中使用自定义转换器之前，还需要将其注册到应用中

```python
from flask import Flask
app = Flask(__name__)
app.url_map.converters['my_cv'] = Flask_DIY_Converter
#将自定义的转换器my_cv，
#添加到Flask类下url_map属性（一个Map类的实例）包含的转换器字典属性中
# converters就是这个转换器字典
# 作用是将URL路径与视图函数进行匹配，并将请求分发到对应的视图函数中处理
```

* url_map 是 Flask 中的 URL 映射表，它的作用同样是将 URL 路径与视图函数进行匹配，并将请求分发到对应的视图函数中处理。
* converters 是一个字典，它的键是自定义参数转换器的名称，值是实现参数转换逻辑的函数或类。

```python
@app.route('/user/<my_cv:user_id>')
def show_(user_id):
  # user_id 是一个整数类型的变量
  return 'User %d' % user_id
```



### 异常捕获 ###

以正则路由匹配手机号格式的代码为例

```python
from flask import Flask,abort
import re
from werkzeug.routing import BaseConverter
app = Flask(__name__)
class RegexConverter(BaseConverter):
  def __init__(self,url_map,regex):
    super(RegexConverter,self).__init__(url_map)
    self.regex = regex
  def to_python(self, value):
    return value  
app.url_map.converters['my_cv'] = RegexConverter
@app.route("/<my_cv(r'1[3456789]\d{9}'):user_id>")
def index(user_id):
  if not re.match('1[3456789]\d{9}', user_id):
    # abort的用法类似于python中的raise，在网页中主动抛出错误
    abort(404)
    return None
  else:
    return '格式正确'
if __name__ == '__main__':
  app.run()
```

#### errorhandler 装饰器 ####

```python
@app.errorhandler(404)
def page_not_found(error):
  return "404错误，您访问的页面不存在，原因是您所输入的URL格式有误！"
# 比如说404，当abort抛出这个404异常之后，就会访问以下的这个代码。
```

使用 ```@app.errorhandler ```装饰器把它注册为一个全局的错误处理程序，当 Flask 应用程序发生 404 错误时，就会自动调用该函数。

### 数据库 ###

```python
class UserInfo(db.Model):
    """用户信息表"""
    __tablename__ = "user_info"  # 表名
    # id
    id = db.Column(db.Integer, primary_key=True, autoincrement=True) 
    nickname = db.Column(db.String(64), nullable=False)  # 用户昵称
    mobile = db.Column(db.String(16))  # 手机号
    sex = db.Column(db.Enum('0', '1', '2'), default='0') 
    # 1=男，2=女，0=暂不填写，默认值为
```

#### ORM ####

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

安装好需要的 SQLAIchemy 库

```sh
pip install flask-sqlalchemy
pip install pymysql
```

新建 lb_04.py 文件，它主要的作用就是在 Flask 应用中连接到 MySQL 数据库服务器。

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












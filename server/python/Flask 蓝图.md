所有的页面处理逻辑都是放在同一个文件中，随着业务代码的增加，将所有代码都放在单个程序文件中是非常不合适的。不仅会让阅读代码变得困难，而且会给后期维护带来麻烦



需要对程序进行模块化的处理，蓝图 (Blueprint) 是 Flask 程序的模块化处理机制，它是一个存储视图方法的集合，**Flask 程序通过 Blueprint 来组织 URL 以及处理请求。**

假设网站包含有如下 4 个页面：

| 页面            | 功能           | 处理函数      |
| :-------------- | :------------- | :------------ |
| /news/society/  | 社会新闻版块   | society_news  |
| /news/tech/     | IT 新闻版块    | tech_news     |
| /products/car/  | 汽车产品版块   | car_products  |
| /products/baby/ | 婴幼儿产品版块 | baby_products |

假设访问的页面路径是 /products/car，Flask 框架在蓝图 news 和蓝图 products 中查找匹配该页面路径的路由，发现在蓝图 products 中，存在和路径 /products/car 相应的处理函数 car_products，最后将请求转发给函数 car_products 处理。

```python
from flask import Flask
import news
import products

app = Flask(__name__)

app.register_blueprint(news.blueprint)
app.register_blueprint(products.blueprint)

app.run(debug = True)

```



```python
from flask import Blueprint

blueprint = Blueprint('news', __name__, url_prefix='/news')

@blueprint.route("/society/")
def society_news():
    return "社会新闻版块"

@blueprint.route("/tech/")
def tech_news():
    return "IT 新闻板块"

```

```python
from flask import Blueprint

blueprint = Blueprint('products', __name__, url_prefix='/products')

@blueprint.route("/car")
def car_products():
    return "汽车产品版块"

@blueprint.route("/baby")
def baby_products():
    return "婴儿产品版块"    

```

如果项目中的 templates 文件夹中没有相应的模板文件，则使用定义蓝图的时候指定的 templates 文件夹下的模板文件

如果项目中的 static 文件夹中没有相应的静态文件，则使用定义蓝图的时候指定的 static 文件夹下的静态文件。






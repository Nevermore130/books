应用阶段主要负责提供各种渲染数据和Shader程序；几何阶段又分为顶点着色器阶段、几何着色器、裁剪和屏幕映射四个小阶段；光栅化阶段分为连接三角形、遍历三角形、片元着色、片元操作几个小阶段。

```java
// 定义顶点着色器
String vertexShaderSource = "#version 300 es\n"
    + "layout (location = 0) in vec3 aPos;\n" // 顶点坐标
    + "layout (location = 1) in vec4 aColor;\n" // 顶点颜色
    + "uniform mat4 model;\n" // 模型变换矩阵
    + "uniform mat4 view;\n" // 视图变换矩阵
    + "uniform mat4 projection;\n" // 投影变换矩阵
    + "out vec4 vColor;\n" // 输出颜色
    + "void main()\n"
    + "{\n"
    + "   gl_Position = projection * view * model * vec4(aPos, 1.0);\n" // 计算裁剪空间坐标
    + "   vColor = aColor;\n" // 传递颜色
    + "}\0";
```

在Shader程序中，我们定义了aPos、model、view、projection等变量用来**接收用户传入的顶点数据、模型变换矩阵、视图变换矩阵、投影变换矩阵**。最后，当用户调用glDrawArrays后，触发可编程管线运行，可编程管线在顶点着色器阶段运行顶点Shader，这样渲染过程就运转起来了。

* 上述代码中，第1行指定了使用的OpenGL版本，300表示使用的是OpenGL3.0，es表示使用OpenGL的嵌入式版本，即OpenGLES
* 第2-3行中的in表示这两个变量都是输入变量，而前面的Layout(location=…)指明了变量的位置，通过这种方式，外面向该变量传数据时，就不再需要通过glGetAttribLocation()函数获取其位置了，可以直接使用glVertexAttribPointerb()函数对其赋值
* 第7行中的out指明vColor是一个输出变量，该变量是片元Shader的输入变量，因此在片元Shader中一定有一个与该变量名一样，但前面有in修饰的变量；最后面的main()函数逻辑非常简单，计算每个顶点的位置，并将顶点位置输出给gl_Position，同时将传入的颜色值输出到vColor。

#### 片元Shader ####

片元Shader的作用是对图形中每个像素的颜色进行填充

```java
String fragmentShaderSource = "#version 300 es\n"
    + "in vec4 vColor;\n" // 输入颜色
    + "out vec4 FragColor;\n" // 输出颜色
    + "void main()\n"
    + "{\n"
    + "   FragColor = vColor;\n" // 直接输出颜色
    + "}\n";
```

* main函数逻辑非常简单，就是将顶点Shader传过来的颜色直接输出给FragColor，也就是输出到屏幕上。


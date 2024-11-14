### 图形渲染管线 ###

* 固定渲染管线指的是，图像在渲染时必须按照固定的顺序一步步执行，用户在渲染开始前提供需要渲染的数据和参数，然后等待渲染结果
* 可编程渲染管线指的是，在图像渲染时仍然按照固定的顺序一步步执行，但在某些阶段，如顶点着色器阶段、片元处理阶段，可以通过编写Shader程序进行控制。在Android系统下，OpenGLES1.0采用的是固定管线，而从OpenGLES2.0之后则变成了可编程管线。

图形渲染管线从大的方面分为：应用阶段、几何阶段和光栅化阶段。其中，**应用阶段是在CPU中运行的，而几何阶段和光栅化阶段是在GPU中运行的。**

**应用阶段主要负责提供各种渲染数据和Shader程序**；**几何阶段又分为顶点着色器阶段、几何着色器**、裁剪和屏幕映射四个小阶段；光栅化阶段分为连接三角形、遍历三角形、片元着色、片元操作几个小阶段。

在使用固定管线渲染三角形时，我们只需要调用指定的API将顶点坐标、模型变换矩阵、视图变换矩阵传给GPU，然后再调用glDrawArrays方法启动绘制即可将三角形绘制出来。

```java
...
// 设置顶点数据
glVertexPointer(3, GL_FLOAT, 0, vertices); // 顶点坐标
glColorPointer(4, GL_FLOAT, 0, colors); // 顶点颜色
glEnableClientState(GL_VERTEX_ARRAY); // 启用顶点数组
glEnableClientState(GL_COLOR_ARRAY); // 启用颜色数组

// 设置模型变换矩阵
glMatrixMode(GL_MODELVIEW); // 选择模型视图矩阵

// 设置视图变换矩阵
gluLookAt(0.0f, 0.0f, 0.0f, // 相机位置
          0.0f, 0.0f, -1.0f, // 相机朝向
          0.0f, 1.0f, 0.0f); // 相机上方向

...

// 绘制三角形
glDrawArrays(GL_TRIANGLES, 0, 3); // 绘制三个顶点
```

使用可编程渲染管线则与固定渲染管线不同，我们首先要编写Shader程序，然后将数据传给Shader程序，最后由Shader程序控制顶点着色器最终将三角形绘制出来。代码如下：

```java
...
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

// 定义片元着色器
String fragmentShaderSource = ...

// 创建顶点着色器对象
int vertexShader;
vertexShader = GL20.glCreateShader(GL20.GL_VERTEX_SHADER);
// 将着色器源码附加到着色器对象上
GL20.glShaderSource(vertexShader, vertexShaderSource);
// 编译着色器
GL20.glCompileShader(vertexShader);

// 创建片元着色器对象
...

// 创建着色器程序对象
int shaderProgram;
shaderProgram = GL20.glCreateProgram();
// 将着色器对象链接到程序对象上
GL20.glAttachShader(shaderProgram, vertexShader);
GL20.glAttachShader(shaderProgram, fragmentShader);
GL20.glLinkProgram(shaderProgram);

...

// 设置顶点数据
float[] vertices = {
  // 顶点坐标          // 顶点颜色
  0.0f,  0.5f, 0.0f,  1.0f, 0.0f, 0.0f, 1.0f, // 顶点0
  -0.5f, -0.5f, 0.0f,  0.0f, 1.0f, 0.0f, 1.0f, // 顶点1
  0.5f, -0.5f, 0.0f,  0.0f, 0.0f, 1.0f, 1.0f  // 顶点2
};

...

// 设置顶点属性指针
GL20.glVertexAttribPointer(0, ...); // 顶点坐标
GL20.glEnableVertexAttribArray(0); // 启用顶点属性

// 设置模型变换矩阵
Matrix4f model = new Matrix4f(); // 初始化为单位矩阵
...
int modelLoc = GL20.glGetUniformLocation(shaderProgram, "model"); // 获取模型变换矩阵的统一变量位置
FloatBuffer modelBuffer = BufferUtils.createFloatBuffer(16);
model.get(modelBuffer); // 将矩阵转换为缓冲区
GL20.glUniformMatrix4fv(modelLoc, false, modelBuffer); // 设置模型变换矩阵的值

// 设置视图变换矩阵
Matrix4f view = new Matrix4f(); // 初始化为单位矩阵
view.lookAt(0.0f, 0.0f, 0.0f, // 相机位置
            0.0f, 0.0f, -1.0f, // 相机朝向
            0.0f, 1.0f, 0.0f); // 相机上方向
int viewLoc = GL20.glGetUniformLocation(shaderProgram, "view"); // 获取视图变换矩阵的统一变量位置
FloatBuffer viewBuffer = BufferUtils.createFloatBuffer(16);
view.get(viewBuffer); // 将矩阵转换为缓冲区
GL20.glUniformMatrix4fv(viewLoc, false, viewBuffer); // 设置视图变换矩阵的值

// 设置投影变换矩阵
Matrix4f projection = new Matrix4f(); // 初始化为单位矩阵
projection.frustum(-1.0f, 1.0f, -1.0f, 1.0f, 1.0f, 10.0f); // 设置透视投影
int projectionLoc = GL20.glGetUniformLocation(shaderProgram, "projection"); // 获取投影变换矩阵的统一变量位置
FloatBuffer projectionBuffer = BufferUtils.createFloatBuffer(16);
projection.get(projectionBuffer); // 将矩阵转换为缓冲区
GL20.glUniformMatrix4fv(projectionLoc, false, projectionBuffer); // 设置投影变换矩阵的值
...
// 绘制三角形
GL11.glDrawArrays(...); // 绘制三个顶点
...

```

在Shader程序中，我们**定义了aPos、model、view、projection等变量用来接收用户传入的顶点数据、模型变换矩阵、视图变换矩阵、投影变换矩阵**。最后，当用户调用glDrawArrays后，触发可编程管线运行，可编程管线在顶点着色器阶段运行顶点Shader，这样渲染过程就运转起来了。




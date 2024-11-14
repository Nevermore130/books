### GLSL 语法与内建函数 ###

这个部分的目标就是实现一组着色器来完成增强对比度的功能，但是这组着色器还不能直接看到效果，因为着色器是需要运行到显卡中的

GLSL（OpenGL Shading Language）就是 OpenGL 为了实现着色器给开发人员提供的一种开发语言。

GLSL 中变量的修饰符，具体如下：

<img src="../images/1703e04fd6c27993cbb8d9e94bfa9a1e.png" alt="img" style="zoom:30%;" />

**GLSL 的基本数据类型：int、float、bool**，这些都是和 C 语言一致的，有一点需要强调的就是，GLSL 中的 float 是可以再加一个修饰符的，这个修饰符用来指定精度。修饰符的可选项有三种：

<img src="../images/349c929770ff5f73aa312edf86216yy7.png" alt="img" style="zoom:33%;" />

向量类型是 Shader 中最常用的一个数据类型，因为在做数据传递的时候经常要传递多个参数，相较于写多个基本数据类型，使用向量类型更加简单

比如，通过 OpenGL 接口把物体坐标和纹理坐标传递到顶点着色器中，用的就是向量类型。每个顶点都是一个四维向量，在顶点着色器中利用这两个四维向量就能去做自己的运算

```c
attribute vec4 position;
```

矩阵类型在 GLSL 中同样也是一个非常重要的数据类型，**在某些效果器的开发中，需要开发者自己传入一些矩阵类型的数据，用于像素计算。比如 GPUImage 中的怀旧效果器，就需要传入一个矩阵来改变原始的像素数据**

```
uniform lowp mat4 colorMatrix;
```

上面的代码表示的是一个 44 的浮点矩阵，如果是 mat2 的声明，代表的就是 22 的浮点矩阵，而 mat3 代表的就是 3*3 的浮点矩阵。OpenGL 为开发者提供了以下接口，把内存中的数据（mColorMatrixLocation）传递给着色器。

```c
glUniformMatrix4fv(mColorMatrixLocation, 1, false, mColorMatrix);
//mColorMatrix 是这个变量在接口程序中的句柄。这里一定要注意，上边的这个函数不属于 GLSL 部分，而是属于客户端代码
```

下面 GLSL 代码是**二维纹理类型**的声明方式。

```
uniform sampler2D texSampler；
```

那客户端如何写代码来把图像传递进来呢？**首先我们需要拿到这个变量的句柄，定义为 mGLUniformTexture**，然后就可以给它绑定一个纹理

```c
glActiveTexture(GL_TEXTURE0); //激活的是哪个纹理句柄
glBindTexture(GL_TEXTURE_2D, texId);
glUniform1i(mGLUniformTexture, 0);
```

比如说**代码中激活的纹理句柄是 GL_TEXTURE0，对应的第三行代码中的第二个参数 Index 就是 0**，如果激活的纹理句柄是 GL_TEXTURE1，那对应的 Index 就是 1，句柄的个数在不同的平台不一样，但是一般都会在 32 个以上。

在 GLSL 中有一个特殊的修饰符就是 varying，这个修饰符修饰的变量都是用来在顶点着色器和片元着色器之间传递参数的。最常见的使用场景就是在顶点着色器中修饰纹理坐标，顶点着色器会改变这个纹理坐标，然后把这个坐标传递到片元着色器

```c
attribute vec2 texcoord;
varying vec2 v_texcoord;
void main(void)
{
    //计算顶点坐标
    v_texcoord = texcoord;
}

//在片元着色器中也要声明同名的变量，然后使用 texture2D 方法来取出二维纹理中这个纹理坐标点上的纹理像素值
varying vec2 v_texcoord;
vec4 texel = texture2D(texSampler, v_texcoord);
```

取出了这个坐标点上的像素值，就可以进行像素变化操作了，比如说去提高对比度，最终将改变的像素值赋值给 gl_FragColor。

### GLSL 的内置变量与内嵌函数 ###

常见的是两个 Shader 的输出变量，一个是顶点着色器的内置变量 gl_position，**它用来设置顶点转换到屏幕坐标的位置。**

```
vec4 gl_posotion;
```

片元着色器的内置变量 gl_FragColor，用来指定当前纹理坐标所代表的像素点的最终颜色值。

```c
vec4 gl_FragColor;
```



### OpenGL ES 的纹理 ###

* OpenGL 中的纹理用 GLUint 类型来表示，通常我们称之为 Texture 或者 TextureID，可以用来表示图像、视频画面等数据。
* 每个二维纹理都由许多小的片元组成，每一个片元我们可以理解为一个像素点
* 大多数的渲染过程，都是基于纹理进行操作的，最简单的一种方式就是从一个图像文件加载数据，然后上传到显存中构造成一个纹理
















我们使用 OpenGL ES 提供给开发者的接口，创建出了一个 GLProgram，但是如果让这个 GLProgram 运行起来还需要有一个上下文环境来支撑。**由于 OpenGL ES 一开始就是为跨平台设计的，所以它本身并不承担窗口管理以及上下文环境构建的职责**，这个职责需要由各自的平台来承担。

Android 平台使用的是 EGL，EGL 是 Khronos 创建的一个框架，用来给 OpenGL 的输出与设备的屏幕搭建起一个桥梁。EGL 的工作机制是双缓冲模式

有一个 Back Frame Buffer 和一个 Front Frame Buffer，**正常绘制操作的目标都是 Back Frame Buffer，渲染完毕之后，调用 eglSwapBuffer 这个 API，会将绘制完毕的 Back Frame Buffer 与当前的 Front Frame Buffer 进行交换**，然后显示出来

iOS 平台使用的是 EAGL，与 EGL 的双缓冲机制比较类似，**iOS 平台也不允许我们直接渲染到屏幕上，而是渲染到一个叫 renderBuffer 的对象上。这个对象你可以理解为一个特殊的 FrameBuffer，最终再调用 EAGLContext 的 presentRenderBuffer 方法**

### Android 平台的环境搭建

要在 Android 平台上使用 OpenGL ES，最简单的方式是使用 GLSurfaceView，因为不需要开发者搭建 OpenGL ES 的上下文环境以及创建 OpenGL ES 的显示设备，只需要遵循 GLSurfaceView 定义的接口，实现对应的逻辑即可

使用 GLSurfaceView 的缺点也比较明显，就是不够灵活，OpenGL ES 很多核心用法，比如共享上下文，使用起来就会比较麻烦。

OpenGL ES 上下文环境，会直接使用 EGL 提供的 API，**在 Native 层基于 C++ 环境进行搭建。原因是在 Java 层进行搭建的话，对于普通的应用也许可以，但是对于要进行解码或使用第三方库的场景，比如人脸识别，又需要到 Native 层来实施**。考虑到效率和性能，我们这里就直接使用 Native 层的 EGL 来搭建一个 OpenGL ES 的开发环境。

#### 引入头文件与 so 库

必须要在 CMake 构建脚本（CMakeLists.txt）中加入 EGL 这个库，并在使用这个库的 C++ 文件中引入 EGL 对应的头文件。

```c++
#include <EGL/egl.h>
#include <EGL/eglext.h>

```

```
//需要引入的 so 库：
target_link_libraries(videoengine
        # 引入系统的动态库
        EGL
        )
```

这样我们就可以在 Android 的 Native 层中使用 EGL 了，不过要使用 OpenGL ES 给开发者提供的接口，还需要引入 OpenGL ES 对应的头文件与库。

```c++
#include <GLES2/gl2.h>
#include <GLES2/gl2ext.h>
```

需要引入的 so 库，注意这里使用的是 OpenGL ES 2.0 版本。

```c++
target_link_libraries(videoengine
        # 引入系统的动态库
        GLESv2
        )
```

下面我们来看一下如何使用 EGL 搭建出 OpenGL 的上下文环境以及实现窗口的管理。

##### EGLDisplay 作为绘制的目标

EGL 首先要解决的问题是，要告诉 OpenGL ES 绘制的目标在哪里，而 EGLDisplay 就是一个封装系统物理屏幕的数据类型，也就是绘制目标的一个抽象

开发者要调用 eglGetDisplay 这个方法来创建出 EGLDisplay 的对象，在调用这个方法的传参中，常量 EGL_DEFAULT_DISPLAY 会被传进这个方法中，每个厂商在自己的实现中都会返回默认的显示设备

```c++
EGLDisplay display;
if ((display = eglGetDisplay(EGL_DEFAULT_DISPLAY)) == EGL_NO_DISPLAY) {
    LOGE("eglGetDisplay() returned error %d", eglGetError());
    return false;
}
```

获得了 EGLDisplay 的对象之后，就需要对这个对象做初始化工作，开发者需要调用 **EGL 提供的 eglInitialize 方法来初始化这个对象**

函数的第一个参数就是 EGLDisplay 对象，后面两个参数是这个函数为了返回 EGL 版本号而设计的，两个参数分别是 Major 和 Minor 的 Version

```c++
if (!eglInitialize(display, 0, 0)) {
    LOGE("eglInitialize() returned error %d", eglGetError());
    return false;
}
```

一旦 EGLDisplay 初始化成功之后，它就可以**将 OpenGL ES 的输出和设备的屏幕桥接起来**

但是需要我们指定一些配置项，比如色彩格式、像素格式、OpenGL 版本以及 SurfaceType 等，不同的系统以及平台使用的 EGL 标准是不同的，在 Android 平台下一般配置的代码如下所示：

```c++
EGLConfig config;
const EGLint attribs[] = {EGL_BUFFER_SIZE, 32,
        EGL_ALPHA_SIZE, 8,
        EGL_BLUE_SIZE, 8,
        EGL_GREEN_SIZE, 8,
        EGL_RED_SIZE, 8,    
        EGL_RENDERABLE_TYPE, EGL_OPENGL_ES2_BIT,
        EGL_SURFACE_TYPE, EGL_WINDOW_BIT,
        EGL_NONE };
if (!eglChooseConfig(display, attribs, &config, 1, &numConfigs)) {
    LOGE("eglChooseConfig() returned error %d", eglGetError());
    return false;
}
```

**EGLDisplay 这个对象是 EGL 给开发者提供的最重要的入口，**接下来我们要基于这个 EGLDisplay 对象还有 EGLConfig 配置来创建上下文了。

##### EGLContext 提供线程的上下文

由于**任何一条 OpenGL ES 指令（OpenGL ES 提供给开发者的接口）都必须运行在自己的 OpenGL 上下文环境中**，EGL 提供 EGLContext 来封装上下文，

```c++
EGLContext context;
EGLint attributes[] = { EGL_CONTEXT_CLIENT_VERSION, 2, EGL_NONE };
if (!(context = eglCreateContext(display, config, NULL,
        eglContextAttributes))) {
    LOGE("eglCreateContext() returned error %d", eglGetError());
    return false;
}
```

第三个参数类型也是 EGLContext 类型的，一般会有两种用法。

* 如果想和已经存在的某个上下文共享 OpenGL 资源（包括纹理 ID、frameBuffer 以及其他的 Buffer），则传入对应的那个上下文变量。
* 如果目标仅仅是创建一个独立的上下文，不需要和其他 OpenGL ES 的上下文共享任何资源，则设置为 NULL。

在一些场景下其实需要**多个线程共同执行 OpenGL ES 的渲染操作，这种情况下就需要用到共享上下文**，共享上下文的关键点就在这里

成功创建出 OpenGL ES 的上下文，说明我们已经把 OpenGL ES 的绘制目标搞定了，但是这个绘制目标并没有渲染到我们某个 View 上，**那如何将这个输出渲染到业务指定的 View 上呢？答案就在 EGLSurface。**

#### EGLSurface 将 EGLDisplay 与系统屏幕桥接起来

EGLSurface 实际上是一个 FrameBuffer，**开发者可以调用 EGL 提供的 eglCreateWindowSurface 创建一个可实际显示的 Surface**

调用 **eglCreatePbufferSuface 可以创建一个 OffScreen（离屏渲染，一般用户后台保存场景）的 Surface**

```c++
EGLSurface surface = NULL;
EGLint format;
if (!eglGetConfigAttrib(display, config, EGL_NATIVE_VISUAL_ID,
        &format)) {
    LOGE("eglGetConfigAttrib() returned error %d", eglGetError());
    return surface;
}
ANativeWindow_setBuffersGeometry(_window, 0, 0, format);
if (!(surface = eglCreateWindowSurface(display, config, _window, 0))) {
    LOGE("eglCreateWindowSurface() returned error %d", eglGetError());
}
```

这个 _window 是 **ANativeWindow 类型的对象，代表了本地业务层想要绘制到的目标 View。**

在 Android 里面可以通过 **Surface（通过 SurfaceView 或 TextureView 来得到或者构建出 Surface 对象）去构建出 ANativeWindow**。

```c++
#include <android/native_window.h>
#include <android/native_window_jni.h>

//surface 就是SurfaceView中传过来的surface
ANativeWindow* window = ANativeWindow_fromSurface(env, surface);
```

里面的 env 是 JNI 层的 JNIEnv 指针类型的变量，surface 就是 jobject 类型的变量，是由 Java 层的 Surface 类型对象传递而来的。到这里我们就把 **EGLSurface 和 Java 层的 View（即设备的屏幕）连接起来了**。

如果想做离屏渲染，也就是在后台使用 OpenGL 处理一些图像，就需要用到处理图像的 Surface 了，创建离屏 Surface 如下：

```c++
EGLSurface surface;
EGLint PbufferAttributes[] = { EGL_WIDTH, width, EGL_HEIGHT, height, EGL_NONE,
        EGL_NONE };
if (!(surface = eglCreatePbufferSurface(display, config, PbufferAttributes))) {
    LOGE("eglCreatePbufferSurface() returned error %d", eglGetError());
}
```

可以看到这个 Surface 并不会和任何一个业务 View 进行关联，进行离屏渲染的时候，就可以把这个 Surface 当作目标进行绘制。

#### 为绘制线程绑定上下文

OpenGL ES 需要开发者自己开辟一个新的线程，来执行 OpenGL ES 的渲染操作，还要求开发者在执行渲染操作前要为这个线程绑定上下文环境。EGL 为绑定上下文环境提供了 eglMakeCurrent 这个接口。

```c++
eglMakeCurrent(display, eglSurface, eglSurface, context);

//在绑定了上下文环境以及窗口之后就可以执行 RenderLoop 循环了，每一次循环都是去调用 OpenGL ES 指令绘制图像。
```

当渲染操作完成之后，要调用函数 eglSwapBuffers 进行前台 frameBuffer 和后台 frameBuffer 的交换。

```c++
eglSwapBuffers(display, eglSurface)
```

#### 销毁资源

```
eglDestroySurface(display, eglSurface);

eglDestroyContext(display, context);
```






































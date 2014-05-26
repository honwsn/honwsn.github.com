---
layout: post
title: "android webkit渲染中的正交投影与视口变换"
date: 2014-05-23 19:21:20 +0800
comments: true
categories: opengl android webkit
tags: 正交投影 视口 webkit ortho shaderprogram glScissor glViewPort
---

这里讨论的是openGL**正交投影矩阵**的概念，结合webkit渲染中的应用分析了下对相关接口的认识。  

###正交投影矩阵与glViewPort视口的关联

andriod4.0以后开启硬件加速，在webkit页面渲染方面，android适配层也有硬件加速的方案，下面的代码就来自android 4.2 webkit 源码android平台在渲染侧的适配。

[GLUtils.cpp](https://android.googlesource.com/platform/external/webkit/+/9b03322/Source/WebCore/platform/graphics/android/rendering/GLUtils.cpp)(打不开，就需要翻墙了，android.googlesource.com)

{% coderay lang:cplusplus %}
void GLUtils::setOrthographicMatrix(TransformationMatrix& ortho, float left,
float top,float right, float bottom, float nearZ,
float farZ)  
{  
float deltaX = right - left;  
float deltaY = top - bottom;  
float deltaZ = farZ - nearZ;  
if (!deltaX || !deltaY || !deltaZ)  
return;  
  
ortho.setM11(2.0f / deltaX);  
ortho.setM41(-(right + left) / deltaX);  
ortho.setM22(2.0f / deltaY);  
ortho.setM42(-(top + bottom) / deltaY);  
ortho.setM33(-2.0f / deltaZ);  
ortho.setM43(-(nearZ + farZ) / deltaZ);  
}
{% endcoderay %}
  

上面的参数坐标是窗口坐标,定义了正交投影在窗口中的区域（位置和大小），返回的ortho是建立好的正交投影矩阵，具体公式原理，可以参考[这里](http://blog.csdn.net/popy007/article/details/4126809)

 
{% img center /images/2014-5/ortho.png 1024 122%}


  
若正交投影坐标矩阵的区域大小若与glViewPort指定的视口大小不一致，则绘制画面就会被拉伸或者缩小。窗口点坐标经过正交矩阵变换后就会被映射到**OpenGL viewport
坐标系（坐标范围是-1，1）**，但glViewPort指定的窗口区域物理大小与正交投影坐标系指定的窗口区域物理大小不一致（
GLUtils::setOrthographicMatrix），所以出现上述情况。
<!--more-->
  

例如下面，我是把ShaderProgram.cpp::setupDrawing,GLUtils::setOrthographicMatrix
中的高度改成了原来的一半,但glviewport的值没有变动，所以，画面就被拉伸了。

{% img center /images/2014-5/halforthoforviewport.png 500 900%}

  


### glScissor与glViewPort关系
  
  二者使用的坐标系都是窗口坐标系以窗口左下角为原点
  
  glScissor是对RenderBuffer进行裁剪。超出裁剪区域的不会进行绘制操作。
  glViewPort是指明RenderBuffer之上的视口大小，这个视口就是后面opengl指令的绘制对象。opengl使用的顶点坐标，坐标系为（-1，1），中间是（0，0），即视口（glViewPort指定）的中间是坐标原点。此外glviewport还定义了opengl坐标到窗口坐标系(使用的像素坐标)的转换，后面将有提到。

  
在裁剪方面，glScissor与glViewPort的区域大小应该是相同的，是一种[互补的关系](http://gamedev.stackexchange.com/questions/40704/what-is-the-
purpose-of-glscissor)

>They are more complementary than alternatives to each other. You almost always
want to set the scissor rectangle to the same values as the viewport.

>glViewport() specifies a transformation from normalized projection space to
screen space. Polygons are clipped to the edge of projection space, **but other
draw operations like glClear() are not**. So, you use glViewport() to determine
the location and size of the screen space viewport region, but the rasterizer
can still occasionally render pixels outside that region.

>That's where scissor in comes in. glScissor() defines a screen space rectangle
beyond which _nothing_ is drawn (if the scissor test is enabled).

>So for example, the following code will clear the whole screen, even though
the viewport is set to a small portion of the larger window:

 {% coderay lang:c %}  
    
    glViewport(200,200,100,100);
    glClear(GL_COLOR_BUFFER_BIT);
 {% endcoderay %}


>Adding glScissor() and enabling the scissor test (which is disabled by
default) restricts the clear.
{% coderay lang:c %}

  	glViewport(200,200,100,100);
    glScissor(200,200,100,100);
    glEnable(GL_SCISSOR_TEST);
    glClear(GL_COLOR_BUFFER_BIT);
 {% endcoderay %}

>Occasionally you come across an implementation that automatically scissors to
the viewport region, but that violates the GL specification.

>Beyond that, the scissor rectangle can be used to temporarily restrict drawing
to a sub-rectangle of the viewport, for special effects, UI elements,
etc.


**就是说glScissor可以只让glViewPort内一部分区域绘制**

 
###opengl坐标转换为窗口坐标

前面说过glViewport指定了opengl坐标系（-1，1 normalized device
coordinates）如何转换到窗口像素坐标系?

公式代码如下：
把opengl坐标openglX,opengY(in -1,1 range)转换到窗口坐标（以左下角为原点的像素坐标）winX，winY。
{% coderay lang:c %}
winX = (openglX+1)/2*GLViewPort.width() + GLViewport.x();
winY = (openglY+1)/2*GLViewPort.height() + GLViewport.Y();
{% endcoderay %}

下面以android 42 webkit 绘制为例，说明下其中的opengl坐标到窗口坐标的转换。

[shaderprogram.cpp](https://android.googlesource.com/platform/external/webkit/+/e927713a108744aab661abdf0ee4b8d98349a1bd/Source/WebCore/platform/graphics/android/rendering/ShaderProgram.cpp)

 ShaderProgram::debugMatrixTransform说明一个道理，坐标不一定是存储在顶点rect里面的，有可能是在matrix里面，在opengles里面这样做的好处有两点：
 
 - 每个绘制单元的顶点坐标都是一样的，这样向gpu传输一次顶点坐标即可，减少了顶点数据的存储和带宽
 - 顶点变换存储在matrix里面，而matrix作为一致性变量传给顶点着色器进行处理。
 - debugMatrixTransform 只是个调试接口，matrix.mapRect的输出是这个绘制单元的最终opengles坐标。
    
{% coderay lang:cpluscplus linenos:true shaderprogram.cpp https://android.googlesource.com/platform/external/webkit/+/e927713a108744aab661abdf0ee4b8d98349a1bd/Source/WebCore/platform/graphics/android/rendering/ShaderProgram.cpp link %}
#if DEBUG_MATRIX


FloatRect ShaderProgram::debugMatrixTransform(const TransformationMatrix& matrix,const char* matrixName)
{
    FloatRect rect(0.0, 0.0, 1.0, 1.0);

    rect = matrix.mapRect(rect);
    ALOGV("After %s matrix:\n %f, %f rect.width() %f rect.height() %f",
          matrixName, rect.x(), rect.y(), rect.width(), rect.height());
    return rect;

}

void ShaderProgram::debugMatrixInfo(float currentScale,
                                    const TransformationMatrix& clipProjectionMatrix,
                                    const TransformationMatrix& webViewMatrix,
                                    const TransformationMatrix& modifiedDrawMatrix,
                                    const TransformationMatrix* layerMatrix)
{
    int viewport[4];
    glGetIntegerv(GL_VIEWPORT, viewport);
    ALOGV("viewport %d, %d, %d, %d , currentScale %f",
          viewport[0], viewport[1], viewport[2], viewport[3], currentScale);
    IntRect currentGLViewport(viewport[0], viewport[1], viewport[2], viewport[3]);

    TransformationMatrix scaleMatrix;
    scaleMatrix.scale3d(currentScale, currentScale, 1.0);

    if (layerMatrix)
        debugMatrixTransform(*layerMatrix, "layerMatrix");

    TransformationMatrix debugMatrix = scaleMatrix * modifiedDrawMatrix;
    debugMatrixTransform(debugMatrix, "scaleMatrix * modifiedDrawMatrix");

    debugMatrix = webViewMatrix * debugMatrix;
    debugMatrixTransform(debugMatrix, "webViewMatrix * scaleMatrix * modifiedDrawMatrix");

    debugMatrix = clipProjectionMatrix * debugMatrix;
    FloatRect finalRect =
        debugMatrixTransform(debugMatrix, "all Matrix");
     
     //finalRect 是opengles的某个区域坐标，in a (-1, 1) range
     //后面输出是以左下角为原点的窗口坐标
       
    // After projection, we will be in a (-1, 1) range and now we can map it back
    // to the (x,y) -> (x+width, y+height)
    ALOGV("final convert to screen coord x, y %f, %f width %f height %f , ",
          (finalRect.x() + 1) / 2 * currentGLViewport.width() + currentGLViewport.x(),
          (finalRect.y() + 1) / 2 * currentGLViewport.height() + currentGLViewport.y(),
          finalRect.width() * currentGLViewport.width() / 2,
          finalRect.height() * currentGLViewport.height() / 2);
         
}
#endif // DEBUG_MATRIX
{% endcoderay %}

  


  

  



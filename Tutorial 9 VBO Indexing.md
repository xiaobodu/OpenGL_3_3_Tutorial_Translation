第九课：VBO索引
===
[TOC]

Tags: OpenGL 教程

索引的原理
---
到目前为止，我们在创建VBO时总是重复存储三角形公共边的顶点。

本课将介绍索引。索引是通过*索引缓冲（index buffer）*来实现的顶点复用。

![indexing](http://www.opengl-tutorial.org/assets/images/tuto-9-vbo-indexing/indexing1.png)

公共顶点？不同顶点？
---
来看看法线的例子。下图中，艺术家创建了两个三角形来表示一个平滑曲面。因此，我们可以把两个三角形的法线混合成一个顶点的法线。为了便于观察，我画了一条红线表示平滑曲面。

![goodsmooth](http://www.opengl-tutorial.org/assets/images/tuto-9-vbo-indexing/goodsmooth.png)

然而在第二幅图中，艺术家想画的是“缝隙”或者“粗糙的边缘”。若混合了法线，就意味着色器会像进行平滑插值，形成一个和上图一样的平滑的表面：

![badmooth](http://www.opengl-tutorial.org/assets/images/tuto-9-vbo-indexing/badmooth.png)

因此这种情况下，我们应该为这个顶点设置两条不同的法线；在OpenGL中，唯一可行的办法是把顶点属性完整复制一份。

![spiky](http://www.opengl-tutorial.org/assets/images/tuto-9-vbo-indexing/spiky.png)

OpenGL中的索引VBO
---
索引的用法很简单。首先，需要再创建一个缓冲来存储索引。代码与之前一样，不过参数由`ARRAY_BUFFER`变为了`ELEMENT_ARRAY_BUFFER`。

```cpp
std::vector<unsigned int> indices;

// fill "indices" as needed

// Generate a buffer for the indices
GLuint elementbuffer;
glGenBuffers(1, &elementbuffer);
glBindBuffer(GL_ELEMENT_ARRAY_BUFFER, elementbuffer);
glBufferData(GL_ELEMENT_ARRAY_BUFFER, indices.size() * sizeof(unsigned int), &indices[0], GL_STATIC_DRAW);
```

只需把`glDrawArrays`替换为如下语句，即可绘制模型：

```cpp
// Index buffer
glBindBuffer(GL_ELEMENT_ARRAY_BUFFER, elementbuffer);

// Draw the triangles !
glDrawElements(
GL_TRIANGLES,      // mode
indices.size(),    // count
GL_UNSIGNED_INT,   // type
(void*)0           // element array buffer offset
);
```
（小提示：最好使用`unsigned short`，不要用`unsigned int`。这样更节省空间，速度也更快。）

填充索引缓冲
---
现在真正的问题来了。如前所述，OpenGL只能使用一个索引缓冲，而OBJ（及一些其他常用的3D格式，如Collada）每个属性都有一个索引缓冲。这意味着，必须通过某种方式把若干个索引缓冲合并成一个。

合并算法如下：

For each input vertex
Try to find a similar ( = same for all attributes ) vertex between all those we already output
If found :
A similar vertex is already in the VBO, use it instead !
If not found :
No similar vertex found, add it to the VBO


完整的C++代码位于`common/vboindexer.cpp`，注释很详尽。如果理解了以上算法，读懂代码应该没问题。

若两顶点的坐标、UV坐标和法线都相等，则认为两顶点是同一顶点。若还有其他属性，就得视具体情况而定了。

为了简单，我们采用了性能较差的线性搜索来寻找相似顶点。实际中用`std::map`更合适。

补充：FPS计数器
---
虽与索引无直接关系，但现在去看看[“FPS计数器”](http://www.opengl-tutorial.org/miscellaneous/an-fps-counter/)是很合适的——这样我们就能看到索引究竟能提升多少性能。[“工具——调试器”](http://www.opengl-tutorial.org/miscellaneous/useful-tools-links/#header-4)中还有其他与性能相关的工具。


> &copy; http://www.opengl-tutorial.org/

> Written with [Cmd Markdown](https://www.zybuluo.com/mdeditor).
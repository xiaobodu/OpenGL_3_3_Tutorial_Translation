第七课：模型加载
===
[TOC]

Tags: OpenGL 教程

目前为止，我们一直在用硬编码描述立方体。您一定也觉得这是种很笨拙、很麻烦的办法。

本课将学习从文件中加载3D模型。和加载纹理类似，我们先写一个小的、功能有限的加载器，接着再为大家介绍几个比我们写的更好的、实用的库。

为了让课程尽可能简单，我们将采用简单、常用的OBJ格式。同样也是出于简单原则，我们只处理每个顶点有一个UV坐标和一个法线的OBJ文件（目前不需要知道什么是法线）。

加载OBJ模型
---
加载函数在`common/objloader.hpp`中声明，在`common/objloader.cpp`中实现。函数原型如下：
```cpp
bool loadOBJ(
const char * path,
std::vector < glm::vec3 > & out_vertices,
std::vector < glm::vec2 > & out_uvs,
std::vector < glm::vec3 > & out_normals
)
```
我们让`loadOBJ`读取文件路径，把数据写入`out_vertices/out_uvs/out_normals`。如果出错则返回false。`std::vector`是C++中的数组，可存放`glm::vec3`类型的数据，数组大小可任意修改，不过`std::vector`和数学中的向量（vector）是两码事。其实它只是个数组。最后提一点，符号&意思是这个函数将会直接修改这些数组。

###OBJ文件示例
OBJ文件大概是这个模样：

# Blender3D v249 OBJ File: untitled.blend
# www.blender3d.org
mtllib cube.mtl
v 1.000000 -1.000000 -1.000000
v 1.000000 -1.000000 1.000000
v -1.000000 -1.000000 1.000000
v -1.000000 -1.000000 -1.000000
v 1.000000 1.000000 -1.000000
v 0.999999 1.000000 1.000001
v -1.000000 1.000000 1.000000
v -1.000000 1.000000 -1.000000
vt 0.748573 0.750412
vt 0.749279 0.501284
vt 0.999110 0.501077
vt 0.999455 0.750380
vt 0.250471 0.500702
vt 0.249682 0.749677
vt 0.001085 0.750380
vt 0.001517 0.499994
vt 0.499422 0.500239
vt 0.500149 0.750166
vt 0.748355 0.998230
vt 0.500193 0.998728
vt 0.498993 0.250415
vt 0.748953 0.250920
vn 0.000000 0.000000 -1.000000
vn -1.000000 -0.000000 -0.000000
vn -0.000000 -0.000000 1.000000
vn -0.000001 0.000000 1.000000
vn 1.000000 -0.000000 0.000000
vn 1.000000 0.000000 0.000001
vn 0.000000 1.000000 -0.000000
vn -0.000000 -1.000000 0.000000
usemtl Material_ray.png
s off
f 5/1/1 1/2/1 4/3/1
f 5/1/1 4/3/1 8/4/1
f 3/5/2 7/6/2 8/7/2
f 3/5/2 8/7/2 4/8/2
f 2/9/3 6/10/3 3/5/3
f 6/10/4 7/6/4 3/5/4
f 1/2/5 5/1/5 2/9/5
f 5/1/6 6/10/6 2/9/6
f 5/1/7 8/11/7 6/10/7
f 8/11/7 7/12/7 6/10/7
f 1/2/8 2/9/8 3/13/8
f 1/2/8 3/13/8 4/14/8

因此：

- #是注释标记，就像C++中的//
- `usemtl`和`mtlib`描述了模型的外观。本课用不到。
- v代表顶点
- vt代表顶点的纹理坐标
- vn代表顶点的法线
- f代表面

v vt vn都很好理解。f比较麻烦。例如`f 8/11/7 7/12/7 6/10/7`：

- 8/11/7描述了三角形的第一个顶点
- 7/12/7描述了三角形的第二个顶点
- 6/10/7描述了三角形的第三个顶点
- 对于第一个顶点，8指向要用的顶点。此例中是-1.000000 1.000000 -1.000000（C++中索引从0开始，OBJ文件中索引从1开始，）
- 11指向要用的纹理坐标。此例中是0.748355 0.998230。
- 7指向要用的法线。此例中是0.000000 1.000000 -0.000000。

我们称这些数字为索引。若几个顶点共用同一个坐标，索引就显得很方便，文件中只需保存一个“V”，可以多次引用，节省了存储空间。

其弊端在于我们无法让OpenGL混用顶点、纹理和法线索引。因此本课采用的方法是创建一个标准的、未加索引的模型。等第九课时再讨论索引，届时将会介绍如何解决OpenGL的索引问题。

###用Blender创建OBJ文件
我们写的蹩脚加载器功能实在有限，因此在导出模型时得格外小心。下图展示了在Blender中导出模型的情形：

![Blender](http://www.opengl-tutorial.org/assets/images/tuto-7-model-loading/Blender.png)

###读取OBJ文件
OK，真正开始写代码了。我们需要一些临时变量存储.obj文件的内容：
```cpp
std::vector< unsigned int > vertexIndices, uvIndices, normalIndices;
std::vector< glm::vec3 > temp_vertices;
std::vector< glm::vec2 > temp_uvs;
std::vector< glm::vec3 > temp_normals;
```
学第五课带纹理的立方体时您你已学会打开文件了：
```cpp
FILE * file = fopen(path, "r");
if( file == NULL ){
printf("Impossible to open the file !\n");
return false;
}
```
读文件直到文件末尾：
```cpp
while( 1 ){

char lineHeader[128];
// read the first word of the line
int res = fscanf(file, "%s", lineHeader);
if (res == EOF)
break; // EOF = End Of File. Quit the loop.

// else : parse lineHeader
```
（注意，我们假设第一行的文字长度不超过128，这样做太笨了。但既然这只是个实验品，就凑合一下吧）

首先处理顶点：
```cpp
if ( strcmp( lineHeader, "v" ) == 0 ){
glm::vec3 vertex;
fscanf(file, "%f %f %f\n", &vertex.x, &vertex.y, &vertex.z );
temp_vertices.push_back(vertex);
```

也就是说，若第一个字是“v”，则后面一定是3个float值，于是以这3个值创建一个`glm::vec3`变量，将其添加到数组。
```cpp
} else if ( strcmp( lineHeader, "vt" ) == 0 ){
glm::vec2 uv;
fscanf(file, "%f %f\n", &uv.x, &uv.y );
temp_uvs.push_back(uv);
```
也就是说，如果不是“v”而是“vt”，那后面一定是2个float值，于是以这2个值创建一个`glm::vec2`变量，添加到数组。

以同样的方式处理法线：
```cpp
} else if ( strcmp( lineHeader, "vn" ) == 0 ){
glm::vec3 normal;
fscanf(file, "%f %f %f\n", &normal.x, &normal.y, &normal.z );
temp_normals.push_back(normal);
```

接下来是“f”，略难一些：
```cpp
} else if ( strcmp( lineHeader, "f" ) == 0 ){
std::string vertex1, vertex2, vertex3;
unsigned int vertexIndex[3], uvIndex[3], normalIndex[3];
int matches = fscanf(file, "%d/%d/%d %d/%d/%d %d/%d/%d\n", &vertexIndex[0], &uvIndex[0], &normalIndex[0], &vertexIndex[1], &uvIndex[1], &normalIndex[1], &vertexIndex[2], &uvIndex[2], &normalIndex[2] );
if (matches != 9){
printf("File can't be read by our simple parser : ( Try exporting with other options\n");
return false;
}
vertexIndices.push_back(vertexIndex[0]);
vertexIndices.push_back(vertexIndex[1]);
vertexIndices.push_back(vertexIndex[2]);
uvIndices    .push_back(uvIndex[0]);
uvIndices    .push_back(uvIndex[1]);
uvIndices    .push_back(uvIndex[2]);
normalIndices.push_back(normalIndex[0]);
normalIndices.push_back(normalIndex[1]);
normalIndices.push_back(normalIndex[2]);
```
代码与前面的类似，只不过读取的数据多一些。

###处理数据
我们只需改变一下数据的形式。读取的是字符串，现在有了一组数组。这还不够，我们得把数据组织成OpenGL要求的形式。也就是去掉索引，只保留顶点坐标数据。这步操作称为索引。

遍历每个三角形（每个“f”行）的每个顶点（每个 v/vt/vn）：
```cpp
// For each vertex of each triangle
for( unsigned int i=0; i<vertexIndices.size(); i++ ){
```
顶点坐标的索引存放到`vertexIndices[i]`：
```cpp
unsigned int vertexIndex = vertexIndices[i];
```
因此坐标是`temp_vertices[ vertexIndex-1 ]`（-1是因为C++的下标从0开始，而OBJ的索引从1开始，还记得吗？）：
```cpp
glm::vec3 vertex = temp_vertices[ vertexIndex-1 ];
```
这样就有了一个顶点坐标：
```cpp
out_vertices.push_back(vertex);
```
UV和法线同理，任务完成！

使用加载的数据
---
到这一步，几乎没什么变化。这次我们不再声明`static const GLfloat g_vertex_buffer_data[] = {…}`，而是创建一个顶点数组（UV和法线同理）。用正确的参数调用`loadOBJ`：
```cpp
// Read our .obj file
std::vector< glm::vec3 > vertices;
std::vector< glm::vec2 > uvs;
std::vector< glm::vec3 > normals; // Won't be used at the moment.
bool res = loadOBJ("cube.obj", vertices, uvs, normals);
```
把数组传给OpenGL：
```cpp
glBufferData(GL_ARRAY_BUFFER, vertices.size() * sizeof(glm::vec3), &vertices[0], GL_STATIC_DRAW);
```
就是这样啦！

结果
---
不好意思，这个纹理不大漂亮。我不太擅长美术:(。欢迎您提供一些漂亮的纹理。

![ModelLoading](http://www.opengl-tutorial.org/assets/images/tuto-7-model-loading/ModelLoading.png)


其他模型格式及加载器
---
这个小巧的加载器应该比较适合初学，不过别在实际中使用它。参考一下[实用链接和工具](http://www.opengl-tutorial.org/miscellaneous/useful-tools-links/)页面，看看有什么能用的。不过请注意，直到第九课才会*真正*用到这些工具。

> &copy; http://www.opengl-tutorial.org/

> Written with [Cmd Markdown](https://www.zybuluo.com/mdeditor).
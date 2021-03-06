<center><h1> Mesh simplification and remesh <h1></center>
<p align="right">-- GAMES101 final project</p>

视频地址：
链接：https://pan.baidu.com/s/143gGlobRAGIxreGuejShng 
提取码：eip0

<center><h2> Introduction <h2></center>
<p align="center">
有时，三角形网格中的三角形数量远远超过所需要的，从而浪费了存储空间，并且会花费很多的计算时间。为了解决这个问题，我们可以简化网格，在其中找到可以使用较少三角形的区域，然后相应地简化这些区域。另外，还可以找到同一表面更好的离散表示。将网格转换为更好的表示形式称为重新网格化。注意这里由于单一方法实现难度较低，我们要求你实现网格简化与重新网络化两种方法。两种方法不要求同时应用。
</p>

<center><h2>数据结构定义</h2><center>
<h3 align="left">点/边/面的存储</h3>
<p align="left">点是 Vector3d<float><p>
<p align="left">
边是由顶点 v1 和 v2组成的结构体，构造函数初始化 v1 和 v2，重载运算符 < 用于比较边的次序，v1越小次序越靠前，v1 相同时，v2越小次序越靠前。
<p>
<p align="left">
面由顶点 v1, v2, v3 组成的类， 包含三个函数： 
</p>
<p align="left">Face::against 传入一条边找到面的下一条边</p>
<p align="left">Face::reverse 把面翻转(交换v1和v2的值)</p>
<p align="left">Face::replace 把面中的一个顶点改变</p>

### 存放网格属性的变量

- triangles \<Triangle\> 所有三角网格
- vertices \<Vertex\> 所有顶点
- refs \<Ref\>  网格的 reference， 网格 id 和顶点 id
- mtllib 网格所使用的 mtl 文件
- materials 材质名称

<center><h2>网格简化</h2><center>


### 理论参考 - Quadric Error Metrics

网格简化采用二次误差与边塌陷[1\]的方法实现，这种方法来自SIGGRAPH 1997 Surface Simplification Using Quadric Error Metrics（Michael&&PaulS）[1]。

边坍缩（Collapsing Edge）

![image-20210925004646370](./images/collapse.png)

二次度量误差

![image-20210925004912224](./images/quadric.png)

算法基本思想：对网格的每一条边计算二次误差，每次迭代，将二次误差最小的边移除（塌陷这条边），不断迭代并计算二次误差，直到达到目标三角形数。

做出的改进：每次迭代不对二次误差进行排序，而是设定一个阈值，将阈值小于该值的边移除，避免了排序，能够提高速度，但是输出的网格质量降低。

### 误差计算

对于网格中每一个顶点: 定义一个4×4的对称误差矩阵Q，那么

顶点$v=[\quad vx\quad vy\quad vz\quad 1\quad ]^T$ 的误差 $Δ(v)=v^TQv$ 。假设对于一条收缩边 $(v1,v2)$，其收缩后顶点变为 $vbar$ ，我们定义顶点 $vbar$ 的误差矩阵$Qbar=Q1+Q2$，对于如何计算顶点vbar的位置有两种策略：一种简单的策略就是在$v1,v2,(v1+v2)/2$中选择一个使得收缩代价$Δ(vbar)$最小的位置。另一种策略就是数值计算顶点$vbar$位置使得$Δ(vbar)$最小，由于Δ的表达式是一个二次项形式，因此令一阶导数为0，即，该式的解可表示为

![image-20210924131342962](./images/q.png)

求解方法：

1. 对所有的初始顶点计算Q矩阵.
2. 选择所有有效的边（这里取的是联通的边，也可以将距离小于一个阈值的边归为有效边）
3. 对每一条有效边(v1,v2)，计算最优抽取目标$$\bar v$$.误差$$\bar v^T(Q1+Q2)\bar v$$是抽取这条边的代价（cost）
4. 将所有的边按照cost的权值放到一个堆里(这里直接放入向量里)
5. 每次移除代价（cost）最小的边，并且更新包含着v1的所有有效边的代价（这里直接找小于阈值的删除，每次迭代会有不同的阈值）

### 算法实现：

#### 从文件读取网格数据到变量

- 输入： 文件，是否处理uv
- 输出：存在class中的网格
- 用 char[1000] 作为 buffer 暂存文件中的数据流， memset初始化为0
- 为uv处理和材质定义 maps，uvs，uvMap
- 在循环中 fget 文件的 line
- 读取mtllib，找到材质，把材质放进 material_map的末尾（后续写文件要用）
- 判断该行的开头是否为 vt，如果是，则读取uv
- 如果是v，读取顶点
- 如果是f，读取三角形
- read line结束
- 如果 uvs 中有元素，设置每个三角形的uvs
- 关闭文件

#### 写网格数据到文件

- 打开文件
- 如果 mtllib 不为空，写入 mtllib
- 遍历 vertices，写入 v x y z
- 遍历 uvs 写入 vt x y
- 遍历 triangles， 写入 f v1 v2 v3 [u v]
- 关闭文件

#### 网格简化判断
  - 输入：目标网格数，迭代次数
  - 迭代开始前的处理：
    - 每五次迭代，在迭代开始前更新一次网格	
    - 每次迭代前清除上一次迭代做的标记 dirty
    - 判断误差阈值err.threshold，当三角形的边小于threshold时将被清除
    - 如果大于阈值，计算是否清除顶点并标记涉及的三角形：
      - 检查边界
      - 计算(i0,i1)塌陷到的顶点p的二次误差 
      - 如果顶点删除后面会翻转，则不删除顶点
      - 更新uv
    - 移除上一步算出的可移除的边，更新三角形
    - 如果三角形数小于等于目标网格的三角形数，停止。

#### 计算二次误差和塌陷顶点坐标

- 构造 p 的对称矩阵 q 
- 如果系数矩阵可逆，那么通过求解方程就可以得到新顶点vbar的位置
  - 计算出塌陷到的点的坐标
    - vx = A41/det(q_delta)
    - vy = A42/det(q_delta)
    - vz = A43/det(q_delta)
  - 顶点误差 vertex error = q[0]*x*x + 2*q[1]*x*y + 2*q[2]*x*z + 2*q[3]*x + q[4]*y*y\+ 2*q[5]*y*z + 2*q[6]*y + q[7]*z*z + 2*q[8]*z + q[9];
- 如果构造出的行列式为0，系数矩阵不可逆，就通过在v1,v2和(v1+v2)/2中选择一个使得收缩代价Δ(vbar)最小的位置
  - p1=vertices[id_v1].p
  - p2=vertices[id_v2].p
  - p3=(p1+p2)/2;
  - 分别计算三个点与q的误差，取最小值
  - 把误差最小的点作为结果

#### 确认未翻转

- 输入一个三角形网格的 起始点坐标，要删除的两个顶点id和坐标；
- 输出删除边后三角面是否翻转
- 构造 p 到 p1 的向量
- 构造 p 到 p2 的向量
- 用两个向量点积
- 如果结果大于1， 返回true
- 两个向量叉积，得到法向量
- 如果法向量与原来的三角面法向量点积小于0.2，返回true
- 其他情况返回false

### 测试

输入：

![image-20210920101712438](./images/mesh_simplification_input.png)

网格简化结果：

从左到右分别是 

- 输入网格（blender monkey Susana） 
- 0.2简化系数简化结果
- 0.02简化系数简化结果



![image-20210920190842701](./images/mesh_simplify_1_02_002.png)

![image-20210920191218315](./images/mesh_simplification.png)


<center><h2>重新网格化</h2><center>

### 理论参考 - Isotropic Remeshing

采用了 Isotropic Remeshing，该算法由Botsch and Kobbelt 04b[2]提出。 

重新网格化的理想目标：

- 所有边等长
- 所有三角形面积相等
- 所有顶点度数为 6

算法的基本思想：设定一个目标边长，并优化网格中所有的边长向目标边长靠近，并最终得到优化结果。主要分为三个步骤来实现向目标边长的优化，即分割Split，塌缩Collapse和翻转Flip

分割Split

![image-20210925004345855](./images/edge_split.png)

塌缩Collapse

![image-20210925004355289](./images/edge_collapse.png)

翻转Flip

![image-20210925004403582](./images/edge_flip.png)

### 算法实现

#### 总体步骤：

- 加载模型，存储到 顶点 vector 和 三角形 vector 中
- 计算输入模型的包围盒的最长的一条边的长度
- 迭代地进行 remesh：
  - 更新网格
  - 平滑网格
- 迭代地进行边翻转
- 迭代地进行边翻转、边切割、边塌陷、平滑顶点
- 迭代结束后，输出网格

#### 详细实现

更新边：

- 遍历边的列表，对于每一条边依次进行下面的判断（任意一个不满足则 continue）：

  - 是否已经被前面执行的代码移除（查看标志位）；
  - 是否与它相邻的边已经被移除；
  - 是否三角形 id 对应的顶点的对边已经被移除；

- 如果以上三个条件都满足：

- 计算三角形的内角，确保未翻转时边两侧的三角形组成的四边形是凸的；

- 检查边是否为边缘

- 如果顶点a和b的角度之和小于180度，则[a,b]是一个翻转候选

- 进行翻转：

  - 删除边两侧的三角形
  - 添加反转后的边
  - 更新网格

- 计算边长的平方和它的中点以及目标网格数来决定是塌陷还是细分边

  - 如果边比最大边长
  - 如果要细分边:
    - 生成新的顶点作为中点
    - 修改网格
    - 执行网格更新和删除
  - 检查边缘是否太短，在这种情况下它应该被塌陷
  - 如果要删除顶点：
    - 删除顶点对应的三角形
    - 如果三角形只是接触该顶点而不是由该顶点标识，检查折叠前后的法线变化和确保三角形也不会翻转检查最小角度是否大于10度，因为我们不想生成退化三角形

#### 边翻转

- 删除边两侧的三角形
- 添加另外两组边形成的三角形
- 添加反转后的边

#### 边切割

- 从三角形列表删除边两侧的三角形
- 添加顶点到顶点列表
- 向三角形列表添加两组边分别与中点形成的四个三角形

  #### 边塌陷

- 将所有短于`low = 4/5 * target_edge_length`的边进行 Edge Collapse 操作；
- 新增的点位置为边两端点的中点；
- 新的边不会长于`high = 4/3 * target_edge_length`；

  

### 测试

输入网格是一个在局部区域被密集环切的 suzanne
输出网格为图片右边的网格

![image-20210924001742644](./images/remesh_compare.png)

输入网格：
![image-20210924001958182](./images/remesh_input.png)
输出网格：

![image-20210924002047205](./images/remesh_output.png)

### 参考文献

[1] Garland M . Surface Simplification Using Quadric Error Metrics[J]. SIGGRAPH 97, 1997.

[2] Botsch M , Kobbelt L . A Remeshing Approach to Multiresolution Modeling[C]// Second Eurographics Symposium on Geometry Processing, Nice, France, July 8-10, 2004.

视频地址：
链接：https://pan.baidu.com/s/143gGlobRAGIxreGuejShng 
提取码：eip0

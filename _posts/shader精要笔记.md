[TOC]
https://zhuanlan.zhihu.com/p/98810080
### 渲染流水线的概念
### 性能优化的概念
### 坐标系和矢量的概念
### 矩阵的基础概念
### 变换的基本概念
### 空间变换的数学原理
#### 世界空间  
圆点表示为
![image](https://pic3.zhimg.com/80/v2-d44fb8299e5d3de42add266852292242_720w.png)  
前三个0，表示它的坐标为（0，0，0），而最后一个1，是因为三维图形学中规定把点记作的齐次空间坐标的最后一位记作1。  
四维向量表示为  
![image](https://pic2.zhimg.com/80/v2-c7d0b04b38ebe04294919b0d49b509bd_720w.jpg)    
第一列到第三列确定的，其实就是空间的三个轴向，而第四列则代表了点的位置。
#### 点的变换和向量变换  
通过变换公式推出点变换矩阵
![image](https://pic4.zhimg.com/80/v2-4a200d97b0a5004e8b34569a357a7b3b_720w.png)  

展开过程  
![image](https://pic1.zhimg.com/80/v2-b046f44e1671a48eb5ad2df5a4233eb4_720w.jpg)  
就可以得到这个变换矩阵，第一到第三列就是子空间的三个坐标轴对应在父空间中的向量，而第四列就是子空间的坐标原点。

#### 对矢量的空间变换 
因为向量是没有位置信息的，它只表示方向，所以原点在哪里对向量来说无关紧要。那么对向量的坐标空间变换可以使用3x3的矩阵来表示  
在shader中我们经常会看到截取变换矩阵的前三行三列来对法线方向和光照方向进行空间变换。
![image](https://pic4.zhimg.com/80/v2-aeecf76aab144a04838a988f61b389eb_720w.jpg)

#### 空间的反向变换
我们在上面已经求解了从子空间变换到父空间的变换矩阵Mc->p 。从父空间变换到子空间的变换矩阵 Mp->c 就是Mc->p的逆矩阵。逆矩阵的求解比较复杂，但是幸运的是有一条定律是正交矩阵的转置矩阵就是它的逆矩阵。  
空间变换的矩阵就是正交矩阵

### 坐标空间的定义和变换演示
#### 模型空间（model space）
在游戏中的每一个模型或者物体都有各自独立的坐标空间，模型有自己的前后左右，这是它的自身属性，旋转或者是移动和缩放模型并不能改变它的前后左右。

#### 世界空间（world space）
世界空间是最外层的空间，可以被用于描述绝对位置  
在渲染中，顶点变换的第一步就是把顶点从模型空间转换到世界空间。空间变换的第一步就是要构建变换矩阵。构建变换矩阵其实就是根据子空间原点在父空间中进行的变换来再对顶点进行一次同样的变换。


**举例**
![image](https://pic2.zhimg.com/80/v2-6b74fa415aadfdec2a55cd5f7f831add_720w.jpg)
根据上图的ransform属性栏可以看出，模型在世界坐标下进行了（2，2，2）的缩放，围绕y轴旋转了30度，还被移动了（1，3，5）个单位。那么就可以计算变换矩阵M model->world：
![image](https://pic2.zhimg.com/80/v2-05ef62ae7b822b51a214c44e9e48b695_720w.jpg)



### 基本光照模型
点光源  
平行光源  
因为辐照度是和到物体表面之间的距离d/cosθ成反比，和cosθ成正比。cosθ可以使用光源方向 和表面法线 点积来得到。   
光线被吸收和散射，散射包括折射和散射；
散射包括高光反射和漫反射  
#### 标准光照模型
- 环境光
- 自发光
- 漫发射
- 高光反射（经验模型）  
 

####  兰伯特模型
Lambert光照模型主要是用来模拟粗糙物体表面的光照现象   
漫反射的基本特点有两点：
- (1)反射强度与观察者的角度没有关系；
- (2)反射强度与光线的入射角度有关系。
 
最终得到的Lambert余弦定理公式如下   DiffuseLight=Kd*I*dot(N,L)。

#### 半兰伯特模型
首先重申我们的优化目的，为了提高不直接受光表面的亮度，而根据Lambert光照模型求得的这些表面处的光照亮度为0，很显然直接用一个大于0的值替换所有结果0，会造成不直接受光表面亮度大于原本可以接收到微弱光照的部分，因此我们的目的就变成了提升整体亮度，在图形学中，提升整体亮度的常用算法就是Remap，也就是把原始计算结果置换到一个新的较大区间。

Half Lambert光照模型正式对Lambert模型的计算结果进行了Remap，将[0,1]的计算结果置换到了[0.5,1]之间，具体计算公式如下：  
DiffuseLight=Kd*I*(dot(N,L)*0.5+0.5)

#### Phong冯氏光照模型
在兰伯特光照的基础上添加了高光反射，用于模拟光滑平面，在光滑面上有更多的高光反射，和部分漫反射  
高光项计算：Specular = Ks* pow(saturate(dot(R,V)),Gloss)  

- Ks为高光强度；
- R为光线经法向量反射后的向量；
- V为视线向量（注意是由片元位置指向摄像机位置构成的向量，反向不可反）;
- Gloss为光滑度。

完整的光照公式需要再加上漫反射项和环境光项

```
PhongLightResult = AmbientLightResult + DiffuseLightResult + SpecularLightResult  
=AmbientLightColor+Kd*dot(Normal,LightDir)+Ks*pow(saturate(dot(R,V)),Gloss)
```


#### Blinn 光照模型
没有计算反射方向，而是引入一个新的矢量h，
对视角方向V和光照方向I相加再归一化得到的。  
使用这个半程向量，这样在反射向量和视角向量大于90度时，也可以较好的进行光照模拟。  
左图是通过反射向量R和视角向量V计算 为Phong光照，当出现右图情况时，因为大于90度，镜面光取值为0，不能很好的体现实际光照效果
![image](https://img-blog.csdnimg.cn/20190420204310459.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FqaDU2MDY=,size_16,color_FFFFFF,t_70)  

实际的Blinn光照  
![image](https://img-blog.csdnimg.cn/20190420204343287.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FqaDU2MDY=,size_16,color_FFFFFF,t_70)  
Blinn-Phong模型不再依赖于反射向量，
而是采用了所谓的 半程向量(Halfway Vector)，即光线与视线夹角一半方向上的一个单位向量。当半程向量与法线向量越接近时，镜面光分量就越大。  
![image](https://img-blog.csdnimg.cn/20190420204433405.png)  
当观察向量与反射向量越接近，那么半角向量与法向量N越接近，观察者看到的镜面光成分越强。



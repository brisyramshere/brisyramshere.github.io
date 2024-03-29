# ITK中的空间对象

空间对象类：itk::SpatialObject

空间对象类的存在使得我们可以将任意的数据结构，如图像、mesh、点、箭头等，在一个空间中表示。将所有物体在一个空间中进行管理，今儿可以方便的对这些物体的位置关系进行比较，并方便于如配准、图像和分割结果的重叠显示、不同分辨率图像的同空间显示等等。

空间对象类本身不是一种特定的数据类型，而是将任意的数据类型，附上空间变换信息、和对象之间的层级关系信息后，定义在一个新的空间坐标中，所形成的新的类。

## 空间对象的使用场景

### 模型到图像的配准

模型到图谱的配准中，模型的表示就是使用spatial object，他可以方便的和image数据结构进行护信息、互相关、边界到图像的测度。

### 模型到模型的配准

ICP，landmark，表面距离最小化方法等都可以使用ITK的transform，进行刚性、非刚性图像配准，或有限元FEM、基于傅立叶描述子的物体表示。

### 图谱构建

使用空间物体构建图谱，以表示物体的特性，并表征他们常见的变化模式。labels可以与图谱的对象相关联。

### 从一个或多个scans中存储分割结果

分割结果最好是存储在物理坐标系下，因为这样能够比较方便的与在不同分辨率下的其他分割结果进行比较。通过手绘轮廓线、像素标注、或从模型配准映射到图像的分割结果，都可以一视同仁，均可表示为空间物体。

### 表征物体之间的功能或逻辑关系

空间物体可以有父物体或子物体。比如对物体进行查询时，同样可以收到子物体的响应；对父物体施加的transform通用可以穿嗲到子物体。

### 和image结果的互转

提供了函数可以将spatial objects渲染为图像。

### IO 

itk::MetaImageIO类提供了空间物体从硬盘读写的函数。

### 常见的空间物体类型：管状结构、球状结构、图像、表面

itk提供了一些常用的空间物体容器和类型。也支持添加新的类型，只需要在派生类中对一两个成员函数进行重新定义即可。

## 空间物体的层次特性（Hierarchy）

空间物体组合在一起可以形成一个层级树结构。
首先，只允许存在一个父object；
其次，每个object单独存储了一个transform（变换）；
因此，层级关系不能通过有向无环图表示，而是表示为一棵树。

用户负责维持这棵树的结构，确保不会形成环，这在代码中是不会自动进行检查的。

### 空间物体的创建\操作\层级设定

```cpp
#include "itkSpatialObject.h"
//声明一种空间物体类型，3维
using SpatialObjectType = itk::SpatialObject<3>;

//实例化创建两个空间物体
SpatialObjectType::Pointer obj1 = SpatialObjectType::New();
obj1->GetPorperty().SetName("First Object");

SpatialObjectType::Pointer obj2 = new SpatialObjectType::New();
obj2->GetPorperty().SetName("Secend Object");

//将obj2设置为obj1的子object
obj1->AddChild(obj2);
//当object的参数进行了变更（包括层级关系的变化），需要调用update触发生效，父object的update会自动触发子object的update。
obj1->UpDate();

//判断obj是否存在父obj
if(obj2->HasParent()){...};

//获取obj的子obj
SpatialObjectType::ChildrenListType* childList = obj1->GetChildren();
//迭代遍历子节点
SpatialObjectType::ChildrenListType::const_iterator it = childList->begin();
while(it!=childList->end()){
    std::cout<<(*it)->GetPorperty().GetName()<<std::endl;
    ++it;
};
delete childList;

//还可以通过删除子节点解除层级关系
obj1->RemoveChild(obj2);
//对被解除关系后的孤立节点执行一次update
obj2->Update();

//清除物体，并删除子节点
obj1->Clear();//清除物体相关的所有信息，包括data，但是父-子关系不会被清除
obj1->RemoveAllChildren();//删除它的子节点
```

## 空间物体的变换（Transformations）

几个后面会用到的概念：
- 物体空间（Object Space）：物体根据自己的性质内在定义的空间；子节点物体可以添加进入到这个空间。
- 物体到父节点的变换（ObjectToParentTransform）：空间物体仅存在一个他们直接控制的变换：ObjectToParentTransform。这个变换表征了如何将物体本身的物体空间变换到和父节点物体物体空间一致。这是一个仿射变换。这个变换矩阵要求必须为一个可逆矩阵。
- 世界空间（WorldSpace）：世界空间是不能被空间物体直接控制的，除非这个物体处在在层级树的最顶端位置。

空间物体的一些成员函数和变量可以让他们获取到他们所在world space的信息：
- ProtectedComputeObjectToWorldTransform()
- GetObjectToWorldTransform()
- SetObjectToWorldTransform()

### 修改空间物体的scale和offset

```cpp
using SpatialObjectType = itk::SpatialObject<2>;
using TransformType = SpatialObjectType::TransformType;
//首先创建两个2维的空间物体obj1和obj2, 并将obj2添加微obj1的子节点
SpatialObjectType::Pointer obj1 = SpatialObjectType::New();
SpatialObjectType::Pointer obj2 = SpatialObjectType::New();
obj1->GetPorperty().SetName("First Object");
obj2->GetPorperty().SetName("Secend Object");
obj1->AddChild(obj2);

//将obj2放大2倍
double scale[2];
scale[0] = 2;
scale[1] = 2;
obj2->GetModifiableObjectToParentTransform()->Salce(scale);

//对obj1施加一个offset,这个offset也会施加在obj2上
TransformType::OffsetType() objOffset;
objOffset[0] = 4;
objOffset[1] = 3;
obj1->GetModifiableObjectToParentTransform()->SetOffset(objOffset);
obj1->Update();//需要执行update函数重新计算所有的依从的变换；同时obj2也会进行这个变换。
```

### 仿射矩阵的数学表示

仿射变换其实是以矩阵进行表示的。仿射矩阵中唯一有效的两个成员是Matrix和Offset。其他的变换，比如scale，其本质上是对Matrix进行了重新计算。
仿射变换使用以下公式进行计算：

$X^{'} = R * ( S * X - C ) + C +V$

R为旋转矩阵；S
为缩放尺度；C为旋转中心坐标；V为平移向量或ffset。
因此M和T分别定义为：

$M=R*S$

$T=C+V-R*C$

### 显示仿射矩阵的代码示例

To do...

## 空间物体的类型

### 箭头类：ArrowSpatialObject

ArrowSpatialObject表示一个空间中的箭头物体。它包括以下属性需要设置：
- 箭头起点位置属性（PointType表示）：通过SetPositionInObjectSpace()函数设置
- 箭头方向属性（VectorType表示）：通过SetDirectionInObjectSpace()函数设置
- 箭头长度属性（标量表示）：通过SetLengthInObjectSpace()函数设置

以下为简单的实例，描述arrow空间对象的创建流程：
```cpp
#include "itkArrowSpatialObject.h"

using ArrowType = itk::ArrowSpatialObject<3>;
ArrowType::Pointer pArrow = ArrowType::New();

ArrowType::PointType pos;
pos.Fill(1);
pArrow->SetPositionInObjectSpace(pos);// 设置位置属性
pArrow->SetLengthInObjectSpace(2);//设置长度属性
ArrowType::VectorType dir;
dir.Fill(0);
dir[1]=1.0;
pArrow->SetDirectionInObjectSpace(dir);//设置方向属性
```

### 标记点:LandmarkSpatialObject

LandmarkSpatialObject本质上表示的是一个landmark点的列表，每一个landmark点有一个position属性和color属性。

创建一个landmark包含以下步骤：
1. 创建LandmarkPoint, LandmarkPoint表示一个landmark点，它需要设置position属性和color属性；通过:
   - SetPositionInObjectSpace(itk::Point)设置position;
   - SetColor()设置颜色
2. 创建LandmarkPointList装载所有LandmarkPoint；
3. 创建Landmark，设置LandmarkPointList；

下面演示创建一个landmark空间对象的实例：

```cpp
#include "itkLandmarkSpatialObject.h"
//别名简化
using LandmarkType = ikt::LandmarkSpatialObject<3>;
using LandmerkPointer = LandmarkType::Pointer;
using LandmarkPointType = LandmarkType::LandmarkPointType;
using PointType = LandmarkType::PointrType;
//实例化一个landmark对象lmk
LandmerkPointer lmk = LandmarkType::New();
//lmk设置名称和id
lmk.GetPorperty().SetName("Landmark");
lmk->SetId(1);
//创建一个landmark点list变量，并向list添加landmark点；每个landmark点需要设置position+color属性
LandmarkType::LandmarkPointListType list;
for(int i=0;i<5;i++)
{
    LandmarkPointType p;
    PointType pnt;
    pnt[0]=i;pnt[1]=i+1;pnt[2]=i+2;
    p.SetPositionInObjectSpace(pnt);
    p.SetColor(1,0,0,1);
    list.push_back(p);
}
//landmark点的列表添加给landmark对象（lmk）
lmk->SetPoint(list);
lmk->Update();
```

### 斑点类：BolbSpatialObject

BolbSpatialObject表示一个N维的斑点结构。它派生自itkPointBasedSpatialObject.

Bolb和landmark的创建方式非常类似。形式也非常类似。只是它的基础元素是BoldPointType而非LandmarkPointType。

### 椭圆类：EllipseSpatialObject

EllipseSpatialObject表示一个N维的椭圆。

创建一个N维椭圆，需要设置各个维度上的半径，通过SetRadiusInObject(EllipseType::ArrayType radius)实现；

另外还有其他可用的成员函数：
- IsInsideInWorldSpace(ikt::Point)：判断点是否在椭圆内；
- IsEvaluableAtInWorldSpace(itk::Point)：判断点是否是可估值的；
- ValueAtInWorldSpace(itk::Point, double)：获取对应位置的值
- GetMyBoundingBoxInWorldSpace()：获取椭圆的外包矩形框，返回类型为EllipseType::BoundingBoxType的指针。

### 高斯类：GaussianSpatialObject

GaussianSpatialObject表示在空间中的一个N维高斯核。

高斯核的创建通常需要指定以下参数：
- SetMaximum(2):设置高斯核的最大值；
- SetRadiusInObjectSpace(3):设置高斯核的半径。

同样可以通过ValueAtInWorldSpace(itk::Point, value)的方式获取指定点位置的值。

### 组类：GroupSpatialObject

GroupSpatialObject不和任何数据相关联，它主要用于将object分组，或伪object添加变换。

### 图像类：ImageSpatialObject

ImageSpatialObject就是对itk::Image进行了一层封装，然后加上了其在空间转的变换信息，以及父子层级关系。

### 图像掩膜:ImageMaskSpatialObject

ImageMaskSpatialObject和ImageSpatialObject非常类似，惟一的区别在于IsInsideInWorldSpace()只在像素不为0的时候才返回true。即对于背景区域，不认为是mask物体的范围。


### 线:LineSpatialObject

定义N维空间中的一条线。它具有以下几个属性：
- position
- normals（与线垂直的向量），有N-1个（比如三位直线由2个normal），由itk::CovariantVector表示
- color
 
### mesh:MeshSpatialObject

包含以下成员：
- 一个指向对应的itk::Mesh的指针；
- 空间变换矩阵；
- 父子层级关系。

### 表面:SurfaceSpatialObject

由一系列的处于surface上的点(SurfacePointType)组成。每个点具有位置（PointType）、法向量(CovariantVector)。

### 管状:TubeSpatialObject

常用来表示血管、器官、神经、胆管等结构。

它由一系列中心线上的点(TubePointType)组成，每个点包含:
- 位置（Point
- 半径
- 法向量(CovariantVector)：3维管状结构有两个normal；通过SetNormal1InObjectSpace和SetNormal2InObjectSpace函数设置。
- 其他特性。

### DTI管状:DTITubeSpatialObject

用来特别的表示使用DTI数据重建得到的神经纤维束管状结构的空间物体类。派生自itk::TubeSpatialObject

## 空间物体的读写

TO DO...

## 统计计算

TO DO...

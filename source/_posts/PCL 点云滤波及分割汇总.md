---
title: PCL 点云滤波及分割汇总
date: 2023-02-27 12:09:05
tags:
    - PCL
    - 点云
    - 机器视觉
    - 算法
    - 滤波
    - 分割
---
## 点云滤波汇总
### 立方体滤波
首先是最简单的容易理解的立方体的剪裁滤波，根据对角点确定一个立方体然后获取立方体内或外的点，并且还可以在这立方体基础上加平移或旋转，因为根据两个点确立的立方体只是坐标轴正方向的，不一定能满足所有需求。
``` C++
#include <pcl/filters/crop_box.h>
/// <summary>
/// 立方体滤波
/// </summary>
/// <param name="cloud_in">输入点云</param>
/// <param name="cloud_out">输出点云</param>
/// <param name="min">最小点位 x y z 1</param>
/// <param name="max">最大点位 x y z 1</param>
/// <param name="negative">默认值为 false，输出点云为在设定字段的设定范围内的点集，如果设置为 true 则刚好相反</param>
void cropFilter(pcl::PointCloud<pcl::PointXYZ>::Ptr cloud_in, pcl::PointCloud<pcl::PointXYZ>::Ptr &cloud_out, Eigen::Vector4f min, Eigen::Vector4f max, bool negative) {
    pcl::CropBox<pcl::PointXYZ> box_filter;						//滤波器对象
    box_filter.setMin(min);	//Min和Max是指立方体的两个对角点。每个点由一个四维向量表示，通常最后一个是1.
    box_filter.setMax(max);
    //box_filter.setRotation(rotation); //旋转 rx ry rz
    //box_filter.setTranslation(tanslation);//仿射矩阵 Affine3f 旋转加平移
    //box_filter.setTransform(transformax);//平移 tx ty tz
    box_filter.setNegative(negative);
    box_filter.setInputCloud(cloud_in);
    box_filter.filter(*cloud_out);
}
```
<!-- more -->
### 凸(凹)包滤波
这个滤波实际用下来效果就是把凸出来或凹进去的一块区域的点云滤出来，入参其实输不输入都行就是维度，默认不输入就是按点云情况来，`pcl::ConvexHull`是用来算凸包的， `pcl:: ConcaveHull`用来算凹包，最后都扔到`pcl::CropHull`里面。
```C++
#include <pcl/surface/concave_hull.h>
#include <pcl/filters/crop_hull.h>

/// <summary>
/// 凸包滤波
/// </summary>
/// <param name="cloud_in">输入点云</param>
/// <param name="cloud_out">输出点云</param>
/// <param name="boundingbox">封闭区域顶点</param>
/// <param name="dimension">设置凸包维度</param>
/// <param name="negative">默认值为 false，输出点云为在设定字段的设定范围内的点集，如果设置为 true 则刚好相反</param>
/// <param name="keepOrganized"> false=删除点（默认），true=重新定义点，保留结构</param>
void cropHullFilter(pcl::PointCloud<pcl::PointXYZ>::Ptr cloud_in, pcl::PointCloud<pcl::PointXYZ>::Ptr& cloud_out, pcl::PointCloud<pcl::PointXYZ>::Ptr boundingbox, int dimension, bool negative, bool keepOrganized) {
    pcl::PointCloud<pcl::PointXYZ>::Ptr surface_hull(new pcl::PointCloud<pcl::PointXYZ>);	//描述凸包形状的点云
    std::vector<pcl::Vertices> polygons;// polygons保存的是所有凸包多边形的顶点在surface_hull中的下标

    pcl::ConvexHull<pcl::PointXYZ> hull;
    hull.setInputCloud(boundingbox);//设置输入点云：封闭区域顶点点云
    hull.setDimension(dimension);//设置凸包维度

    //surface_hull存储最终产生的凹多边形上的顶点，polygons存储一系列顶点集,每个顶点集构成的一个多边形,Vertices数据结构包含一组点的索引,索引是point中的点对应的索引。
    hull.reconstruct(*surface_hull, polygons);

    pcl::CropHull<pcl::PointXYZ> CH;//创建CropHull滤波对象
    CH.setDim(dimension);	//设置维度，与凸包维度一致
    CH.setInputCloud(cloud_in);		//设置需要滤波的点云
    CH.setHullIndices(polygons);	//输入封闭区域的顶点
    CH.setHullCloud(surface_hull);	//输入封闭区域的形状
    CH.setNegative(negative);
    CH.setKeepOrganized(keepOrganized);
    CH.filter(*cloud_out);
}
```
### 体素采样滤波
体素滤波可以按照给定长宽高的立方体来填充点云，每个立方体的质心点留下来以此让减少点云中点的数目，有效的提升后续点云处理速度，一般用来下采样。
```C++
#include <pcl/filters/voxel_grid.h>

/// <summary>
/// 体素采样滤波
/// </summary>
/// <param name="cloud_in">输入点云</param>
/// <param name="cloud_out">输出点云</param>
/// <param name="lx">立方体X轴长度</param>
/// <param name="ly">立方体Y轴长度</param>
/// <param name="lz">立方体Z轴长度</param>
void voxelFilter(pcl::PointCloud<pcl::PointXYZ>::Ptr cloud_in, pcl::PointCloud<pcl::PointXYZ>::Ptr &cloud_out, float lx, float ly, float lz) {
    std::cout << "size before voxelFilter: " << cloud_in->size() << std::endl;
    pcl::VoxelGrid<pcl::PointXYZ> vox_grid;
    vox_grid.setLeafSize(lx, ly, lz);//设置滤波时创建的体素大小为lx m*ly m*lz m的立方体
    vox_grid.setInputCloud(cloud_in);
    vox_grid.filter(*cloud_out);
    std::cout << "size after voxelFilter: " << cloud_out->size() << std::endl;
}
```
### 近似体素重心采样滤波
`ApproximateVoxelGrid`近似体素滤波企图以 **更快的速度** 实现与`VoxelGrid` 体素滤波相同的下采样，它通过 **散列函数**（Hashing Function）**快速逼近质心**，而不是精细确定质心并对点云进行下采样。采样结果是近似逼近的体素质心，并不是体素中心。
```C++
#include <pcl/filters/approximate_voxel_grid.h>

/// <summary>
/// 近似体素重心采样滤波
/// </summary>
/// <param name="cloud_in">输入点云</param>
/// <param name="cloud_out">输出点云</param>
/// <param name="lx">立方体X轴长度</param>
/// <param name="ly">立方体Y轴长度</param>
/// <param name="lz">立方体Z轴长度</param>
/// <param name="downsampleAllData">如果只有XYZ字段，则设置为false，如果对所有字段，如intensity，都进行下采样，则设置为true</param>
void approximateVoxelGridFilter(pcl::PointCloud<pcl::PointXYZ>::Ptr cloud_in, pcl::PointCloud<pcl::PointXYZ>::Ptr& cloud_out, float lx, float ly, float lz, bool downsampleAllData) {
    std::cout << "size before voxelFilter: " << cloud_in->size() << std::endl;
    pcl::ApproximateVoxelGrid<pcl::PointXYZ> vox_grid;
    vox_grid.setLeafSize(lx, ly, lz);//设置滤波时创建的体素大小为lx m*ly m*lz m的立方体
    vox_grid.setInputCloud(cloud_in);
    vox_grid.setDownsampleAllData(downsampleAllData);
    vox_grid.filter(*cloud_out);
    std::cout << "size after voxelFilter: " << cloud_out->size() << std::endl;
}
```
### 均匀采样滤波
均匀采样滤波原理和体素滤波是很相似的，但是体素采用单位立方体，而均匀采样取给定半径的球体内的点保留质心。
```C++
#include <pcl/filters/uniform_sampling.h>

/// <summary>
/// 均匀采样滤波
/// </summary>
/// <param name="cloud_in">输入点云</param>
/// <param name="cloud_out">输出点云</param>
/// <param name="radius_seach">半径范围内搜寻邻居点</param>
void uniformSamplingFilter(pcl::PointCloud<pcl::PointXYZ>::Ptr cloud_in, pcl::PointCloud<pcl::PointXYZ>::Ptr& cloud_out, double radius_seach) {
    pcl::UniformSampling<pcl::PointXYZ> filter;
    filter.setRadiusSearch(radius_seach);
    filter.setInputCloud(cloud_in);
    filter.filter(*cloud_out);
}
```
### 随机采样滤波
随机选择n个点，每个点被选到的概率相同，可以得到固定的点数量；
缺点：如果原点云是不均匀的，比如有的地方密度大，有的地方密度小，那么RandomSample采样后的点云同样是不均匀的，分布同云点云。虽然每个点被选到的概率一样，但密度大的区域内点比较多，这个区域内被采样的机会就更多，因此原来密度大的地方，采样后，密度还是大。
```C++
#include <pcl/filters/random_sample.h>

/// <summary>
/// 随机采样滤波
/// </summary>
/// <param name="cloud_in">输入点云</param>
/// <param name="cloud_out">输出点云</param>
/// <param name="sample">设置下采样点云的点数</param>
/// <param name="seed">随机函数种子点</param>
/// <param name="negative">默认值为 false，输出点云为在设定字段的设定范围内的点集，如果设置为 true 则刚好相反</param>
/// <param name="keepOrganized"> false=删除点（默认），true=重新定义点，保留结构</param>
void randomSamplingFilter(pcl::PointCloud<pcl::PointXYZ>::Ptr cloud_in, pcl::PointCloud<pcl::PointXYZ>::Ptr& cloud_out, unsigned int sample, unsigned int seed, bool negative, bool keepOrganized) {
    pcl::RandomSample<pcl::PointXYZ> filter;
    filter.setSample(sample);
    filter.setSeed(seed);
    filter.setNegative(negative);
    filter.setKeepOrganized(keepOrganized);
    filter.setInputCloud(cloud_in);
    filter.filter(*cloud_out);
}
```
### 移动最小二乘增采样滤波
增采样是一种表面重建方法，当你有比你想象的要少的点云数据时，增采样可以帮你恢复原有的表面（S），通过内插你目前拥有的点云数据，这是一个复杂的猜想假设的过程。所以构建的结果不会百分之一百准确，但有时它是一种可选择的方案。所以，在你的点云云进行下采样时，一定要保存一份原始数据！
```C++
#include <pcl/surface/mls.h>

/// <summary>
/// 移动最小二乘增采样滤波
/// </summary>
/// <param name="cloud_in">输入点云</param>
/// <param name="cloud_out">输出点云</param>
/// <param name="tree">提取搜索方法的树对象，一般用近邻的kdTree即可</param>
/// <param name="upsamplingMethod">Upsampling 采样的方法有 DISTINCT_CLOUD, RANDOM_UNIFORM_DENSITY</param>
/// <param name="searchRadius">设置搜索邻域的半径</param>
/// <param name="upsamplingRadius">采样的半径</param>
/// <param name="upsamplingStepSize">采样步数的大小</param>
void movingLeastSquaresFilter(pcl::PointCloud<pcl::PointXYZ>::Ptr cloud_in, pcl::PointCloud<pcl::PointXYZ>::Ptr& cloud_out, pcl::search::Search<pcl::PointXYZ>::Ptr tree, pcl::MovingLeastSquares<pcl::PointXYZ, pcl::PointXYZ>::UpsamplingMethod upsamplingMethod, double searchRadius, double upsamplingRadius, double upsamplingStepSize) {
    pcl::MovingLeastSquares<pcl::PointXYZ, pcl::PointXYZ> filter;
    filter.setInputCloud(cloud_in);
    filter.setSearchMethod(tree);
    //设置搜索邻域的半径
    filter.setSearchRadius(searchRadius);
    // Upsampling 采样的方法有 DISTINCT_CLOUD, RANDOM_UNIFORM_DENSITY
    filter.setUpsamplingMethod(upsamplingMethod);
    // 采样的半径
    filter.setUpsamplingRadius(upsamplingRadius);
    // 采样步数的大小
    filter.setUpsamplingStepSize(upsamplingStepSize);
    filter.process(*cloud_out);
}
```
深度传感器的测量是不准确的，和由此产生的点云也是存在的测量误差，比如离群点，孔等表面，可以用一个算法重建表面，遍历所有的点云和插值数据，试图重建原来的表面。
### 统计滤波
统计滤波器用于去除明显离群点（离群点往往由测量噪声引入）。
其特征是在空间中分布稀疏，可以理解为：每个点都表达一定信息量，某个区域点越密集则可能信息量越大。噪声信息属于无用信息，信息量较小。所以离群点表达的信息可以忽略不计。考虑到离群点的特征，则可以定义某处点云小于某个密度，既点云无效。计算每个点到其最近的k(设定)个点平均距离。则点云中所有点的距离应构成高斯分布。给定均值与方差，可剔除ｎ个西格玛之外的点

激光扫描通常会产生密度不均匀的点云数据集，另外测量中的误差也会产生稀疏的离群点，此时，估计局部点云特征（例如采样点处法向量或曲率变化率）时运算复杂，这会导致错误的数值，反过来就会导致点云配准等后期的处理失败。
解决办法：对每个点的邻域进行一个统计分析，并修剪掉一些不符合标准的点。
具体方法为在输入数据中对点到临近点的距离分布的计算，对每一个点，
计算它到所有临近点的平均距离（假设得到的结果是一个高斯分布，
其形状是由均值和标准差决定），那么平均距离在标准范围之外的点，
可以被定义为离群点并从数据中去除。
```C++
#include <pcl/filters/statistical_outlier_removal.h>

/// <summary>
/// 统计滤波
/// </summary>
/// <param name="cloud_in">输入点云</param>
/// <param name="cloud_out">输出点云</param>
/// <param name="meanK">K近邻搜索点个数</param>
/// <param name="std_dev_mul">标准差倍数</param>
/// <param name="negative">默认值为 false，输出点云为在设定字段的设定范围内的点集，如果设置为 true 则刚好相反</param>
/// <param name="keepOrganized"> false=删除点（默认），true=重新定义点，保留结构</param>
void sorFilter(pcl::PointCloud<pcl::PointXYZ>::Ptr cloud_in, pcl::PointCloud<pcl::PointXYZ>::Ptr &cloud_out, int meanK, float std_dev_mul,bool negative, bool keepOrganized) {
    std::cout << "size before statisticalOutlierRemoval: " << cloud_in->size() << std::endl;
    pcl::StatisticalOutlierRemoval<pcl::PointXYZ> filter(true);
    filter.setInputCloud(cloud_in);
    filter.setMeanK(meanK);
    filter.setStddevMulThresh(std_dev_mul);
    filter.setNegative(negative);
    filter.setKeepOrganized(keepOrganized);
    filter.filter(*cloud_out);
    std::cout << "size after statisticalOutlierRemoval: " << cloud_out->size() << std::endl;
}
```
### 半径滤波
球半径滤波器与统计滤波器相比更加简单粗暴。
以某点为中心　画一个球计算落在该球内的点的数量，当数量大于给定值时，
则保留该点，数量小于给定值则剔除该点。
此算法运行速度快，依序迭代留下的点一定是最密集的，
但是球的半径和球内点的数目都需要人工指定。
```C++
#include <pcl/filters/radius_outlier_removal.h>

/// <summary>
/// 半径滤波
/// </summary>
/// <param name="cloud_in">输入点云</param>
/// <param name="cloud_out">输出点云</param>
/// <param name="radiusSearch">半径范围内搜寻邻居点</param>
/// <param name="minNeighborsInRadius">邻居少于点数认为是离群点</param>
/// <param name="negative">默认值为 false，输出点云为在设定字段的设定范围内的点集，如果设置为 true 则刚好相反</param>
/// <param name="keepOrganized"> false=删除点（默认），true=重新定义点，保留结构</param>
void rorFilter(pcl::PointCloud<pcl::PointXYZ>::Ptr cloud_in, pcl::PointCloud<pcl::PointXYZ>::Ptr &cloud_out, int radiusSearch, float minNeighborsInRadius, bool negative, bool keepOrganized) {
    std::cout << "size before radiusOutlierRemoval: " << cloud_in->size() << std::endl;

    pcl::RadiusOutlierRemoval<pcl::PointXYZ> ror;
    ror.setInputCloud(cloud_in);
    ror.setRadiusSearch(radiusSearch);
    ror.setMinNeighborsInRadius(minNeighborsInRadius);
    ror.setNegative(negative);
    ror.setKeepOrganized(keepOrganized);
    ror.filter(*cloud_out);

    std::cout << "size after radiusOutlierRemoval: " << cloud_out->size() << std::endl;

}
```
### 直通滤波
直通滤波是直接根据某一个轴的最大最小值，直接过滤出在这范围里的点云
```C++
#include <pcl/filters/passthrough.h>

/// <summary>
/// 直通滤波
/// </summary>
/// <param name="cloud_in">输入点云</param>
/// <param name="cloud_out">输出点云</param>
/// <param name="field_name">坐标轴名称:x ,y ,z</param>
/// <param name="limit_min">最大值</param>
/// <param name="limit_max">最小值</param>
/// <param name="negative">默认值为 false，输出点云为在设定字段的设定范围内的点集，如果设置为 true 则刚好相反</param>
void passThroughFilter(pcl::PointCloud<pcl::PointXYZ>::Ptr cloud_in, pcl::PointCloud<pcl::PointXYZ>::Ptr &cloud_out, std::string field_name, float limit_min, float limit_max, bool negative) {
    pcl::PassThrough<pcl::PointXYZ> pass;
    pass.setInputCloud(cloud_in);            //设置输入点云
    pass.setFilterFieldName(field_name);         //设置过滤时所需要点云类型的字符串字段 "x" "y" "z"
    pass.setFilterLimits(limit_min, limit_max);        //设置在过滤字段的范围
    pass.setFilterLimitsNegative (negative);   //设置保留范围内还是过滤掉范围内
    pass.filter(*cloud_out);            //执行滤波，保存过滤结果在cloud_out

}
```
### 条件滤波
条件滤波相比直通滤波更加灵活的设置各种值范围过滤点云，用`pcl::ConditionAnd`创建与条件，用`pcl::ConditionOr`创建或条件，最后扔到`pcl::ConditionalRemoval`里。
```C++
#include <pcl/filters/conditional_removal.h>

/// <summary>
/// 条件滤波
/// </summary>
/// <param name="cloud_in">输入点云</param>
/// <param name="cloud_out">输出点云</param>
/// <param name="range_cond">条件范围</param>
/// <param name="keepOrganized">false=删除点（默认），true=重新定义点，保留结构</param>
void conditionalRemovalFilter(pcl::PointCloud<pcl::PointXYZ>::Ptr cloud_in, pcl::PointCloud<pcl::PointXYZ>::Ptr& cloud_out, ConditionBaseT::Ptr condition, bool keepOrganized) {
    //pcl::ConditionAnd<pcl::PointXYZ>::Ptr condition(new pcl::ConditionAnd<pcl::PointXYZ>());//创建与条件
    //pcl::ConditionOr表示或条件
    //condition->addComparison(pcl::FieldComparison<pcl::PointXYZ>::ConstPtr(new
    //	pcl::FieldComparison<pcl::PointXYZ>("z", pcl::ComparisonOps::GT, 0.0)));
    //condition->addComparison(pcl::FieldComparison<pcl::PointXYZ>::ConstPtr(new
    //	pcl::FieldComparison<pcl::PointXYZ>("z", pcl::ComparisonOps::LT, 0.8)));
    //GT表示大于等于，LT表示小于等于，EQ表示等于，GE表示大于，LE表示小于
    pcl::ConditionalRemoval<pcl::PointXYZ> condrem;
    condrem.setCondition(condition);
    condrem.setInputCloud(cloud_in);
    condrem.setKeepOrganized(keepOrganized);
    condrem.filter(*cloud_out);
}
```
### 投影滤波
投影滤波比较高级用的比较多的是向一个平面投影，很多时候把3维的数据降维到2维很多复杂的处理都能因此变得简单，标准的降维打击，设置`ModelType`类型为`pcl::SACMODEL_PLANE`,按照平面方程公式ax+by+cz+d=0，把a,b,c,d值依次放入pcl::ModelCoefficients类型对象中即可。
```C++
#include <pcl/filters/project_inliers.h>
#include <pcl/ModelCoefficients.h>

/// <summary>
/// 投影滤波
/// </summary>
/// <param name="cloud_in">输入点云</param>
/// <param name="cloud_out">输出点云</param>
/// <param name="coefficients">设置模型对应的系数,类似数组提供coefficients->values.resize(n)定义大小,coefficients->values[0]=赋值,平面的话值对应ax+by+cz+d=0</param>
/// <param name="modelType">设置对应的投影模型,平面用pcl::SACMODEL_PLANE</param>
void projectInliersFilter(pcl::PointCloud<pcl::PointXYZ>::Ptr cloud_in, pcl::PointCloud<pcl::PointXYZ>::Ptr &cloud_out, pcl::ModelCoefficients::Ptr coefficients, int modelType) {

    pcl::ProjectInliers<pcl::PointXYZ> proj;		//创建投影滤波对象
    proj.setModelType(modelType);			//设置对象对应的投影模型
    proj.setInputCloud(cloud_in);			//设置输入点云
    proj.setModelCoefficients(coefficients);		//设置模型对应的系数
    proj.filter(*cloud_out);	//执行投影滤波存储结果cloud_projected

}
```
### 双边滤波
双边滤波利用的并非XYZ字段的数据进行,而是利用强度数据字段进行双边滤波算法的实现,所以在使用该类时点云的类型中字段必须有强度字段,否则无法进行双边滤波处理。
```C++
#include <pcl/filters/bilateral.h>

/// <summary>
/// 双边滤波
/// </summary>
/// <param name="cloud_in">输入点云</param>
/// <param name="cloud_out">输出点云</param>
/// <param name="sigma_s">高斯双边滤波窗口大小</param>
/// <param name="sigma_r">高斯标准差表示强度差异</param>
/// <param name="tree">提取搜索方法的树对象，一般用近邻的kdTree即可</param>
void bilateralFilter(pcl::PointCloud<pcl::PointXYZI>::Ptr cloud_in, pcl::PointCloud<pcl::PointXYZI>::Ptr& cloud_out, float sigma_s, float sigma_r, pcl::search::Search<pcl::PointXYZI>::Ptr tree) {
    pcl::BilateralFilter<pcl::PointXYZI> bf;
    bf.setInputCloud(cloud_in);            //设置输入点云
    bf.setSearchMethod(tree);
    bf.setHalfSize(sigma_s);
    bf.setStdDev(sigma_r);
    bf.filter(*cloud_out);
}

#include <pcl/filters/fast_bilateral.h>

/// <summary>
/// 快速双边滤波
/// </summary>
/// <param name="cloud_in">输入点云</param>
/// <param name="cloud_out">输出点云</param>
/// <param name="sigma_s">高斯双边滤波窗口大小</param>
/// <param name="sigma_r">高斯标准差表示强度差异</param>
void fastBilateralFilter(pcl::PointCloud<pcl::PointXYZ>::Ptr cloud_in, pcl::PointCloud<pcl::PointXYZ>::Ptr& cloud_out, float sigma_s, float sigma_r) {
    pcl::FastBilateralFilter<pcl::PointXYZ> fbf;
    fbf.setInputCloud(cloud_in);            //设置输入点云
    fbf.setSigmaS(sigma_s);
    fbf.setSigmaR(sigma_r);
    fbf.filter(*cloud_out);
}
```
### 索引滤波
根据点云给定索引过滤出对应点云，一般不会单独用，都是配合一些特征提取完对应索引后来用。
```C++
#include <pcl/filters/extract_indices.h>

/// <summary>
/// 索引滤波
/// </summary>
/// <param name="cloud_in">输入点云</param>
/// <param name="cloud_out">输出点云</param>
/// <param name="indices">过滤的点云索引</param>
/// <param name="negative">默认值为 false，输出点云为在设定字段的设定范围内的点集，如果设置为 true 则刚好相反</param>
void extractIndicesFilter(pcl::PointCloud<pcl::PointXYZ>::Ptr cloud_in, pcl::PointCloud<pcl::PointXYZ>::Ptr &cloud_out, pcl::PointIndices::Ptr indices, bool negative) {

    pcl::ExtractIndices<pcl::PointXYZ> extract;
    extract.setInputCloud(cloud_in);
    extract.setIndices(indices);
    extract.setNegative(negative);
    extract.filter(*cloud_out);
}
```

### 总结
一般场景这些滤波组合能解决95%的应用场景，其他的滤波应用场景是在很少如`pcl::BoxClipper3D`用来空间剪裁也可以指定立方体，`pcl::filters::Convolution`是卷积滤波设置卷积核，`pcl::filters::GaussianKernel`和`pcl::filters::GaussianKernelRGB`是高斯卷积核滤波，基本其他这些在实际场景里很少用到

## 点云分割汇总
这里的分割结果基本为索引再配合上问的索引滤波可以提取出对应点云。
### 欧式距离分割
欧式距离分割是基于欧氏距离进行聚类分割的类，这里注意这个类` pcl::gpu::EuclideanClusterExtraction
`这么写会在gpu上跑，而` pcl::EuclideanClusterExtraction
`这么写则会在cpu上，这类写法同样适用pcl内其他的类，另外还有一些别的欧式聚类的类看名称就能猜到用途，如`LabeledEuclideanClusterExtraction`就是多了一个`setMaxLabels`设置点云标签的最大数量。

这个方法用的也算频繁的，大部分场景 下采样-欧式聚类-投影，这时候就直接变二维图像了，opencv提取个轮廓基本工业场景的视觉要做的就完成了很大一部分。
```C++
#include <pcl/segmentation/extract_clusters.h>

/// <summary>
/// 欧式距离分割
/// </summary>
/// <param name="cloud_in">输入点云</param>
/// <param name="cluster_indices_out">输出索引</param>
/// <param name="tolerance">近邻搜索的搜索半径 m</param>
/// <param name="min_cluster_size">一个聚类需要的最少点数目</param>
/// <param name="max_cluster_size">一个聚类需要的最大点数目</param>
void euclideanClusterExtraction(pcl::PointCloud<pcl::PointXYZ>::Ptr cloud_in, std::vector<pcl::PointIndices> &cluster_indices_out, double tolerance, int min_cluster_size, int max_cluster_size) {
    // 创建用于提取搜索方法的kdtree树对象
    pcl::search::KdTree<pcl::PointXYZ>::Ptr tree(new pcl::search::KdTree<pcl::PointXYZ>);
    tree->setInputCloud(cloud_in);

    pcl::EuclideanClusterExtraction<pcl::PointXYZ> ec;		//欧式聚类对象
    ec.setClusterTolerance(tolerance);						// 设置近邻搜索的搜索半径为tolerance m
    ec.setMinClusterSize(min_cluster_size);						//设置一个聚类需要的最少的点数目为100
    ec.setMaxClusterSize(max_cluster_size);					//设置一个聚类需要的最大点数目为200000
    ec.setSearchMethod(tree);						//设置点云的搜索机制
    ec.setInputCloud(cloud_in);
    ec.extract(cluster_indices_out);					//从点云中提取聚类，并将点云索引保存在cluster_indices中
}
```
### 采样一致性分割
采样一致性分割算法的目的主要是从原点云中提取目标模型，比如说面，球体，圆柱等等，从而为后续的目标识别或者点云匹配等等做准备。这里只详细讲比较有代表性的`pcl::SACSegmentation`,这一类还有`SACSegmentationFromNormals`其在算法实现时采用了法线信息，即该类在进行运算输出之前需要设定法线信息。

##### 采样一致性算法
###### 1. 随机采样一致性算法(SAC_RANSAC)
`pcl::RandomSampleConsensus< PointT > Class Template Reference`
RandomSampleConsensus 表示随机采样一致性算法的实现，该算法的工作原理如下：
从云中随机选择样本，数量与确定模型所需的数量一样多；
从样本中计算模型的系数；
在给定阈值的情况下，计算云中有多少点属于模型。 这些被称为内点；
重复直到找到一个好的模型或达到最大迭代次数，返回具有最多内点的模型。

###### 2. 最小平方中值算法(SAC_LMEDS)
`pcl::LeastMedianSquares< PointT > Class Template Reference`
LeastMedianSquares 为最小平方中值算法的实现。LMedS 是一种类似于 RANSAC 的模型拟合算法，可以容忍高达 50% 的异常值，而无需设置阈值。与 RANSAC 相比，LMedS 在查找模型时不会将点分为内点和异常点。 相反，它使用所有点模型距离的中值作为模型好坏的衡量标准。 只有在最后确定哪些点属于找到的模型时才需要一个阈值。

###### 3. M-估计量样本一致性算法(SAC_MSAC)
`pcl::MEstimatorSampleConsensus< PointT > Class Template Reference`
MEstimatorSampleConsensus 表示 M-估计量样本一致性算法的实现。RANSAC 在给定阈值的情况下计算内点数。 内点越多，模型越好——内点与模型的实际距离有多近并不重要，只要它们在阈值内即可。 MSAC 通过使用所有点模型距离的总和作为质量度量来改变这一点，但是异常值仅添加阈值而不是它们的真实距离。 与 RANSAC 相比，这种方法可以产生更好的结果。

###### 4. 快点的随机采样一致性算法(SAC_RRANSAC)
`pcl::RandomizedRandomSampleConsensus< PointT > Class Template Reference`
RandomizedRandomSampleConsensus 表示 RRANSAC的实现，该算法的工作原理与 RANSAC 类似，但有一个补充：计算模型系数后，随机选择一小部分点；如果这些点中的任何一个点在给定的阈值内不属于拟合模型，则继续下一次迭代，而不是检查所有点。 如果预先测试的分数选择得当，则可能会加快模型的拟合速度。

###### 5. 随机M-估计量样本一致性(SAC_RMSAC)
`pcl::RandomizedMEstimatorSampleConsensus< PointT > Class Template Reference`
RandomizedMEstimatorSampleConsensus 表示 随机M-估计量样本一致性算法的实现，RMSAC 在大部分样本数据属于模型的情况下很有用。是MEstimatorSampleConsensus和RandomizedRandomSampleConsensus的综合版本，首先随机选取一些采样点，如果这些点中的任何一个点在给定阈值范围内不属于待拟合的模型，则继续下一次迭代，而不是检查所有点。同时使用所有点模型距离的总和作为质量度量来改变这一点，异常值仅添加阈值而不是它们的真实距离。

###### 6. 最大似然估计样本一致性算法(SAC_MLESAC)
`pcl::MaximumLikelihoodSampleConsensus< PointT > Class Template Reference`
MaximumLikelihoodSampleConsensus 表示 最大似然估计样本一致性算法的实现。通过最小化概率损失作为评价指标。使用由内点和外点产生的误差的概率模型来评估模型的适应性，选取模型适应性最高的模型参数作为最终结果。

###### 7. 渐进样本一致性算法(SAC_PROSAC)
`pcl::ProgressiveSampleConsensus< PointT > Class Template Reference`
ProgressiveSampleConsensus 表示 渐进样本一致性算法的实现，渐进一致性算法 (PROSAC) 是对经典 RANSAC采样的一种优化。相比经典 RANSAC 方法均匀地从整个样本集合中采样，PROSAC 不是从所有数据点中进行随机采样，而是先对数据点进行排序，然后在评价函数值最高的数据点子集中进行随机采样。该方法从采样点的选取方式入手，选择评价函数值最高的数据进行随机采样。所以这种方法可以节省计算量，提高运行速度。

##### 采样一致性的几何模型的类型
###### SACMODEL_PLANE模型
定义为平面模型,共设置4个参数[ normal_x, normal_y, normal_z d],其中(normal_x, normal_y, normal_z)为Hessian 范式中法向量的坐标及常量d值,ax+by+cz+d=0,从点云中分割提取的内点都处在估计参数对应的平面上或与平面距离在一定范围内。

###### SACMODEL_LINE模型
定义为直线模型,共设置6个参数[point_on_line.x, point_on_line.y, point_on_line.z, line_direction.x, line_direction.y, line_direction.z],其中(point_on_line.x,point_on_line.y,point_on_line.z)为直线上一点的三维坐标，(line_direction.x, line_direction.y, line_direction.z)为直线方向向量的三维坐标，从点云中分割提取的内点都处在估计参数对应直线上或与直线的距离在一定范围内。

###### SACMODEL_CIRCLE2D模型
定义为二维圆的圆周模型,共设置3个参数[center.x, center.y, radius],其中(center.x, center.y)为圆周中心点的二维坐标,radius为圆周半径,从点云中分割提取的内点都处在估计参数对应的圆周上或距离圆周边线的距离在一定范围内。

###### SACMODEL_SPHERE模型
定义为三维球体模型,共设置4个参数[center.x, center.y, center.z radius],其中(center. x, center. y, center. z)为球体中心的三维坐标,radius为球体半径,从点云中分割提取的内点都处在估计参数对应的球体上或距离球体边线的距离在一定范围内。

###### SACMODEL_NORMAL_SPHERE模型
定义为有条件限制的三维球体模型,参数设置参见SampleConsensusModelNormalSphere模型。

###### SACMODEL_CYLINDER模型
定义为圆柱体模型,共设置7个参数[point_on_axis.x, point_on_axis.y, point_on_axis.z, axis_direction.x ,axis_direction.y ,axis_diection.z, radius],其中,(point_on_axis.x, point_on_axis.y, point_on_axis.z)为轴线上点的三维坐标,(direction.x ,axis_direction.y, axis_direction.z)为轴线方向向量的三维坐标,radius为圆柱体半径，从点云中分割提取的内点都处在估计参数对应的圆柱体上或距离圆柱体表面的距离在一定范围内。

###### SACMODEL_CONE模型
定义为圆锥模型，共设置7个参数[apex.x, apex.y, apex.z, axis_direction.x, axis_direction.y, axis_direction.z, opening_angle],其中(apex.x, apex.y, apex.z)圆锥顶点,(axis_direction.x, axis_direction.y, axis_direction.z)为圆锥轴方向的三维坐标,opening_angle为圆锥的开口角度
		
###### SACMODEL_TORUS模型
定义为圆环面模型,尚未实现。

###### SACMODEL_PARALLEL_LINE 模型
定义为有条件限制的直线模型,在规定的最大角度偏差限制下,直线模型与给定轴线平行,共设置6个参数[point_on_line.x, point_on_line.y, point_on_line.z, line_direction.x, line_direction.y, line_direction.z],其中(point_on_line.x, point_on_line.y, point_on_line.z)为线上一个点的三维坐标,(line_direction.x, line_direction.y, line_direction.z)为直线方向的三维坐标。

###### SACMODEL_PERPENDICULAR_PLANE模型
定义为有条件限制的平面模型,在规定的最大角度偏差限制下,平面模型与给定轴线垂直,参数设置参见SAC-MODEL_PLANE模型。

###### SACMODEL_NORMAL_PLANE模型
定义为有条件限制的平面模型,在规定的最大角度偏差限制下,每一个局内点的法线必须与估计的平面模型的法线平行,参数设置参见SACMODEL_PLANE模型。

###### SACMODEL_PARALLEL_PLANE模型
定义为有条件限制的平面模型,在规定的最大角度偏差限制下,平面模型与给定的轴线平行,参数设置参见SACMODEL_PLANE模型。

###### SACMODEL_NORMAL_PARALLEL_PLANE模型
定义为有条件限制的平面模型,在法线约束下，三维平面模型必须与用户设定的轴线平行,共4个参数[a,b,c,d],其中a是平面法线的 X 坐标（归一化）,b是平面法线的 Y 坐标（归一化）,c是平面法线的 Z 坐标（归一化）,d是平面方程的第四个 Hessian 分量。

###### SACMODEL_STICK模型
定义棒是用户给定最小/最大宽度的线。共7个参数[point_on_line.x, point_on_line.y, point_on_line.z, line_direction.x, line_direction.y, line_direction.z, line_width],其中(point_on_line.x, point_on_line.y, point_on_line.z)为线上一个点的三维坐标,(line_direction.x, line_direction.y, line_direction.z)为直线方向的三维坐标,line_width为线的宽度。

#### 具体实现
```C++
#include <pcl/segmentation/sac_segmentation.h>

/// <summary>
/// 一致性分割
/// </summary>
/// <param name="cloud_in">输入点云</param>
/// <param name="cluster_indices_out">输出索引</param>
/// <param name="coefficients_out">输出模型</param>
/// <param name="methodType">采样一致性方法的类型</param>
/// <param name="modelType">采样一致性所构造的几何模型的类型</param>
/// <param name="distance">点到模型的距离阈值</param>
/// <param name="max_iterations">迭代次数的上限</param>
/// <param name="probability">选取至少一个局内点的概率</param>
void sacSegmentationExtraction(pcl::PointCloud<pcl::PointXYZ>::Ptr cloud_in, int methodType, pcl::SacModel modelType, double distance, int max_iterations, double probability, pcl::PointIndices &cluster_indices_out, pcl::ModelCoefficients &coefficients_out) {
    pcl::SACSegmentation<pcl::PointXYZ> sac;		//一致性分割
    sac.setInputCloud(cloud_in);
    //设置使用采样一致性方法的类型(用户给定参数),采样一致性方法的类型有
    //SAC_RANSAC（随机采样一致性）、SAC_LMEDS（最小平方中值）、 SAC_MSAC（M估计采样一致性）、SAC_RRANSAC（随机采样一致性）、SAC_RMSAC（M估计采样一致性）、SAC_MLESAC（最大似然采样一致性）、SAC_PROSAC（渐进一致性采样）
    sac.setMethodType(methodType);
    //设置随机采样一致性所构造的几何模型的类型(用户给定的参数),model为指定的模型类型参数SACMODEL_PLANE,SACMODEL_LINE,SACMODEL_CIRCLE2D,SACMODEL_CIRCLE3D,SACMODEL_SPHERE,SACMODEL_CYLINDER,SACMODEL_CONE,SACMODEL_TORUS,SACMODEL_PARALLEL_LINE,SACMODEL_PERPENDICULAR_PLANE,SACMODEL_PARALLEL_LINES,SACMODEL_NORMAL_PLANE,SACMODEL_NORMAL_SPHERE,SACMODEL_REGISTRATION,SACMODEL_REGISTRATION_2D,SACMODEL_PARALLEL_PLANE,SACMODEL_NORMAL_PARALLEL_PLANE,SACMODEL_STICK
    sac.setModelType(modelType);
    //设置点到模型的距离阈值,如果点到模型的距离不超过这个距离阈值,认为该点为局内点,否则认为是局外点,被剔除。
    sac.setDistanceThreshold(distance);
    sac.setMaxIterations(max_iterations);	//设置迭代次数的上限
    sac.setProbability(probability);		//设置每次从数据集中选取至少一个局内点的概率
    
    ////设置是否对估计的模型参数进行优化
    //sac.setOptimizeCoefficients(optimize); 
    ////该函数配合,当用户设定带有半径参数的模型类型时，设置模型半径参数的最大最小半径阈值
    //sac.setRadiusLimits(min_radius, max_radius);
    ////该函数配合，当用户设定与轴线平行或垂直有关的模型类型时,设置垂直或平行于所要建立模型的轴线。
    //sac.setAxis(ax);
    ////该函数配合，当用户设定有平行或垂直限定有关的模型类型时，设置判断是否平行或垂直时的角度阈值,ea是最大角度差,采用弧度制。
    //sac.setEpsAngle(ea);
    
    //输出最终估计的模型参数，以及分割得到的点集合索引
    sac.segment(cluster_indices_out, coefficients_out);
}
```
### 凸包分割
通过设置一组处于同一平面模型上的点索引向量，并指定一高度，利用指定的点形成二维凸包，再结合指定高度一起生成一个立体多边形棱柱模型，该类用于分割出该棱柱模型内部的点集，比如分割出放在平面上的物体。
```C++
#include <pcl/segmentation/extract_polygonal_prism_data.h>
/// <summary>
/// 凸包分割
/// </summary>
/// <param name="cloud_in">输入点云</param>
/// <param name="cluster_indices_out">输出索引</param>
/// <param name="hull_in">凸包点云</param>
/// <param name="height_min">最大高度</param>
/// <param name="height_max">最小高度</param>
/// <param name="vpx">视点坐标x</param>
/// <param name="vpy">视点坐标x</param>
/// <param name="vpz">视点坐标z</param>
void extractPolygonalPrismDataExtraction(pcl::PointCloud<pcl::PointXYZ>::Ptr cloud_in, pcl::PointCloud<pcl::PointXYZ>::Ptr hull_in, pcl::PointIndices& cluster_indices_out, double height_min, double height_max, float vpx, float vpy, float vpz) {
    pcl::ExtractPolygonalPrismData<pcl::PointXYZ> eppd;
    //设置高度范围(height_max为最大高度、height_min为最小高度),当给定点云中的点到构造棱柱体的平面模型的距离超出指定的这个高度范围时,该点视为局外点,所有的局外点都会被剔除。
    eppd.setHeightLimits(height_min, height_max);
    eppd.setViewPoint(vpx, vpy, vpz);		//设置视点,vpx,vpy,vpz分别为视点的三维坐标。
    eppd.setInputCloud(cloud_in);
    eppd.setInputPlanarHull(hull_in);		//设置平面模型上的点集,hull为指向该点集的指针
    eppd.segment(cluster_indices_out);		//从点云中提取聚类，并将点云索引保存在cluster_indices中
}
```
### 其他不常用的
`pcl::SeededHueSegmentation`是基于颜色信息的点云区域生长分割算法，该算法在分割时不仅使用了点云的空间信息还使用了点云所带的可见光信息，非常适合对基于RGBD设备获取的点云进行分割处理。
`pcl::SegmentDifferences`可以得到在对应点最大偏差距离限制下的两个配准后点云的差异，此算法输入为两组已经配准的点云，设置两组点云对应点之间的偏差距离阈值，最后返回两组点云的差。
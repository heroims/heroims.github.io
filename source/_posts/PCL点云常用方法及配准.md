---
title: PCL 点云常用方法及配准
date: 2023-02-28 12:09:05
tags:
    - PCL
    - 点云
    - 机器视觉
    - 算法
    - 配准
    - 压缩
    - 轮廓
    - 特征
---
## 点云配准汇总
配准比较常用的套路就是法线提取，然后fpfh特征提取，粗配准，最后再来精配准，基本就能得到比较准确的旋转矩阵和模板点云配上。
### 特征提取
#### 轮廓提取
```C++
#include <pcl/features/boundary.h>

/// <summary>
/// 轮廓计算
/// </summary>
/// <param name="cloud_in">输入点云</param>
/// <param name="normals_in">输入法线</param>
/// <param name="radius">半径</param>
/// <param name="angle">角度</param>
/// <param name="cloud_out">输出点云</param>
void boundaryEstimationFeatures(pcl::PointCloud<pcl::PointXYZ>::Ptr cloud_in, pcl::PointCloud<pcl::Normal>::Ptr normals_in, pcl::PointCloud<pcl::PointXYZ>::Ptr& cloud_out, double radius, float angle) {
    pcl::BoundaryEstimation<pcl::PointXYZ, pcl::Normal, pcl::Boundary> be;
    be.setInputCloud(cloud_in);
    be.setInputNormals(normals_in);
    pcl::search::KdTree<pcl::PointXYZ>::Ptr tree(new pcl::search::KdTree<pcl::PointXYZ>);
    tree->setInputCloud(cloud_in);
    //设置搜索时所用的球半径,参数radius为设置搜索球体半径
    be.setRadiusSearch(radius);
    //设置搜索时近邻的来源点云,参数cloud指向搜索时被搜索的点云对象,常常用稠密的原始点云,而不用稀疏的下采样后的点云,在特征描述和提取中查询点往往是关键点
    //be.setSearchSurface(overlap_cloud);
    //设置搜索时所用的k近邻个数,参数k为设置搜索近邻个数
    //be.setKSearch(k);

    //设置将点标记为边界或规则的决定边界(角度阈值)。
    be.setAngleThreshold(angle);
    pcl::PointCloud<pcl::Boundary>::Ptr boundarys(new pcl::PointCloud<pcl::Boundary>());
    be.compute(*boundarys);
    for (size_t i = 0; i < cloud_in->size(); i++)
    {
        if ((*boundarys)[i].boundary_point > 0) {
            cloud_out->points.push_back(cloud_in->points[i]);
        }
    }
}
```
#### 法线提取
```C++
#include <pcl/features/normal_3d.h>

/// <summary>
/// 法线计算
/// </summary>
/// <param name="cloud_in">输入点云</param>
/// <param name="radius">半径</param>
/// <param name="normals_out">输出点云</param>
void normalEstimationFeatures(pcl::PointCloud<pcl::PointXYZ>::Ptr cloud_in, double radius, pcl::PointCloud<pcl::Normal>::Ptr& normals_out) {
    pcl::NormalEstimation<pcl::PointXYZ, pcl::Normal> ne;
    ne.setInputCloud(cloud_in);
    pcl::search::KdTree<pcl::PointXYZ>::Ptr tree(new pcl::search::KdTree<pcl::PointXYZ>);
    tree->setInputCloud(cloud_in);
    ne.setSearchMethod(tree);
    //设置搜索时所用的球半径,参数radius为设置搜索球体半径
    ne.setRadiusSearch(radius);
    //设置搜索时近邻的来源点云,参数cloud指向搜索时被搜索的点云对象,常常用稠密的原始点云,而不用稀疏的下采样后的点云,在特征描述和提取中查询点往往是关键点
    //ne.setSearchSurface(overlap_cloud);
    //设置搜索时所用的k近邻个数,参数k为设置搜索近邻个数
    //ne.setKSearch(k);
    ne.compute(*normals_out);
}
```
#### FPFH特征提取
```C++
#include <pcl/features/fpfh.h>

/// <summary>
/// fpfh计算
/// </summary>
/// <param name="cloud_in">输入点云</param>
/// <param name="normals_in">输入法线</param>
/// <param name="radius">半径</param>
/// <param name="features_out">输出特征</param>
void fpfhEstimationFeatures(pcl::PointCloud<pcl::PointXYZ>::Ptr cloud_in, pcl::PointCloud<pcl::Normal>::Ptr normals_in, pcl::PointCloud<pcl::FPFHSignature33>::Ptr& features_out, double radius) {
    pcl::FPFHEstimation<pcl::PointXYZ, pcl::Normal, pcl::FPFHSignature33> fpfhe;
    fpfhe.setInputCloud(cloud_in);
    fpfhe.setInputNormals(normals_in);
    pcl::search::KdTree<pcl::PointXYZ>::Ptr tree(new pcl::search::KdTree<pcl::PointXYZ>);
    tree->setInputCloud(cloud_in);
    fpfhe.setSearchMethod(tree);

    //设置搜索时所用的球半径,参数radius为设置搜索球体半径
    fpfhe.setRadiusSearch(radius);
    //设置搜索时近邻的来源点云,参数cloud指向搜索时被搜索的点云对象,常常用稠密的原始点云,而不用稀疏的下采样后的点云,在特征描述和提取中查询点往往是关键点
    //fpfhe.setSearchSurface(overlap_cloud);
    //设置搜索时所用的k近邻个数,参数k为设置搜索近邻个数
    //fpfhe.setKSearch(k);
    
    //设置SPFH的统计区间个数。
    //fpfhe.setNrSubdivisions(nr_bins_f1, nr_bins_f2, nr_bins_f3);
    fpfhe.compute(*features_out);
}
```
### 粗配准
```C++
#include <pcl/registration/ia_ransac.h>

/// <summary>
/// 粗配准矩阵
/// </summary>
/// <param name="cloud_src_in">源点云</param>
/// <param name="cloud_tgt_in">目标点云</param>
/// <param name="fpfhs_src_in">源点云特征</param>
/// <param name="fpfhs_tgt_in">目标点云特征</param>
/// <param name="sac_trans_out">输出粗配准矩阵</param>
/// <param name="maximum_iterations">最大迭代数</param>
/// <param name="distance">样本间的最小距离</param>
/// <param name="number">使用的样本数量</param>
/// <param name="k">k邻</param>
void sacRegistration(
    pcl::PointCloud<pcl::PointXYZ>::Ptr cloud_src_in, 
    pcl::PointCloud<pcl::PointXYZ>::Ptr cloud_tgt_in, 
    pcl::PointCloud<pcl::FPFHSignature33>::Ptr fpfhs_src_in,
    pcl::PointCloud<pcl::FPFHSignature33>::Ptr fpfhs_tgt_in, 
    Eigen::Matrix4f& sac_trans_out,
    int maximum_iterations, 
    float distance,
    int number,
    int k) {
    pcl::SampleConsensusInitialAlignment<pcl::PointXYZ, pcl::PointXYZ, pcl::FPFHSignature33> scia;
    scia.setInputSource(cloud_src_in);
    scia.setInputTarget(cloud_tgt_in);
    scia.setSourceFeatures(fpfhs_src_in);
    scia.setTargetFeatures(fpfhs_tgt_in);
    scia.setMaximumIterations(maximum_iterations);// 最大迭代次数
    scia.setMinSampleDistance(distance);//设置样本间的最小距离
    scia.setNumberOfSamples(number);//设置每次迭代中使用的样本数量

    //设置计算协方差时选择附近多少个点为邻居:设置的点的数量k越高,协方差计算越准确但也会相应降低计算速度
    scia.setCorrespondenceRandomness(k);

    pcl::PointCloud<pcl::PointXYZ>::Ptr sac_result(new pcl::PointCloud<pcl::PointXYZ>);
    scia.align(*sac_result);
    std::cout << "sac has converged:" << scia.hasConverged() << " score: " << scia.getFitnessScore() << endl;
    sac_trans_out = scia.getFinalTransformation();
    std::cout << sac_trans_out << endl;
}
```
### 精配准
```C++
#include <pcl/registration/icp.h>
/// <summary>
/// 精配准
/// </summary>
/// <param name="cloud_src_in">源点云</param>
/// <param name="cloud_tgt_in">目标点云</param>
/// <param name="icp_trans_out">输出配准矩阵</param>
/// <param name="naximum_iterations">最大迭代数</param>
/// <param name="transformation_epsilon">两次变化矩阵之间的差值</param>
/// <param name="euclidean_fitness_epsilon">均方误差</param>
/// <param name="use_reciprocal">设置是否使用倒数对应</param>
/// <param name="sac_trans">粗配准矩阵</param>
void icpRegistration(
    pcl::PointCloud<pcl::PointXYZ>::Ptr cloud_src_in, 
    pcl::PointCloud<pcl::PointXYZ>::Ptr cloud_tgt_in, 
    Eigen::Matrix4f& icp_trans_out, 
    int naximum_iterations,
    double transformation_epsilon = 1e-10,
    double euclidean_fitness_epsilon = 0.2,
    bool use_reciprocal = false,
    Eigen::Matrix4f sac_trans = {}) {
    pcl::IterativeClosestPoint<pcl::PointXYZ, pcl::PointXYZ> icp;
    icp.setInputSource(cloud_src_in);
    icp.setInputTarget(cloud_tgt_in);

    //最大迭代次数
    icp.setMaximumIterations(naximum_iterations);
    //两次变化矩阵之间的差值
    icp.setTransformationEpsilon(transformation_epsilon);
    //均方误差
    icp.setEuclideanFitnessEpsilon(euclidean_fitness_epsilon);
    //设置是否使用倒数对应
    icp.setUseReciprocalCorrespondences(use_reciprocal);
    pcl::PointCloud<pcl::PointXYZ>::Ptr icp_result(new pcl::PointCloud<pcl::PointXYZ>);
    Eigen::Matrix4f sac_trans_null = {};
    if (sac_trans == sac_trans_null) {
        icp.align(*icp_result);
    }
    else
    {
        icp.align(*icp_result, sac_trans);
    }

    std::cout << "ICP has converged:" << icp.hasConverged()
        << " score: " << icp.getFitnessScore() << std::endl;

    icp_trans_out = icp.getFinalTransformation();
    std::cout << icp_trans_out << endl;
}
```
## 常用方法
### 求质心点
```C++
Eigen::Vector4f pcaCentroid;//质心点(4x1)
pcl::compute3DCentroid(cloud_in, pcaCentroid);
```
### PCA求主方向(转正)的变换矩阵
```C++
void getPCATransfrom(const pcl::PointCloud<pcl::PointXYZ> cloud_in, Eigen::Vector4f centroid, Eigen::Matrix4f &tm) {

    Eigen::Vector4f pcaCentroid = centroid;//质心点(4x1)

    Eigen::Matrix3f covariance;
    pcl::computeCovarianceMatrixNormalized(cloud_in, pcaCentroid, covariance);
    Eigen::SelfAdjointEigenSolver<Eigen::Matrix3f> eigen_solver(covariance, Eigen::ComputeEigenvectors);
    Eigen::Matrix3f eigenVectorsPCA = eigen_solver.eigenvectors();//特征向量ve(3x3)
    Eigen::Vector3f eigenValuesPCA = eigen_solver.eigenvalues();//特征值va(3x1)
    eigenVectorsPCA.col(2) = eigenVectorsPCA.col(0).cross(eigenVectorsPCA.col(1)); //校正主方向间垂直
    eigenVectorsPCA.col(0) = eigenVectorsPCA.col(1).cross(eigenVectorsPCA.col(2));
    eigenVectorsPCA.col(1) = eigenVectorsPCA.col(2).cross(eigenVectorsPCA.col(0));

    tm.block<3, 3>(0, 0) = eigenVectorsPCA.transpose();   //R.
    tm.block<3, 1>(0, 3) = -1.0f * (eigenVectorsPCA.transpose()) *(pcaCentroid.head<3>());//  -R*t

}
```
### 矩阵旋转
```C++
#include <pcl/common/transforms.h>
pcl::PointCloud<pcl::PointXYZ>::Ptr transformedCloud(new pcl::PointCloud<pcl::PointXYZ>);
pcl::transformPointCloud(*cloud, *transformedCloud, tm);
```
### 求最小包围盒
比较普通的处理是通过上面求质心点然后PCA求矩阵再转正最后直接获取对角线的两个点（也就是最大点最小点）即可。
```C++
pcl::PointXYZ min_p1, max_p1;
pcl::getMinMax3D(*transformedCloud, min_p1, max_p1);
```
但上面这种通常只是凑活能用，并不是真正的最小包围，原因就是转正的时候没法保证一个不规则物体哪个方向摆正才能正对着立方体包围起来就是最小包围。
如果要求比较高，也就是针对体积的最小包围盒，采用第三方[ApproxMVBB](https://github.com/gabyx/ApproxMVBB)其实比较好。
### 关于KDTree和OCTree
这两个用示例代码很难讲清楚只能多解释一下
#### KDTree
KD树，在k维空间中组织点云的数据结构，是一种二叉搜索树（对于3维点云，k=3）。可用于最近邻的搜索。简单说就是给定条件的二叉树一边大于一边小于这么往下来。KD树包括构建阶段和搜索阶段。主要用途也就通过把点云按KD树排列后加速搜索。
- 构建阶段：即对空间的切分，每轮切分按照一定原则先后切分k个维度。切分域(维度)的先后顺序可以根据方差（空间点的分布）确定，切分点选择中间点即可。
- 搜索阶段：1.寻找目标数据的近似最近点，即根据目标数据从根节点开始搜索kd树，找到对应的叶子节点作为近似最近点。2.回溯，沿着搜索路径回溯，以目标数据和近似最近点的距离作为判断依据，看看有无更近的点。
一般`setSearchMethod`让传入时懒得搞用下面代码即可创建用于提取搜索方法的kdtree树对象。
```C++
pcl::search::KdTree<pcl::PointXYZ>::Ptr tree(new pcl::search::KdTree<pcl::PointXYZ>);
tree->setInputCloud(cloud_in);
```
使用KdTree做检索,下面用上面创建的tree对象也可以只不过要用->,因为上面是指针。
```C++
#include <pcl/kdtree/kdtree_flann.h>
//K近邻搜索，即搜索该点周围的K个点。
void consoleKSearchPoint(pcl::PointCloud<pcl::PointXYZ>::Ptr cloud_in, pcl::PointXYZ searchPoint, int K) {
    pcl::KdTreeFLANN<pcl::PointXYZ> kdtree;
    kdtree.setInputCloud(cloud_in);
    //结果为cloud_in内的K个点的索引集合
    std::vector<int> pointIdxKNNSearch;
    //结果为周围的K个点与搜索点的平方距离
    std::vector<float> pointKNNSquaredDistance;
    if (kdtree.nearestKSearch(searchPoint, K, pointIdxKNNSearch, pointKNNSquaredDistance) > 0)
    {
        for (std::size_t i = 0; i < pointIdxKNNSearch.size(); ++i)
            std::cout << "x:" << (*cloud_in)[pointIdxKNNSearch[i]].x
            << " y:" << (*cloud_in)[pointIdxKNNSearch[i]].y
            << " z:" << (*cloud_in)[pointIdxKNNSearch[i]].z
            << " (squared distance: " << pointKNNSquaredDistance[i] << ")" << std::endl;
    }
}
//半径搜索
void consoleRadiusSearchPoint(pcl::PointCloud<pcl::PointXYZ>::Ptr cloud_in, pcl::PointXYZ searchPoint,double radius) {
    pcl::KdTreeFLANN<pcl::PointXYZ> kdtree;
    kdtree.setInputCloud(cloud_in);
    //结果为cloud_in内在搜索点半径radius范围内的点的索引集合
    std::vector<int> pointIdxRadiusSearch;
    //结果为搜索点半径radius范围内的点与搜索点的平方距离
    std::vector<float> pointRadiusSquaredDistance;
    if (kdtree.radiusSearch(searchPoint, radius, pointIdxRadiusSearch, pointRadiusSquaredDistance) > 0)
    {
        for (std::size_t i = 0; i < pointIdxRadiusSearch.size(); ++i)
            std::cout << "x:" << (*cloud_in)[pointIdxRadiusSearch[i]].x
            << " y:" << (*cloud_in)[pointIdxRadiusSearch[i]].y
            << " z:" << (*cloud_in)[pointIdxRadiusSearch[i]].z
            << " (squared distance: " << pointRadiusSquaredDistance[i] << ")" << std::endl;
    }
}
```

#### OCTree
八叉树是一种基于树的数据结构，每个节点有八个孩子。即每一次划分，都同时划分三个维度。主要用途包括空间划分、搜索、检测以及点云压缩。
一般setSearchMethod让传入时想用octree用下面代码即可
```C++
#include <pcl/octree/octree_search.h>
//resolution为八叉树分辨率，即最小体素的边长
pcl::octree::OctreePointCloudSearch<pcl::PointXYZ>::Ptr octree(new pcl::octree::OctreePointCloudSearch<pcl::PointXYZ>(resolution));
octree->setInputCloud(cloud_in);
octree->addPointsFromInputCloud();
```
##### 空间划分和搜索
三种搜索方式：体素内搜索，K近邻搜索，半径内搜索
使用与KDTree类似不过初始化用上面的`pcl::octree::OctreePointCloudSearch`即可
```C++
// 体素内搜索：输入搜索点，返回点索引向量
std::vector<int> pointIdxVec;
if (octree.voxelSearch(searchPoint, pointIdxVec)) {

}
// K近邻搜索
std::vector<int> pointIdxNKNSearch;
std::vector<float> pointNKNSquaredDistance;
if (octree.nearestKSearch(searchPoint, K, pointIdxNKNSearch, pointNKNSquaredDistance) > 0 {

}
// 半径内搜索
  std::vector<int> pointIdxRadiusSearch;
  std::vector<float> pointRadiusSquaredDistance;
 if (octree.radiusSearch(searchPoint, radius, pointIdxRadiusSearch, pointRadiusSquaredDistance) > 0) {

}
```
##### 空间变化检测
通过递归对比树结构，可以识别体素布局差异所体现的空间变化。
```C++
#include <pcl/octree/octree_pointcloud_changedetector.h>
/// <summary>
/// 八叉树点云变化检测
/// </summary>
/// <param name="cloud_in1">输入点云1</param>
/// <param name="cloud_in2">输入点云2</param>
/// <param name="resolution">八叉树分辨率，体素边长</param>
/// <param name="point_out">新的点的索引</param>
void octreeChangeDetector(pcl::PointCloud<pcl::PointXYZ>::Ptr cloud_in1, pcl::PointCloud<pcl::PointXYZ>::Ptr cloud_in2, float resolution, std::vector<int> & indices_out) {
    // 创建八叉树点云变化检测器
    pcl::octree::OctreePointCloudChangeDetector<pcl::PointXYZ> octree(resolution);
    // 添加CloudA的点到八叉树
    octree.setInputCloud(cloud_in1);
    octree.addPointsFromInputCloud();
    //切换缓冲区: 重置octree，但是保存之前内存中的树结构
    octree.switchBuffers();
    // 添加CloudB的点到octree
    octree.setInputCloud(cloud_in2);
    octree.addPointsFromInputCloud();
    // 获的新的点的索引
    octree.getPointIndicesFromNewVoxels(indices_out);
}
```
##### 点云压缩
点云压缩主要用于数据传输中，可以压缩单个点云或点云流。
###### 压缩配置文件：
压缩配置文件为PCL点云编码器定义了参数集。并针对压缩从OpenNI采集器获取的普通点云进行了优化设置。请注意，解码对象不需要用参数表示，因为它在解码时检测并获取对应的编码参数配置。下面的压缩配置文件是可用的：
- LOW_RES_ONLINE_COMPRESSION_WITHOUT_COLOR:分辨率1cm3，无颜色，快速在线编码
- LOW_RES_ONLINE_COMPRESSION_WITH_COLOR:分辨率1cm3，有颜色，快速在线编码
- MED_RES_ONLINE_COMPRESSION_WITHOUT_COLOR:分辨率5mm3，无颜色，快速在线编码
- MED_RES_ONLINE_COMPRESSION_WITH_COLOR:分辨率5mm3，有颜色，快速在线编码
- HIGH_RES_ONLINE_COMPRESSION_WITHOUT_COLOR:分辨率1mm3，无颜色，快速在线编码
- HIGH_RES_ONLINE_COMPRESSION_WITH_COLOR:分辨率1mm3，有颜色，快速在线编码
- LOW_RES_OFFLINE_COMPRESSION_WITHOUT_COLOR:分辨率1cm3，无颜色，高效离线编码
- LOW_RES_OFFLINE_COMPRESSION_WITH_COLOR:分辨率1cm3，有颜色，高效离线编码
- MED_RES_OFFLINE_COMPRESSION_WITHOUT_COLOR:分辨率5mm3，无颜色，高效离线编码
- MED_RES_OFFLINE_COMPRESSION_WITH_COLOR:分辨率5mm3，有颜色，高效离线编码
- HIGH_RES_OFFLINE_COMPRESSION_WITHOUT_COLOR:分辨率5mm3，无颜色，高效离线编码
- HIGH_RES_OFFLINE_COMPRESSION_WITH_COLOR:分辨率5mm3，有颜色，高效离线编码
- MANUAL_CONFIGURATION允许为高级参数化进行手工配置
```C++
#include <pcl/compression/octree_pointcloud_compression.h>
#include <fstream>

/// <summary>
/// 点云解压缩
/// </summary>
/// <param name="filepath">文件路径</param>
/// <param name="cloud_out">输出点云</param>
void compressionDecode(std::string filepath, pcl::PointCloud<pcl::PointXYZ>::Ptr& cloud_out) {
    pcl::io::OctreePointCloudCompression<pcl::PointXYZ>* PointCloudDecoder = new pcl::io::OctreePointCloudCompression<pcl::PointXYZ>();

    ifstream fin(filepath, ios::binary | ios::out);
    stringstream out;
    copy(istreambuf_iterator<char>(fin),
        istreambuf_iterator<char>(),
        ostreambuf_iterator<char>(out));
    fin.close();

    PointCloudDecoder->decodePointCloud(out, cloud_out);
    delete (PointCloudDecoder);
}

/// <summary>
/// 点云压缩
/// </summary>
/// <param name="cloud_in">输入点云</param>
/// <param name="filepath">文件路径</param>
void compressionEncode(std::string filepath, pcl::PointCloud<pcl::PointXYZ>::Ptr cloud_in) {
            
    //设置压缩选项为：分辨率1立方厘米，有颜色，快速在线编码
    pcl::io::compression_Profiles_e compressionProfile = pcl::io::LOW_RES_ONLINE_COMPRESSION_WITHOUT_COLOR;
    bool showStatistics = true;
    double pointResolution = 0.001;
    double octreeResolution = 0.01;
    bool doVoxelGridDownDownSampling = false;
    unsigned int iFrameRate = 30;
    bool doColorEncoding = true;
    unsigned char colorBitResolution = 6;

    //初始化压缩和解压缩对象  其中压缩对象需要设定压缩参数选项，解压缩按照数据源自行判断
    pcl::io::OctreePointCloudCompression<pcl::PointXYZ> *PointCloudEncoder = new pcl::io::OctreePointCloudCompression<pcl::PointXYZ>(compressionProfile, showStatistics, pointResolution, octreeResolution, doVoxelGridDownDownSampling,iFrameRate,doColorEncoding,colorBitResolution);

    std::stringstream compressedData;
    PointCloudEncoder->encodePointCloud(cloud_in->makeShared(), compressedData);

    fstream file(filepath, ios::binary | ios::out);
    file << compressedData.str();
    file.close();

    //删除压缩与解压缩的实例
    delete (PointCloudEncoder);
}
```
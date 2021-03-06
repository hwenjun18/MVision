# 纹理对象的实时姿态估计
[官方参考](https://docs.opencv.org/trunk/dc/d2c/tutorial_real_time_pose.html)
# 运行 
    一、生成物体三维纹理模型数据 ====
     首先 Data/box.ply mesh 文件 
       提供了 单张图片中 盒子 8个定点的 3d坐标点信息(以某个定点为世界坐标系原点)
       以及 6个 长方形面 构成的 12个三角形
     我们首先需要 生成其 3d描述 信息，需要运行 src/main_registration.cpp 获取
       1. 手动指定 图像中 物体顶点的位置（得到相机像素坐标系下的2d坐标 对应 ply文件中是3D位置）
       2. 求取 欧式变换矩阵
        由对应的 2d-3d点对关系
        u
        v  =  K × [R t] X
        1               Y
                        Z
                        1
        K 为图像拍摄时 相机的内参数
            世界坐标中的三维点(以文件中坐标为(0,0,0)某个定点为世界坐标系原点)
            经过 旋转矩阵R  和平移向量t 变换到相机坐标系下
            在通过相机内参数 变换到 相机的图像平面上
            
        由 PnP 算法可解的 旋转矩阵R  和平移向量t 
        
       3.  把从图像中得到的纹理信息 加入到 物体的三维纹理模型中
         在图像中提取 ORB特征点 和 对应的描述子
	     利用 内参数K 、 旋转矩阵R  和平移向量t  反向投影2d像素点 到  三维空间，获取3d坐标
	     标记 该反投影的3d点 是否在三维物体的 某个平面上
         
       4. 将 2d-3d点对 、关键点 以及 关键点描述子 存 入 物体的三维纹理模型中
          Data/cookies_ORB.yml
       
       运行:
       ./pnp_registration 
       
     二、使用数据库对视频/拍摄图像 进行实时 检测 目标定位
       主程序 src/main_detection.cpp
       1. 读取网格数据文件Data/box.ply  和 三维纹理数据文件Data/cookies_ORB.yml (上一步获取) 获取3d描述数据库 
       2. 对真实场景(视频文件帧/摄像头拍摄数据) 提取特征点2D 及其 描述子
       3. 与模型库中的 3d点带有的 描述子进行匹配，得到 2d-3d匹配点
       4. 使用PnP + Ransac 估计 当前物体的姿态 (R,t)
       5. 显示 PNP求解后　得到的内点
       6. 使用线性卡尔曼滤波去除错误的姿态估计变换矩阵 (R,t)
       7. 更新pnp 的　变换矩阵
       8. 将数据库中的 8个3D顶点 使用(R,t) 反投影到 图像2D平面上，绘制3D框，显示姿态
       
       运行
       ./pnp_detection 
       
# 项目分析
## 1 PLY网格模型，CSV格式的ply文件类
    src/CsvReader.cpp
    src/CsvWriter.cpp
    
## 2 三维纹理
    src/Mesh.cpp  实际CsvReader类读取了文件
    src/Model.cpp 模型读取保存 添加关键点、描述子、外点
    
    物体网格模型 物体的 三维纹理 模型文件
    包含：2d-3d点对 特征点 特征点描述子
   
    
## 3 ORB鲁棒匹配
    src/RobustMatcher.cpp
    鲁棒匹配 两者相互看对眼了
    图像1匹配到图像2的点　和图像2 匹配 图像1的点相互对应　才算是匹配

## 4  PnPRansac 2d-3d点对匹配 Rt变换求解器
    src/PnPProblem.cpp

## 5 由简单的 长方体 顶点  面 描述的ply文件 和 物体的彩色图像 生产 物体的三维纹理模型文件
    src/ModelRegistration.cpp 二维图像创建 三维数据  textured 3D model.
    程序执行，首先从输入图像中提取 ORB特征描述子，
    然后使用网格数据和 Möller–Trumbore intersection 算法 
    来计算特征的3D坐标系。
    最后，3D坐标点和特征描述子存在YAML格式文件的不同列表中，
    每一行存储一个不同的点.
    
    src/main_registration.cpp

        由简单的 长方体 顶点  面 描述的ply文件 和 物体的彩色图像 生产 物体的三维纹理模型文件
        2d-3d点配准
        【1】手动指定 图像中 物体顶点的位置（得到二维像素值位置）
            ply文件 中有物体定点的三维坐标
            由对应的 2d-3d点对关系
            u
            v  =  K × [R t] X
            1               Y
                        Z
                        1
            K 为图像拍摄时 相机的内参数
            世界坐标中的三维点(以文件中坐标为(0,0,0)某个定点为世界坐标系原点)
            经过 旋转矩阵R  和平移向量t 变换到相机坐标系下
            在通过相机内参数 变换到 相机的图像平面上
        【2】由 PnP 算法可解的 旋转矩阵R  和平移向量t 
        【3】把从图像中得到的纹理信息 加入到 物体的三维纹理模型中
            在图像中提取特征点 和对应的描述子
            利用 内参数K 、 旋转矩阵R  和平移向量t  反向投影到三维空间
               标记 该反投影的3d点 是否在三维物体的 某个平面上
        【4】将 2d-3d点对 、关键点 以及 关键点描述子 存入物体的三维纹理模型中

    
## 6 纹理对象的实时姿态估计
    src/main_detection.cpp

        【0】 读取网格数据文件 和 三维纹理数据文件 获取3d描述数据库 
              设置特征检测器 描述子提取器 描述子匹配器 
        【1】 场景中提取特征点 描述子 在　模型库(3d描述子数据库 
              3d points +description )中提取　匹配点 
        【2】 获取场景图片中　和　模型库中匹配的2d-3d点对
        【3】 使用PnP + Ransac进行姿态估计 
        【4】 显示 PNP求解后　得到的内点
        【5】 使用线性卡尔曼滤波去除错误的姿态估计
        【6】 更新pnp 的　变换矩阵
        【7】 显示位姿　轴 帧率 可信度 
        【8】显示调试数据 
        

        

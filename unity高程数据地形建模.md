---
title: untiy高程数据建模
categories:
- shengjunjie
tags:
- Unity 
---

通过水经注下载的tif高程数据，在unity中实现模型创建。
<!--more-->

## 解决方案
### 前期准备
1. global mapper、world creator软件下载


### 核心思路
将raw格式的高程数据导入unity中

### 详细过程

1. 将高程文件tif导入到Global Mapper中，需要将光晕渲染取消，同时将渲染模式选择为Gradient Shader。将处理后的图片输出为png格式，注意输出时文件格式选择为24-bit rgb。
2. 打开world creator在area,选择fit to Terrain Size,将border blend range左边滑轮拉到右边，使整个地形变平，勾选has height-map，选择file将png导入进来。
3. 在surface中percision选择为1/2m，在area中将level step选择为9等级。返回surface中filters添加渲染器Erosion fluvial、Erosion、Ridged。将渲染器中的height range范围选择为所选地形的范围高度（如果要增加细节，可以调节渲染器中的filter level strengths）。
4. 选择右上角的export，将地形的高程数据以raw 16-Bit的格式输出给unity（Normalization type选择为best fit）.
5. 将raw导入到unity的地形中，并添加纹理，地形就可以创建出来。



## 总结
untiy支持raw最大为4096*4096，一定要注意图片大小。另外纹理的图片大小也要注意，过大将无法导入到unity中去。

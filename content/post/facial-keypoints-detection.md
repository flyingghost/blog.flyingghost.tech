---
title: "深度学习实战：面部特征点检测"
date: 2019-07-16T15:13:57+08:00
draft: true
---

纸上得来终觉浅，绝知此事要躬行。囫囵吞枣的看了好几本机器学习相关入门书，总要拿个项目来练练手。

[Kaggle](https://www.kaggle.com/) 是个好网站，它以竞赛的形式提供大量数据分析/机器学习的题目，以“Making Data Science a Sport”为口号引诱各路英雄好汉冲击一个个难题。基于我们当前超低的技术水平和超高的自知之明，我们在“Getting Started”入门级竞赛题目中选择 [Facial Keypoints Detection](https://www.kaggle.com/c/facial-keypoints-detection) 作为练手项目。

这个任务的目标是在2D人脸图片上预测面部特征点的位置。原始数据给出了7000余张图片，以及每张图片上最多15个特征点。如图所示：

![](https://fg-public-1252239724.file.myqcloud.com/blog/0.png)

目标则是建立模型，能在给定规格的图片上检测出所有面部特征点。

## 数据描述

我们可以从 https://www.kaggle.com/c/facial-keypoints-detection/data 下载数据文件以及获得数据描述信息。

数据文件有好几个，其中`training.csv`是最重要的，提供了7049个训练数据。每行包含15个特征点坐标以及图片数据。

每个特征点用(x,y)坐标对标明。共有15种特征点：

- left_eye_center
- right_eye_center
- left_eye_inner_corner
- left_eye_outer_corner
- right_eye_inner_corner
- right_eye_outer_corner
- left_eyebrow_inner_end
- left_eyebrow_outer_end
- right_eyebrow_inner_end
- right_eyebrow_outer_end
- nose_tip
- mouth_left_corner
- mouth_right_corner
- mouth_center_top_lip
- mouth_center_bottom_lip

其中，左右是以主体为视角基准。

另外，并不是所有图片都有完整15个特征点，训练集中部分图片的某些特征点位置有缺失情况。

数据文件的最后一个字段是图片数据，逐行逐个像素组成列表，每个像素取值0-255，图片总尺寸96x96像素。

```
66.0335639098,39.0022736842,30.2270075188,36.4216781955,59.582075188,39.6474225564,73.1303458647,39.9699969925,36.3565714286,37.3894015038,23.4528721805,37.3894015038,56.9532631579,29.0336481203,80.2271278195,32.2281383459,40.2276090226,29.0023218045,16.3563789474,29.6474706767,44.4205714286,57.0668030075,61.1953082707,79.9701654135,28.6144962406,77.3889924812,43.3126015038,72.9354586466,43.1307067669,84.4857744361,238 236 237 238 ... (9216 pixels)
```

`test.csv`是留给我们的“作业”，包含1783个需要预测的图片。建立模型后我们将使用这些图片预测并生成特征点坐标。



## 数据读入

先读入数据集。由于Image字段原本是16进制的字符串，需要转换为numpy数组。

```python
TRAINING_CSV = path.join(os.getcwd(), 'data/training.csv')  # 定义CSV文件路径
train_data = pd.read_csv(TRAINING_CSV)  # 读入数据集
train_data['Image'] = train_data['Image'].apply(lambda im: np.fromstring(im, sep=' '))  # image字段的二进制字符串转为二进制
```


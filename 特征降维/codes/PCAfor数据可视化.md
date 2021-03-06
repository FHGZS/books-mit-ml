PCA for 数据可视化
=============

我们有一张小鸟的图片，这是一个三通道彩色图像：

<div style="text-align:center">
<img src="../attachments/bird_small.png"></img>
</div>

我们将图片的像素按颜色进行聚类，并在三维空间观察聚类成果：

<div style="text-align:center">
<img src="../attachments/3D观察.png" width="500"></img>
</div>

似乎在三维空间可视化不是那么直观，借助于PCA，我们将聚类结果降到二维空间进行可视化：

<div style="text-align:center">
<img src="../attachments/2D观察.png" width="350"></img>
</div>

测试代码：

```python
# coding: utf8
# pca/test_pca4visualization.py

import numpy as np
import kmeans
import pca
from scipy.io import loadmat
from mpl_toolkits.mplot3d import Axes3D
import matplotlib.pyplot as plt
import matplotlib.cm as cmx
import matplotlib.colors as colors

def getCmap(count):
    color_norm  = colors.Normalize(vmin=0, vmax=count-1)
    scalar_map = cmx.ScalarMappable(norm=color_norm, cmap='hsv')
    def map_index_to_rgb_color(index):
        return scalar_map.to_rgba(index)
    return map_index_to_rgb_color

fig = plt.figure()
ax = fig.add_subplot(111, projection='3d')

data = loadmat('data/bird_small.mat')
A = data['A']

A = A / 255.0;

height, width, channels = A.shape
X = np.mat(A.reshape(height * width, channels))

m, n = X.shape

clusterNum = 16
cmap = getCmap(clusterNum)
centroids, clusterAssment = kmeans.kMeans(X, clusterNum)
# 随机选择 1000 个样本绘制
sampleSize = 1000
sampleIndexs = np.random.choice(m, sampleSize)
clusters = clusterAssment[sampleIndexs]
samples = X[sampleIndexs]

# 三维下观察
for i in range(sampleSize):
    x, y, z = samples[i,:].A[0]
    center = clusters[i, 0]
    color = cmap(center)
    ax.scatter([x], [y], [z], color=color, marker='o')
plt.show()

# 二维下观察
reducedSamples = pca.PCA(samples, k=2)[1]
for i in range(sampleSize):
    x, y = reducedSamples[i,:].A[0]
    center = clusters[i, 0]
    color = cmap(center)
    plt.scatter([x], [y], color=color, marker='o')
plt.show()
```

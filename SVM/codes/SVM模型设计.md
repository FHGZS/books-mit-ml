程序示例--基于 SMO 的 SVM 模型
============

在这里，我们会实现一个基于 SMO 的 SVM 模型，在其中，提供了**简化版 SMO ** 和 **完整版 SMO** 的实现。

- 简化版 SMO：不使用启发式方法选择 $$(\alpha^{(i)}, \alpha^{(j)})$$，始终在整个样本集上选择违反了 KKT 条件的 $$\alpha^(i)$$，$$\alpha^{(j)}$$ 则随机选择。

- 完整版 SMO：使用启发式方法选择 $$(\alpha^{(i)}, \alpha^{(j)})$$。在整个训练集或者非边界样本中选择违反了 KKT 条件的 $$\alpha^{(i)}$$。并选择使得 $$|E^{(i)} - E^{(j)}|$$ 达到最大的 $$alpha^{(j)}$$。

### 核函数

模型支持了核函数，并提供了**线性核函数** 和 **高斯核（又称之为径向基核函数 RBF）函数** 使用：

```python
# coding: utf8
# svm/smo.py

def linearKernel():
    """线性核函数
    """
    def calc(X, A):
        return X * A.T
    return calc

def rbfKernel(delta):
    """rbf核函数
    """
    gamma = 1.0 / (2 * delta**2)

    def calc(X, A):
        return np.mat(rbf_kernel(X, A, gamma=gamma))
    return calc
```

> RBF 核的实现我们使用了性能更高的 `sklearn.metrics.pairwise` 提供的的 `rbf_kernel`

### 训练初始化

```python
def getSmo(X, y, C, tol, maxIter, kernel=linearKernel()):
    """SMO

    Args:
        X 训练样本
        y 标签集
        C 正规化参数
        tol 容忍值
        maxIter 最大迭代次数
        K 所用核函数

    Returns:
        trainSimple 简化版训练算法
        train 完整版训练算法
        predict 预测函数
    """
    m, n = X.shape
    # 存放核函数的转化结果
    K = kernel(X, X)
    # Cache存放预测误差，用以加快计算速度
    ECache = np.zeros((m,2))

    # ...
```

### 权值向量 $$w$$

我们知道，模型 $$f(x)$$ 仅仅与支持向量有关，所以，权值 $$w$$ 可以定义为：

$$

\begin{align*}
w &= \sum_{i=1}^m\alpha^{(i)} y^{(i)} (x^{(i)})^T \\
& = \sum_{i \in S}\alpha^{(i)} y^{(i)} (x^{(i)})^T
\end{align*}

$$

其中，$$S$$ 表示了属于支持向量的样本坐标集合。

```python
# coding: utf8
# svm/smo.py

def getSmo(X, y, C, tol, maxIter, kernel=linearKernel()):
    # ...
    def w(alphas, b, supportVectorsIndex, supportVectors):
        return (np.multiply(alphas[supportVectorsIndex], y[
            supportVectorsIndex]).T * supportVectors).T
```

### 预测误差 $$E^{(i)}$$

$$

\begin{align*}
E^{(i)} &= f(x^{(i)}) - y^{(i)} \\
    &= \sum_{j=1}^m \alpha^{(j)} y^{(j)} k(x^{(j)}, x^{(i)}) + b - y^{(i)}
\end{align*}

$$

```python
# coding: utf8
# svm/smo.py

def getSmo(X, y, C, tol, maxIter, kernel=linearKernel()):
    # ...
    def E(i, alphas, b):
        """计算预测误差

        Args:
            i i
            alphas alphas
            b b
        Returns:
            E_i 第i个样本的预测误差
        """
        FXi = float(np.multiply(alphas, y).T * K[:, i]) + b
        E = FXi - float(y[i])
        return E
```

### 预测函数

$$

f(x) = \sum_{i \in S}\alpha^{(i)} y^{(i)} (x^{(i)})^T + b

$$

```python
# coding: utf8
# svm/smo.py

def getSmo(X, y, C, tol, maxIter, kernel=linearKernel()):
    # ...
    def predict(X, alphas, b, supportVectorsIndex, supportVectors):
        """计算权值向量

        Args:
            X 预测矩阵
            alphas alphas
            b b
            supportVectorsIndex 支持向量坐标
            supportVectors 支持向量
        Returns:
            predicts 预测结果
        """
        Ks = kernel(supportVectors, X)
        predicts = (np.multiply(alphas[supportVectorsIndex], y[
            supportVectorsIndex]).T * Ks + b).T
        predicts = np.sign(predicts)
        return predicts
```

### 选择 $$\alpha^{(i)}$$

```python
# coding: utf8
# svm/smo.py

def getSmo(X, y, C, tol, maxIter, kernel=linearKernel()):
        # ...
    def select(i, alphas, b):
        """alpha对选择
        """
        Ei = E(i, alphas, b)
        # 选择违背KKT条件的，作为alpha2
        Ri = y[i] * Ei
        if (Ri < -tol and alphas[i] < C) or \
                (Ri > tol and alphas[i] > 0):
            # 选择第二个参数
            j = selectJRand(i)
            Ej = E(j, alphas, b)
            # j, Ej = selectJ(i, Ei, alphas, b)
            # get bounds
            if y[i] != y[j]:
                L = max(0, alphas[j] - alphas[i])
                H = min(C, C + alphas[j] - alphas[i])
            else:
                L = max(0, alphas[j] + alphas[i] - C)
                H = min(C, alphas[j] + alphas[i])
            if L == H:
                return 0, alphas, b
            Kii = K[i, i]
            Kjj = K[j, j]
            Kij = K[i, j]
            eta = 2.0 * Kij - Kii - Kjj
            if eta >= 0:
                return 0, alphas, b
            iOld = alphas[i].copy()
            jOld = alphas[j].copy()
            alphas[j] = jOld - y[j] * (Ei - Ej) / eta
            if alphas[j] > H:
                alphas[j] = H
            elif alphas[j] < L:
                alphas[j] = L
            if abs(alphas[j] - jOld) < tol:
                alphas[j] = jOld
                return 0, alphas, b
            alphas[i] = iOld + y[i] * y[j] * (jOld - alphas[j])
            # update ECache
            updateE(i, alphas, b)
            updateE(j, alphas, b)
            # update b
            bINew = b - Ei - y[i] * (alphas[i] - iOld) * Kii - y[j] * \
                (alphas[j] - jOld) * Kij
            bJNew = b - Ej - y[i] * (alphas[i] - iOld) * Kij - y[j] * \
                (alphas[j] - jOld) * Kjj
            if alphas[i] > 0 and alphas[i] < C:
                bNew = bINew
            elif alphas[j] > 0 and alphas[j] < C:
                bNew = bJNew
            else:
                bNew = (bINew + bJNew) / 2
            return 1, alphas, b
        else:
            return 0, alphas, b
```

### 选择 $$\alpha^{(j)}$$

```python
# coding: utf8
# svm/smo.py

def getSmo(X, y, C, tol, maxIter, kernel=linearKernel()):
        # ...
    def selectJRand(i):
        """
        """
        j = i
        while j == i:
            j = int(np.random.uniform(0, m))
        return j

    def selectJ(i, Ei, alphas, b):
        """选择权值 $$\alpha^{(i)}$$
        """
        maxJ = 0; maxDist=0; Ej = 0
        ECache[i] = [1, Ei]
        validCaches = np.nonzero(ECache[:, 0])[0]
        if len(validCaches) > 1:
            for k in validCaches:
                if k==i: continue
                Ek = E(k, alphas, b)
                dist = np.abs(abs(Ei-Ek))
                if maxDist < dist:
                    Ej = Ek
                    maxJ = k
                    maxDist = dist
            return maxJ, Ej
        else:
            ### 随机选择
            j = selectJRand(i)
            Ej = E(j, alphas, b)
            return j, Ej
```

### 完整版 SMO 训练方法：

```python
# coding: utf8
# svm/smo.py

def getSmo(X, y, C, tol, maxIter, kernel=linearKernel()):
    # ...
    def train():
        """完整版训练算法

        Returns:
            alphas alphas
            w w
            b b
            supportVectorsIndex 支持向量的坐标集
            supportVectors 支持向量
            iterCount 迭代次数
        """
        numChanged = 0
        examineAll = True
        iterCount = 0
        alphas = np.mat(np.zeros((m, 1)))
        b = 0
        # 如果所有alpha都遵从 KKT 条件，则在整个训练集上迭代
        # 否则在处于边界内 (0, C) 的 alpha 中迭代
        while (numChanged > 0 or examineAll) and (iterCount < maxIter):
            numChanged = 0
            if examineAll:
                for i in range(m):
                    changed, alphas, b = select(i, alphas, b)
                    numChanged += changed
            else:
                nonBoundIds = np.nonzero((alphas.A > 0) * (alphas.A < C))[0]
                for i in nonBoundIds:
                    changed, alphas, b = select(i, alphas, b)
                    numChanged += changed
            iterCount += 1

            if examineAll:
                examineAll = False
            elif numChanged == 0:
                examineAll = True
        supportVectorsIndex = np.nonzero(alphas.A > 0)[0]
        supportVectors = np.mat(X[supportVectorsIndex])
        return alphas, w(alphas, b, supportVectorsIndex, supportVectors), b, \
            supportVectorsIndex, supportVectors, iterCount
```

### 简化版 SMO 训练方法：

```python
# coding: utf8
# svm/smo.py

def getSmo(X, y, C, tol, maxIter, kernel=linearKernel()):
    # ...
    def trainSimple():
        """简化版训练算法

        Returns:
            alphas alphas
            w w
            b b
            supportVectorsIndex 支持向量的坐标集
            supportVectors 支持向量
            iterCount 迭代次数
        """
        numChanged = 0
        iterCount = 0
        alphas = np.mat(np.zeros((m, 1)))
        b = 0
        L = 0
        H = 0
        while iterCount < maxIter:
            numChanged = 0
            for i in range(m):
                Ei = E(i, alphas, b)
                Ri = y[i] * Ei
                # 选择违背KKT条件的，作为alpha2
                if (Ri < -tol and alphas[i] < C) or \
                        (Ri > tol and alphas[i] > 0):
                    # 选择第二个参数
                    j = selectJRand(i)
                    Ej = E(j, alphas, b)
                    # get bounds
                    if y[i] != y[j]:
                        L = max(0, alphas[j] - alphas[i])
                        H = min(C, C + alphas[j] - alphas[i])
                    else:
                        L = max(0, alphas[j] + alphas[i] - C)
                        H = min(C, alphas[j] + alphas[i])
                    if L == H:
                        continue
                    Kii = K[i, i]
                    Kjj = K[j, j]
                    Kij = K[i, j]
                    eta = 2.0 * Kij - Kii - Kjj
                    if eta >= 0:
                        continue
                    iOld = alphas[i].copy();
                    jOld = alphas[j].copy()
                    alphas[j] = jOld - y[j] * (Ei - Ej) / eta
                    if alphas[j] > H:
                        alphas[j] = H
                    elif alphas[j] < L:
                        alphas[j] = L
                    if abs(alphas[j] - jOld) < tol:
                        alphas[j] = jOld
                        continue
                    alphas[i] = iOld + y[i] * y[j] * (jOld - alphas[j])
                    # update b
                    bINew = b - Ei - y[i] * (alphas[i] - iOld) * Kii - y[j] * \
                        (alphas[j] - jOld) * Kij
                    bJNew = b - Ej - y[i] * (alphas[i] - iOld) * Kij - y[j] * \
                        (alphas[j] - jOld) * Kjj
                    if alphas[i] > 0 and alphas[i] < C:
                        b = bINew
                    elif alphas[j] > 0 and alphas[j] < C:
                        b = bJNew
                    else:
                        b = (bINew + bJNew) / 2.0
                    numChanged += 1
            if numChanged == 0:
                iterCount += 1
            else:
                iterCount = 0
        supportVectorsIndex = np.nonzero(alphas.A > 0)[0]
        supportVectors = np.mat(X[supportVectorsIndex])
        return alphas, w(alphas, b, supportVectorsIndex, supportVectors), b, \
            supportVectorsIndex, supportVectors, iterCount
```

### 完整代码

```python
# coding: utf8
# svm/smo.py

import numpy as np

from sklearn.metrics.pairwise import rbf_kernel

"""
svm模型
"""

def linearKernel():
    """线性核函数
    """
    def calc(X, A):
        return X * A.T
    return calc

def rbfKernel(delta):
    """rbf核函数
    """
    gamma = 1.0 / (2 * delta**2)

    def calc(X, A):
        return np.mat(rbf_kernel(X, A, gamma=gamma))
    return calc

def getSmo(X, y, C, tol, maxIter, kernel=linearKernel()):
    """SMO

    Args:
        X 训练样本
        y 标签集
        C 正规化参数
        tol 容忍值
        maxIter 最大迭代次数
        K 所用核函数

    Returns:
        trainSimple 简化版训练算法
        train 完整版训练算法
        predict 预测函数
    """
    # 存放核函数的转化结果
    K = kernel(X, X)
    # Cache存放预测误差，用以加快计算速度
    ECache = np.zeros((m,2))

    def predict(X, alphas, b, supportVectorsIndex, supportVectors):
        """计算权值向量

        Args:
            X 预测矩阵
            alphas alphas
            b b
            supportVectorsIndex 支持向量坐标集
            supportVectors 支持向量
        Returns:
            predicts 预测结果
        """
        Ks = kernel(supportVectors, X)
        predicts = (np.multiply(alphas[supportVectorsIndex], y[
            supportVectorsIndex]).T * Ks + b).T
        predicts = np.sign(predicts)
        return predicts

    def w(alphas, b, supportVectorsIndex, supportVectors):
        """计算权值

        Args:
            alphas alphas
            b b
            supportVectorsIndex 支持向量坐标
            supportVectors 支持向量
        Returns:
            w 权值向量
        """
        return (np.multiply(alphas[supportVectorsIndex], y[
            supportVectorsIndex]).T * supportVectors).T

    def E(i, alphas, b):
        """计算预测误差

        Args:
            i i
            alphas alphas
            b b
        Returns:
            E_i 第i个样本的预测误差
        """
        FXi = float(np.multiply(alphas, y).T * K[:, i]) + b
        E = FXi - float(y[i])
        return E

    def updateE(i, alphas, b):
        ECache[i] = [1, E(i, alphas, b)]

    def selectJRand(i):
        """
        """
        j = i
        while j == i:
            j = int(np.random.uniform(0, m))
        return j

    def selectJ(i, Ei, alphas, b):
        """选择权值 $$\alpha^{(i)}$$
        """
        maxJ = 0; maxDist=0; Ej = 0
        ECache[i] = [1, Ei]
        validCaches = np.nonzero(ECache[:, 0])[0]
        if len(validCaches) > 1:
            for k in validCaches:
                if k==i: continue
                Ek = E(k, alphas, b)
                dist = np.abs(abs(Ei-Ek))
                if maxDist < dist:
                    Ej = Ek
                    maxJ = k
                    maxDist = dist
            return maxJ, Ej
        else:
            ### 随机选择
            j = selectJRand(i)
            Ej = E(j, alphas, b)
            return j, Ej

    def select(i, alphas, b):
        """alpha对选择
        """
        Ei = E(i, alphas, b)
        # 选择违背KKT条件的，作为alpha2
        Ri = y[i] * Ei
        if (Ri < -tol and alphas[i] < C) or \
                (Ri > tol and alphas[i] > 0):
            # 选择第二个参数
            j = selectJRand(i)
            Ej = E(j, alphas, b)
            # j, Ej = selectJ(i, Ei, alphas, b)
            # get bounds
            if y[i] != y[j]:
                L = max(0, alphas[j] - alphas[i])
                H = min(C, C + alphas[j] - alphas[i])
            else:
                L = max(0, alphas[j] + alphas[i] - C)
                H = min(C, alphas[j] + alphas[i])
            if L == H:
                return 0, alphas, b
            Kii = K[i, i]
            Kjj = K[j, j]
            Kij = K[i, j]
            eta = 2.0 * Kij - Kii - Kjj
            if eta >= 0:
                return 0, alphas, b
            iOld = alphas[i].copy()
            jOld = alphas[j].copy()
            alphas[j] = jOld - y[j] * (Ei - Ej) / eta
            if alphas[j] > H:
                alphas[j] = H
            elif alphas[j] < L:
                alphas[j] = L
            if abs(alphas[j] - jOld) < tol:
                alphas[j] = jOld
                return 0, alphas, b
            alphas[i] = iOld + y[i] * y[j] * (jOld - alphas[j])
            # update ECache
            updateE(i, alphas, b)
            updateE(j, alphas, b)
            # update b
            bINew = b - Ei - y[i] * (alphas[i] - iOld) * Kii - y[j] * \
                (alphas[j] - jOld) * Kij
            bJNew = b - Ej - y[i] * (alphas[i] - iOld) * Kij - y[j] * \
                (alphas[j] - jOld) * Kjj
            if alphas[i] > 0 and alphas[i] < C:
                bNew = bINew
            elif alphas[j] > 0 and alphas[j] < C:
                bNew = bJNew
            else:
                bNew = (bINew + bJNew) / 2
            return 1, alphas, b
        else:
            return 0, alphas, b

    def train():
        """完整版训练算法

        Returns:
            alphas alphas
            w w
            b b
            supportVectorsIndex 支持向量的坐标集
            supportVectors 支持向量
            iterCount 迭代次数
        """
        numChanged = 0
        examineAll = True
        iterCount = 0
        alphas = np.mat(np.zeros((m, 1)))
        b = 0
        # 如果所有alpha都遵从 KKT 条件，则在整个训练集上迭代
        # 否则在处于边界内 (0, C) 的 alpha 中迭代
        while (numChanged > 0 or examineAll) and (iterCount < maxIter):
            numChanged = 0
            if examineAll:
                for i in range(m):
                    changed, alphas, b = select(i, alphas, b)
                    numChanged += changed
            else:
                nonBoundIds = np.nonzero((alphas.A > 0) * (alphas.A < C))[0]
                for i in nonBoundIds:
                    changed, alphas, b = select(i, alphas, b)
                    numChanged += changed
            iterCount += 1

            if examineAll:
                examineAll = False
            elif numChanged == 0:
                examineAll = True
        supportVectorsIndex = np.nonzero(alphas.A > 0)[0]
        supportVectors = np.mat(X[supportVectorsIndex])
        return alphas, w(alphas, b, supportVectorsIndex, supportVectors), b, \
            supportVectorsIndex, supportVectors, iterCount

    def trainSimple():
        """简化版训练算法

        Returns:
            alphas alphas
            w w
            b b
            supportVectorsIndex 支持向量的坐标集
            supportVectors 支持向量
            iterCount 迭代次数
        """
        numChanged = 0
        iterCount = 0
        alphas = np.mat(np.zeros((m, 1)))
        b = 0
        L = 0
        H = 0
        while iterCount < maxIter:
            numChanged = 0
            for i in range(m):
                Ei = E(i, alphas, b)
                Ri = y[i] * Ei
                # 选择违背KKT条件的，作为alpha2
                if (Ri < -tol and alphas[i] < C) or \
                        (Ri > tol and alphas[i] > 0):
                    # 选择第二个参数
                    j = selectJRand(i)
                    Ej = E(j, alphas, b)
                    # get bounds
                    if y[i] != y[j]:
                        L = max(0, alphas[j] - alphas[i])
                        H = min(C, C + alphas[j] - alphas[i])
                    else:
                        L = max(0, alphas[j] + alphas[i] - C)
                        H = min(C, alphas[j] + alphas[i])
                    if L == H:
                        continue
                    Kii = K[i, i]
                    Kjj = K[j, j]
                    Kij = K[i, j]
                    eta = 2.0 * Kij - Kii - Kjj
                    if eta >= 0:
                        continue
                    iOld = alphas[i].copy();
                    jOld = alphas[j].copy()
                    alphas[j] = jOld - y[j] * (Ei - Ej) / eta
                    if alphas[j] > H:
                        alphas[j] = H
                    elif alphas[j] < L:
                        alphas[j] = L
                    if abs(alphas[j] - jOld) < tol:
                        alphas[j] = jOld
                        continue
                    alphas[i] = iOld + y[i] * y[j] * (jOld - alphas[j])
                    # update b
                    bINew = b - Ei - y[i] * (alphas[i] - iOld) * Kii - y[j] * \
                        (alphas[j] - jOld) * Kij
                    bJNew = b - Ej - y[i] * (alphas[i] - iOld) * Kij - y[j] * \
                        (alphas[j] - jOld) * Kjj
                    if alphas[i] > 0 and alphas[i] < C:
                        b = bINew
                    elif alphas[j] > 0 and alphas[j] < C:
                        b = bJNew
                    else:
                        b = (bINew + bJNew) / 2.0
                    numChanged += 1
            if numChanged == 0:
                iterCount += 1
            else:
                iterCount = 0
        supportVectorsIndex = np.nonzero(alphas.A > 0)[0]
        supportVectors = np.mat(X[supportVectorsIndex])
        return alphas, w(alphas, b, supportVectorsIndex, supportVectors), b, \
            supportVectorsIndex, supportVectors, iterCount
    return trainSimple, train, predict
```

参考资料
---------------
- [《机器学习实战》](https://item.jd.com/11242112.html)
- [《机器学习》](https://item.jd.com/11867803.html)

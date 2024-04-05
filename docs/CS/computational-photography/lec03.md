# Lec03 交互式图像分割与抠像

!!! note ""

    - 图像分割指的是将数字图像中感兴趣的区域分割出来。
    - 抠像除了要得到前景轮廓外还要进一步估计半透明通道，以实现真实的合成。

!!! tips "内容"

    1. 常用模型
        1. 高斯混合模型；
        2. 马尔科夫随机场；
    2. 交互式图像分割算法
       1. GrabCut；
       2. Lazy Snapping；
    3. 抠像算法
       1. 贝叶斯抠像
       2. Closed Form Matting；
    4. 基于深度学习的图像分割
    5. 基于深度学习的抠像

## 常用模型

### 高斯混合模型

#### 混合模型

混合模型是一个可以用来表示在总体分布（distribution）中含有 K 个子分布的概率模型，换句话说，混合模型表示了观测数据在总体中的概率分布，它是一个由 K 个子分布组成的混合分布。混合模型不要求观测数据提供关于子分布的信息，来计算观测数据在总体分布中的概率。

#### 高斯模型

高斯模型就是用高斯概率密度函数（正态分布曲线）精确地量化事物，将一个事物分解为若干的基于高斯概率密度函
数（正态分布曲线）形成的模型。

高斯模型有单高斯模型（SGM）和混合高斯模型（GMM）两种

##### 单高斯模型 | SGM

当样本数据 X 是一维数据（Univariate）时，高斯分布遵从下方概率密度函数（Probability Density Function）：

$$
N(x; \mu,\sigma^2) = \frac{1}{\sqrt{2\pi\sigma^2}}exp\left(-\frac{(x-\mu)^2}{2\sigma^2}\right)
$$

其中，$\mu$ 是均值，$\sigma^2$ 是方差。

当样本数据 X 是多维数据（Multivariate）时，高斯分布遵从下方概率密度函数：

$$
N(x; \mu,\Sigma) = \frac{1}{(2\pi)^{D/2}|\Sigma|^{1/2}}exp\left(-\frac{1}{2}(x-\mu)^T\Sigma^{-1}(x-\mu)\right)
$$

其中，$\mu$ 是均值，$\Sigma$ 是协方差矩阵。

在实际应用中，$\mu$ 通常用样本均值来代替，$\Sigma$ 通常用样本协方差矩阵来代替。

因为每个类别都有自己的 $\mu$ 和 $\Sigma$，所以我们很容易判断一个样本 $x$ 是否属于类别 C，只需把 $x$ 代入上式，当概率大于一定阈值时我们就认为 $x$ 属于类别 C。

从几何上讲，单高斯分布模型在二维空间应该近似于椭圆，在三维空间上近似于椭球。

![alt text](images/image-72.png){width=50%}

##### 高斯混合模型 | GMM

GMM（Gaussian Mixture Model），高斯混合模型（或者混合高斯模型），也可以简写为 MOG（Mixture of Gaussian）。

高斯混合模型可以看作是由 $K$ 个单高斯模型组合而成的模型，这 $K$ 个子模型是混合模型的隐变量（Hidden variable）。一般来说，一个混合模型可以使用任何概率分布，这里使用高斯混合模型是因为高斯分布具备很好的数学性质以及良好的计算性能。

举个不是特别稳妥的例子，比如我们现在有一组狗的样本数据，不同种类的狗，体型、颜色、长相各不相同，但都属于狗这个种类，此时单高斯模型可能不能很好的来描述这个分布，因为样本数据分布并不是一个单一的椭圆，所以用混合高斯分布可以更好的描述这个问题，如下图所示：

![alt text](images/image-73.png)

GMM认为数据是从几个GSM中生成出来的，即

$$
Pr(x) = \sum_{k=1}^{K} \pi_k N(x; \mu_k,\Sigma_k)
$$

其中的任意一个高斯分布 $N(x; \mu_k,\Sigma_k)$ 称为一个分量（Component）；$K$ 需要事先确定好，代表component个数；$\pi_k$ 是权值系数，满足 $\sum_{k=1}^{K} \pi_k = 1$。

将 $K$ 个高斯模型混合在一起，每个点出现的概率是几个高斯混合的结果。当 $K$ 足够大时，GMM可以用来逼近任意连续的概率密度分布。

![alt text](images/image-74.png)

GMM是一种聚类算法，每个component就是一个聚类中心。为了在只有样本点，不知道样本分类（含有隐含变量）的情况下，计算出模型参数 $\pi_k, \mu_k, \Sigma_k$，可以使用EM算法。

!!! note "样本分类已知情况下的 GMM"

    当每个样本所属分类已知时，GMM的参数非常好确定，直接利用Maximum Likelihood。设样本容量为 $N$ ，属于$K$个分类的样本数量分别是 $N_1, N_2, ..., N_k$，属于第 $k$ 个分类的样本集合是 $L(k)$。 

    于是我们有：

    $$
    \begin{aligned}
    \pi_k &= \frac{N_k}{N} \\
    \mu_k &= \frac{1}{N_k} \sum_{x \in L(k)} x \\
    \Sigma_k &= \frac{1}{N_k} \sum_{x \in L(k)} (x-\mu_k)(x-\mu_k)^T
    \end{aligned}
    $$

    其中，$\pi_k$ 是第 $k$ 个分类的权重，$\mu_k$ 是第 $k$ 个分类的均值，$\Sigma_k$ 是第 $k$ 个分类的协方差矩阵。

!!! note "样本分类未知情况下的 GMM"

    !!! note "对数似然函数"
        有 $N$ 个数据点，服从某种分布 $Pr(x;\theta)$，我们想找到一组参数 $\theta$，使得生成这些数据点的概率最大，这个概率就是

        $$ \prod_{i=1}^{N} Pr(x_i;\theta) $$

        这称作似然函数（Likelihood Function）。
        
        通常单个点的概率很小，连乘之后数据会更小，容易造成浮点数下溢，所以一般取其对数，变成

        $$ \sum_{i=1}^{N} log(Pr(x_i;\theta)) $$

        这称作对数似然函数（Log Likelihood Function）。

    对于 GMM，我们的目标是找到一组参数 $\theta = \{\pi_k, \mu_k, \Sigma_k\}$，使得数据点 $x_i$ 的对数似然函数最大，即

    $$ \max_{\theta} \sum_{i=1}^{N} log(\sum_{k=1}^{K} \pi_k N(x_i; \mu_k,\Sigma_k)) $$

    这里每个样本 $x_i$ 所属的类别 $z_k$ 是未知的，$Z$ 是隐含变量，我们就是要找到最佳的模型参数，使得上式所示的期望最大，**“期望最大化算法”**名字由此而来。

    对于每个观测数据点来说，事先并不知道它是属于哪个子分布的（hidden variable），因此 $\log$ 里面还有求和，对于每个子模型都有未知的 $\pi_k, \mu_k, \Sigma_k$，直接求导无法计算。需要通过迭代的方法求解。

    !!! note "EM 算法"

        EM 算法是一种迭代算法，1977 年由 Dempster 等人总结提出，用于含有隐变量（Hidden variable）的概率模型参数的最大似然估计。

        EM 要求解的问题一般形式是:

        $$
        \theta^* = \arg \max_{\theta} \prod_{j=1}^{|x|}\sum_{y\in Y} Pr(X=x_j,Y=y;\theta)
        $$

        其中，$X$ 是观测变量，$Y$ 是隐变量，$\theta$ 是模型参数。

        EM算法的基本思路是：随机初始化一组参数 $\theta(0)$，根据后验概率 $Pr(Y|X;\theta)$ 来更新 $Y$ 的期望 $E(Y)$，然后用 $E(Y)$ 代替 $Y$ 求出新的模型参数 $\theta(1)$。如此迭代直到 $\theta$ 趋于稳定。

        > 如果数据点的分类标签 $Y$ 是已知的，那么求解模型参数直接利用Maximum Likelihood就可以了。

        !!! note "E-Step"
            E 是Expectation的意思，就是假设模型参数已知的情况下求隐含变量 $Z$ 分别取 $z_1,z_2,...$ 的期望，亦即 $Z$ 分别取 $z_1,z_2,...$ 的概率。在 GMM 中就是求数据点由各个 component 生成的概率。

            $$ \gamma(i,k) = \alpha_k Pr(z_k|x_i;\pi,\mu,\Sigma) $$

            注意到我们在Z的后验概率前面乘以了一个权值因子 $\alpha_k$，它表示在训练集中数据点属于类别 $z_k$ 的频率，在 GMM 中它就是 $\pi_k$。

            因为 $Pr(x) = \sum_{k=1}^{K} \pi_k N(x; \mu_k,\Sigma_k)$，所以

            $$ 
            \gamma(i,k) = \alpha_k \frac{\pi_k N(x_i; \mu_k,\Sigma_k)}{\sum_{j=1}^{K} \pi_j N(x_i; \mu_j,\Sigma_j)}
            $$

        !!! note "M-Step"
            M 就是Maximization的意思，就是用最大似然的方法求出模型参数。

            现在我们认为上一步求出的 $\gamma(i,k)$ 就是“数据点 $x_i$ 由 component $k$ 生成的概率”，那么我们可以用这个概率来更新模型参数。

            $$
            \begin{aligned}
            N_k &= \sum_{i=1}^{N} \gamma(i,k) \\
            \pi_k &= \frac{N_k}{N} \\
            \mu_k &= \frac{1}{N_k} \sum_{i=1}^{N} \gamma(i,k) x_i \\
            \Sigma_k &= \frac{1}{N_k} \sum_{i=1}^{N} \gamma(i,k) (x_i-\mu_k)(x_i-\mu_k)^T
            \end{aligned}
            $$

        重复计算 E-step 和 M-step 直至收敛 （$\|\theta_{i+1}-\theta_i\| < \epsilon$，$\epsilon$ 是一个很小的正数，表示经过一次迭代之后参数变化非常小）。

    至此，我们就找到了高斯混合模型的参数。需要注意的是，EM 算法具备收敛性，但并不保证找到全局最大值，有可能找到局部最大值。解决方法是初始化几次不同的参数进行迭代，取结果最好的那次。

### 马尔科夫随机场与图像处理

#### 马尔科夫随机过程

通俗的讲，马尔科夫随机过程就是，下一个时间点的状态只与当前的状态有关系，而与以前的状态没有关系，即未来的状态决定于现在而不决定于过去。

!!! note "一维马尔科夫过程"

    设有随机过程 $\{X_n,n\in T\}$，若对于任意正整数 $n \in T$ 和任意的 $i_0,i_1,...,i_{n+1} \in I$，条件概率满足

    $$
    P(X_{n+1}=i_{n+1}|X_n=i_n,X_{n-1}=i_{n-1},...,X_0=i_0) = P(X_{n+1}=i_{n+1}|X_n=i_n)
    $$

    就称 $\{X_n,n\in T\}$ 为马尔科夫过程，该随机过程的统计特性完全由条件概率所决定。

马尔科夫随机过程可以按照参数集和状态空间分成四类：

- 时间和状态都是离散的马尔科夫过程。也称为马尔科夫链；
- 时间连续、状态离散的马尔科夫过程。通常称为纯不连续马尔科夫过程；
- 时间和状态都是连续的马尔科夫过程；
- 时间离散、状态连续的马尔科夫过程。

!!! note "马尔科夫随机场"

    马尔科夫随机场包含两层意思：
    - 马尔科夫性质
    - 随机场

    !!! note "马尔科夫性质"

        马尔科夫性质指的是一个随机变量序列按时间先后关系依次排开的时候，第 $N+1$ 时刻的分布特性，与 $N$ 时刻以前的随机变量的取值无关。

        即在一个随机过程中，给定现在状态，未来状态与过去状态是独立的。即

        $$
        P(X_{n+1}=i_{n+1}|X_n=i_n,X_{n-1}=i_{n-1},...,X_0=i_0) = P(X_{n+1}=i_{n+1}|X_n=i_n)
        $$

    !!! note "随机场"

        **随机场**是一种随机变量的集合，这些随机变量的取值是空间上的某种结构，比如二维平面上的像素点。

        当给每一个位置中按照某种分布随机赋予相空间的一个值之后，其全体就叫做随机场。其中有两个概念：位置（site），相空间（phase space）。

        不妨拿种地来打个比方。“位置”好比是一亩亩农田；“**相空间**” 好比是要种的各种庄稼。我们可以给不同的地种上不同的庄稼，这就好比给随机场的每个“**位置**”，赋予相空间里不同的值。所以，通俗点讲，随机场就是在哪块地里种什么庄稼的事情。
        
        如果任何一块地里种的庄稼的种类仅仅与它邻近的地里种的庄稼的种类有关，与其它地方的庄稼的种类无关，那么这些地里种的庄稼的集合，就是一个**马尔科夫随机场**

    !!! note "马尔科夫随机场与图像的关系"

        一维马尔科夫随机过程很好的描述了随机过程中某点的状态只与该点之前的一个点的状态有关系。
        
        对于定义在二维空间上的图像，也可以将它看为一个二维随机场。
        
        说到二维随机场，自然也存在二维马尔科夫随机场。此时我们必须考虑空间上的关系，二维MRF的平面网格结构同样可以较好的**表现图像中像素之间的空间相关性**。

        设 
        
        - $S=\{(i,j)|1\leq i\leq M,1\leq j\leq N\}$ 表示 MN 位置的有限格点集，即随机场中的位置；
        - $\Lambda$ 表示状态空间，即随机场中的相空间；
        - $X=\{x_s|s\in S\}$ 表示定义在 $\forall s \in S$ 处的随机场；
            - $x_s$ 表示在随机场 $X$ 上，状态空间为 $\Lambda$ 的隐状态随机变量，即$x_s\in \Lambda$

        在图像中
        
        - 格点集 $S$ 表示像素的位置 ；
        - $X$ 称为标号场，也可以表示像素值的集合或图像经小波变换后的小波系数集合；
        - $\Lambda$为标号随机变量 $x_s$ 的集合；
        - $L$表示将图像分割为不同区域的数目。


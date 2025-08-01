---
toc: true
tags: AI DiffusionModel
---
本文介绍扩散模型，从最经典的 DDPM 到 一些改进工作，从 实施步骤 到 一些总括性的理论。
### DDPM
扩散模型从自然界中的扩散过程获得灵感，正向的扩散过程是一种 熵增过程，会损失一部分信息量。

将扩散过程和图片生成联系起来是十分天才的想法，为了更好地理解扩散模型，我们需要一些先验知识。
i)时间跨度足够长的扩散过程是极度无序的，几乎任意一个状态出现的概率都是 0，但是当时间间隔足够短地时候，在一定程度上预测未来是可行的。
ii)类似地，虽然一个暂态可能有无数种起源，但是随着逆扩散过程的进行，数据中蕴含的信息越来越多，其路径的随机性也会显著降低。

iii)有意义的图片本身是带有一定的结构性信息的(或者说，方程的约束)

DDPM 分为两个主要部分（正向加噪、反向去噪）。

### 正向加噪
DDPM 中，我们利用下面这个公式对原图片添加噪声：
$\textbf{x}\_{t+1}=a_t \textbf{x}\_{t}+b_t \mathbf{\epsilon}\_{t} \space \space \space (a_t^2 + b_t^2 = 1;a, b, c \ge 0)$ 
其中， $x_t$ 代表时刻 $t$ 的图像，$\epsilon_t \sim \mathcal{N}(0,\mathbf{I})$

因此，我们可以得到：
$\textbf{x}\_{t + 1}=a_t(a_{t-1}\textbf{x}\_{t-1} + b_{t-1}\epsilon_{t-1})+b_t\epsilon_{t}=a_ta_{t-1}\textbf{x}\_{t-1}+a_tb_{t-1}\epsilon_{t-1}+b_t\epsilon_t=a_ta_{t-1}(a_{t-2}\textbf{x}\_{t-2} + b_{t-2}\epsilon_{t-2})+a_tb_{t-1}\epsilon_{t-1}+b_t\epsilon_t=a_ta_{t-1}a_{t-2}\textbf{x}\_{t-2}+a_ta_{t-1}b_{t-2}\epsilon_{t-2}+a_tb_{t-1}\epsilon_{t-1}+b_t\epsilon_t$
不停地进行这个过程，根据观察到的规律可得：

$\textbf{x}\_{t+1}=a_ta_{t-1}\dots a_0\textbf{x}\_0 + a_ta_{t-1}\dots a_1b_0\epsilon_0 + a_ta_{t-1}\dots a_2b_1\epsilon_1 +\dots +a_tb_{t-1}\epsilon_{t-1}+b_t\epsilon_t$

显然，根据正态分布的性质，我们可以试着将后面一长串相互独立的正态分布的和写成一个正态分布，其方差为：

${(a_ta_{t-1}\dots a_1b_0)^2 + (a_ta_{t-1}\dots a_2b_1)^2+(a_tb_{t-1})^2+b_t^2}$ 

这里就可以解释为什么 $a_t^2 + b_t^2=1$ 了，不妨带入这个式子可将方差进行化简：

$1 - (a_0a_1\dots a_t)^2$ 

不妨设：

$\bar{a}_t=a_0a_1 \dots a_t$

则可以容易地得到：

$\textbf{x}\_{t+1}=\bar{a}_t\textbf{x}\_0 + \sqrt{1 - \bar{a}_t^2}\epsilon,\epsilon \sim \mathcal{N}(0, I)$  

不难发现，$t\rightarrow \infty$ 时，$\bar{a}_t\rightarrow 0$，所以最终原始图片数据在不停地增加噪声的情况下会几乎完全服从标准正态分布，就如同滴入水中的墨水最终变得均匀。

**这使得我们可以从纯噪声中试图返回任意可能的图片（所有图片加噪后均变成纯噪声），这种多样性是扩散模型相比传统的GAN的显著优点。**

### 去噪过程

去噪过程的目的是 使得模型可以学会 逆向扩散。在没有其他监督信号的情况下，这种逆向扩散是不可能完美的（信息的损失），模型需要试着对任意一个状态找到其前一个状态（这个过程可能带来信息的构建）。

让我们想象墨水已经均匀混合在水中了，前一刻的状态可能性很多，但是我们往往希望前一刻没有墨水的分布没有那么均匀，这会带来结构性信息（方程、降秩），这种信息是模型根据训练的数据“编造”的。

**如何根据正向的扩散过程数据来训练模型？**

首先我们需要明确一个任务（或者说，损失函数）。

而最朴素的想法是：**假设我们已经拥有了正向扩散的数据（** $\textbf{x}\_{t-1},\textbf{x}\_t$ **），我们希望我们的模型Diffusion Model 接受参数为** $\textbf{x}\_t$ **，输出结果为** $\textbf{x}\_{t-1}$

因此构建损失函数：
$Loss=||\textbf{x}\_{t-1}-Diffusion Model(\textbf{x}\_t)||_2^2$ 

但是扩散模型应当可以为任意时刻的状态预测前一时刻的状态，所以，当输入参数包括时间步 t 的时候，显然效果可以更好（一方面提供更多的信息，另一方面诱导模型在思考时考虑时间步的概念）

我们将正向加噪过程的式子：$\textbf{x}\_{t+1}=a_t \textbf{x}\_{t}+b_t \mathbf{\epsilon}\_{t} \space \space \space (a_t^2 + b_t^2 = 1且均为正数)$ 做一个简单的改写：
$\textbf{x}\_{t}=\frac{1}{a_t}(\textbf{x}\_{t+ 1}-b_t\epsilon_{t})$ 

**一个朴素的想法是，我们的去噪过程应当与加噪过程是一个逆过程，所以可以自然地想到，令：**

$\hat{\textbf{x}}\_t=\frac{1}{a_t}(\textbf{x}\_{t+1}-b_t\epsilon_{\theta}(t + 1, \textbf{x}_{t + 1}))$

其中，$\theta$ 代表模型的参数。

可以发现，我们的模型实际上是在预测**在现在这个状态的基础上，粒子应当做怎样的运动才可以恢复原貌，而不是直接预测原貌**

每个粒子就如同在空间中的一个场运动，这个场是依靠神经网络强大的拟合能力得到的。

不妨将 $\hat{\textbf{x}}\_t=\frac{1}{a_t}(\textbf{x}\_{t+1}-b_t\epsilon_{\theta}(t + 1, \textbf{x}_{t + 1}))$ 带入 损失函数并做一个简单的整理：

$Loss=(\frac{b_{t-1}}{a_{t-1}})^2 \parallel \epsilon_{\theta}(t,\textbf{x}\_{t})-\epsilon_{t-1}\parallel_2^2$  

忽略常数因子，只考虑后半部分：
我们希望平均意义（或者说 期望）上的损失最小，存在无数的路径可以得到 $\textbf{x}_t$。 不妨记路径为 $x$，则期望损失为 **$\sum_{x}p(x)Loss(x)$** 由于这是一个期望，所以我们可以采样 $x$，利用大数定律求解。

最朴素的想法是：
对于原始图片 $\textbf{x}\_0$ ，不断采样 $\epsilon_0,\epsilon_1,\dots,\epsilon_{t-1}$，得到若干路径，并计算 损失 的均值。

这样做存在一个显著的问题：**方差太大（采样的随机变量越多，方差也往往越大）**

所幸的是，根据前面提到的 $\textbf{x}\_{t+1}=\bar{a}\_t\textbf{x}\_0 + \sqrt{1 - \bar{a}\_t^2}\epsilon,\epsilon \sim \mathcal{N}(0, I)$ ，我们可以发现，前面的 $\epsilon_0,\epsilon_1,\dots,\epsilon_{t-2}$ 可以融合成一个 正态分布，不妨记为 $\bar{\epsilon}\_{t-2}$ ，因此我们只需要 采样 $\bar{\epsilon}\_{t-2}, \epsilon_{t-1}$ 即可针对轨迹进行采样，从而得到损失的期望。

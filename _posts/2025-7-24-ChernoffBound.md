# Chernoff Bound 切尔诺夫限
### 马尔可夫不等式 & 切比雪夫不等式
首先，让我们回顾马尔可夫不等式。

$P\lbrace  X \ge \epsilon \rbrace \le \frac{E[  X^r]} {\epsilon^r}\space (\epsilon > 0)$  

Markov 不等式衡量了随机变量大于某个正数 $\epsilon$ 概率的上界（由期望限制）。

将 上式 的 $X$ 替换为另一个随机变量： $\lvert X-E(X)\rvert$ ，再令 $r = 2$ 即可得到切比雪夫不等式。

$P\lbrace \lvert X-E(X)\rvert \ge \epsilon \rbrace \le \frac{D(X)}{\epsilon^2}$ 

这个不等式对于分布没有具体要求，可以用来求得 随机变量 $X$ 离散程度的上界（受 $D(X)$ 约束）。

### 矩生成函数
矩生成函数按如下方式定义：
$M_{X}(s)=\mathbb{E}[e^{sX}]= \int e^{sx}f_X(x)dx$   

其拥有下述性质：
$\frac{d}{ds}M_{X}(s)=\mathbb{E}[Xe^{sX}]$  

### 切尔诺夫限
切尔诺夫限是 马尔可夫不等式 的直接应用。
对于 $\lambda > 0$ ，
$P\lbrace  X \ge \epsilon \rbrace = P\lbrace  e^{\lambda X} \ge e^{\lambda\epsilon} \rbrace \le \frac{\mathbb{E}[e^{\lambda X}]}{e^{\lambda \epsilon}}$ （应用 r = 1 的马尔可夫不等式）

假设 $X_{i}(1\le i\le n)$ 的均值为 $\mu_i(1\le i \le n)$ 且这 n 个随机变量相互独立。
则
$P\lbrace \sum_{i=1}^n(X_i-\mu_i) \ge \epsilon\rbrace \le \frac{\mathbb{E}[e^{\lambda\sum_{i=1}^n(X_i-\mu_i)}]}{e^{\lambda \epsilon}}=\frac{\prod_{i=1}^n\mathbb{E}[e^{\lambda(X_i-\mu_i)}]}{e^{\lambda\epsilon}}$ 

上式被称为 切尔诺夫不等式，用来求 n 个独立的随机变量和 大于 某个整数概率的上界。

### 例子：bound l2 norm
假设 $X_{i}\sim \mathcal{N}(0, 1)(1\le i\le n)$  且这 n 个随机变量相互独立。
则二范数的平方 $Y=\sum_{i=1}^n X_i^2$ 。应用切尔诺夫不等式易知

$P\lbrace \sum_{i=1}^n X_i^2 \ge \epsilon\rbrace \le \frac{\prod_{i=1}^n M_{X_i^2}(\lambda)}{e^{\lambda\epsilon}}$ 

又由于卡方分布的矩生成函数为 $\frac{1}{\sqrt{1-2\lambda}}$ 。

所以最后有：
$P\lbrace\sum_{i=1}^n X_i^2 \ge \epsilon\rbrace\le \frac{1}{(1-2\lambda)^{\frac{n}{2}}e^{\lambda\epsilon}}$ 对任意的 $\lambda > 0$ 均成立。

**我们还可将不等式右侧看作 $\lambda$ 的函数，找到使得函数值最小的 $\lambda_0$ ，进一步加强不等式。**

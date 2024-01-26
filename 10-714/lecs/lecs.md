## 01: Intro:
why study deep learning systems?

- to provide you an introduction to the functioning of modern deep learning systems.
- not just for the "big players" 
### reasons:
1. developing your own new frameworks for specific tasks
2. understanding how the deep learning system works
3. deep learning systems are fun

## 02: Machine learning lookback
### basic of machine learning
as a data-driven programming method.

supervised ML approach: collect a training set of images with known labels and feed these into a machine learning algorithm.
#### Linear hypthesis function:
map $x \in R^{n}$ to k-dimensional vectors:
$$
h: R^{n} -> R^{k}
$$
a linear hypothesis function uses a linear operator for this transformation:
$$
h_{\theta}(x) = \theta^{T}x
$$
while $\theta \in R ^{n * k}$ 

unfamiliar part:
- softmax/cross-entropy loss
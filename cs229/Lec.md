## Lec1:
Def: give computers the ability to learn without being explicitly programmed.

讲了一下监督学习，非监督学习的一些概念，相对轻松的一节课。

## Lec2:
For the **cost function**.
$J(\theta) = \frac{1}{2} \sum^{n}_{i = 1} (h_{\theta} (x^(i)) - y^(i))^{2}$

$\theta_j := \theta_j - \alpha \frac{\partial}{\partial \theta_j} J(\theta)$

Least Mean Squares:

$\frac{\partial}{\partial \theta_j} J(\theta) = (h_\theta (x) - y)x_j$

Gradient Descent: Batch vs Stochastic
- Full Dataset vs Only One Training Data (each step you pick randomly a data)

## Lec3:
Linear algebra review: Easy.
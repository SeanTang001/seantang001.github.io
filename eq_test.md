---
layout: default
title: Sean Tang
---


### Question 1: Show that minimizing the Mean Square Error objective is equivalent to estimating the mean parameter of a Gaussian random variable from a set of samples through Maximum Log Liklihood Estimation.  

Answer:

By definition, MSE is:

$$
\arg \min E[(\hat{Y}-Y)^2]
$$

Suppose we model a data distribution as a gaussian. We want to find the best mean parameter for this gaussian that matches the data distribution. To do this, we can try to maximize our Log Liklihood, which is a numerically stable version of Maximizing Liklihood.

By definition, Maximum Log Likelihood is:

$$
\theta_{ML} = \arg_{\theta}\max \frac{1}{m}\sum_{i=1}^m log(p_{model}(y^i | x^i, \theta))
$$

If we assume $$p_{model} = N(\mu,\sigma^2)$$, where $$\theta_{ML} = (\mu,\sigma^2)$$ (we assume $$\sigma^2$$ is pre-chosen), or :

$$
N(Y^i; \mu, \sigma^2) = \frac{1}{\sqrt{2\pi \sigma^2}} \exp(-\frac{1}{2}\frac{(y^i - \mu)^2}{\sigma^2})
$$

then when plugged into maximum log likelihood, we have:

$$
\frac{1}{m}\sum_{i=1}^m log(p_{model}(y^i | x^i, \theta)) = \frac{1}{m}\sum_{i=1}^m(0 - \frac{1}{2} \log(2\pi) + 0 - \log(\mu) - \frac{1}{2}\frac{(\hat{y}^i - y)^2}{\sigma^2})
$$

And with the first few terms being constants on the right-hand size, we can only optimize the very last term which is a function of $$\mu$$. Since it is negative, minimizing that term is equivalent to maximizing the overall expression.

Thus by *miniziming* the MSE, we actually achieve the goal of *maximizing* the overall mean log-likelihood for the case of estimating a guassian mean parameter.
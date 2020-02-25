# An Introduction to Sparse Gaussian Processes 
*A deep learning first explanation by [Joost van Amersfoort](https://joo.st)*

## Introduction

Until about seven years ago, Gaussian Processes (GPs) were roughly as popular as
deep learning, according to [Google
Trends](https://trends.google.com/trends/explore?date=all&q=deep%20learning,gaussian%20process).
Since then deep learning has seen a 30x increase in interest, while interest in
GPs remained unchanged (although the UK is 3x more interested relative to the
US!). Despite the community not increasing as fast, there's been steady research
happening with GPs. In this blog, I'll explain one of the main research advances
in GPs from a deep learning first perspective by showing it can be derived
similarly to the Variational Auto-Encoder (VAE)!

GPs come out-of-the-box with uncertainty, some interpretability, and rigorous
mathematical foundations. So it's not surprising that they remain a topic of
great interest to research! Compared to deep learning, GPs are more difficult to
scale, both in terms of training data set size, and the dimensionality of the
data itself. On both fronts the GP community is making progress, but for this
blog I'll focus on one of the proposed solutions to scaling to large data set
sizes: variational inference in GPs. I'll use notation from the deep learning
literature and attempt to rederive a famous result in GPs using steps that
resemble deriving the VAE objective. I'll also include as much intuition as
possible and by doing this I hope to make the GP literature more accessible for
those with only experience in deep learning.

For the purpose of this blog, I assume that you have some familiarity with the
[Variational Auto-Encoder](https://arxiv.org/abs/1312.6114) and variational
inference in general. A good reference to get up to speed with both those
concepts is this
[blog](https://jaan.io/what-is-variational-autoencoder-vae-tutorial/) by Jaan
Altosaar, specifically the section titled "The probability model perspective".
Familiarity with GPs is not necessary, but if you would like a gentle
explanation with many beautiful visualisations then the [Distill article on
GPs](https://distill.pub/2019/visual-exploration-gaussian-processes/) is great.

## Variational Auto encoders

Let's start by recalling the assumptions that underlie the VAE. Our first
assumption is that there is a random variable $Z$ from which it is possible to
generate our data set $X$. Inversely, if we have a data point $x$, we also want
to be able find ("infer") its corresponding $z$. To make things interesting, we
restrict $z$ such that it is lower dimensional than our data points and easy to
sample from.

It turns out that if we were given the $x$ and $z$ pairs, then it's
straightforward to train a model to predict $x$ from $z$, i.e. $p(x|z)$. However
we only have our data set $X$, and we need to figure out how to structure $Z$,
such that it becomes usable, i.e. $p(z|x)$, and afterwards we can model
$p(x|z)$. Unfortunately, computing the true value of $p(z|x) is
intractable (it requires evaluating all possible values of $z$!). Instead we
will try to find a good approximate distribution, which we call q. We decide
that q will be a Gaussian distribution parametrised by $\theta$, which lends
itself well for applying the [reparametrisation
trick](https://arxiv.org/abs/1312.6114) and also allows easy sampling of new
data points (fun!).

**The process of finding the best approximation q(z|x) to p(z|x) is called Variational Inference!**

We measure the difference between $p(z|x)$ and $q(z|x)$ using the [KL
divergence](https://en.wikipedia.org/wiki/Kullback%E2%80%93Leibler_divergence).

Rewriting the KL divergence we obtain:
$$
\begin{aligned}
&\operatorname{argmin}_\theta KL(q_\theta(Z|X)|p(Z|X)) \\
&= \int_Z q_\theta(Z|X) \log \frac{q_\theta(Z|X)}{p(Z|X)} \\
&= \int_Z q_\theta(Z|X) \log \frac{q_\theta(Z|X)p(X)}{p(Z,X)} \\
&= \int_Z q_\theta(Z|X) \left [ \log \frac{q_\theta(Z|X)}{p(Z,X)} + \log p(X) \right ]\\
&= \log p(X) + \int_z q_\theta(Z|X) \log \frac{q_\theta(Z|X)}{p(Z,X)}. \\
\end{aligned}
$$

Since $p(X)$ is independent of $Z$, we can take it out of the integral and
because $q$ is a probability distribution (which integrate to 1), we simply have
$p(X)$ left. Also it doesn't depend on $\theta$, so we can disregard it for this
minimisation. We now take the integral, express it as an expectation and flip
the sign (turning the argmin into and argmax) and obtain the evidence lower
bound (ELBO):

$$
\begin{aligned}
&\operatorname{argmax}_\theta \mathbb{E}_{z \sim q_\theta(Z|X)}[- \log \frac{q_\theta(Z|X)}{p(Z,X)}] \\
&= \mathbb{E}_{z \sim q_\theta(Z|X)}[- \log q_\theta(Z|X) - \log p(X|Z) + \log(p(Z))] \\
&= \mathbb{E}_{z \sim q_\theta(Z|X)}[\log p(X|Z)] - KL[q_\theta(Z|X) || (p(Z)]. \\
\end{aligned}
$$

The last equation is one we can work with! $\log p(X|Z)$ is the distribution we
are interested in and we will also model it using a neural network, but with
parameters $\phi$. We choose $p(Z)$ to be $N(0,I)$, which makes the KL with $q$
easy to compute. The only problem we have left is how to compute the expectation
over $Z$!

### Reparametrisation trick

To evaluate the gradient of the expectation, we would like to use samples of
$q$. However, sampling from $q$ is not a differentiable operation and won't
allow us to backpropagate back to $q$. The key insight from the
reparametrisation trick (introduced concurrently by [Kingma et
al.](https://arxiv.org/abs/1312.6114), [Rezende et
al.](https://arxiv.org/abs/1401.4082), and [Titsias et
al.](http://proceedings.mlr.press/v32/titsias14.pdf)) is that we can instead
sample from a standard Gaussian $N(0,1)$ and transform its samples in a
deterministic way such that are samples from q:

$$z = g(\epsilon) = \mu + \sigma \cdot \epsilon, \text{ and } \epsilon \sim N(0,1)$$

With $\mu$ and $\sigma$ being outputs of the neural network $q_\theta(z|x)$.
Modelling the variational parameters as a function of the observation $x$ is also referred to as
"amortised variational inference". The previous equation changes into:

$$
\mathbb{E}_{\epsilon \sim N(0,1)}[\log p_\phi(X|g(\epsilon))] -
KL[q_\theta(g(\epsilon)|X) || p(Z)].
$$

This equation is fully differentiable, so we can optimise it using
backpropagation! We have derived the ELBO and a nice, easy, way to optimise it.
With this result in hand, we will go look at the situation in GPs.

## Gaussian Processes

We start by looking at the definition of what is sometimes called the "full"
Gaussian process and continue with the more practical sparse Gaussian process.

### Definition

A Gaussian process is defined by a mean and covariance function, respectively
$M$ and $K$, of the data set $X$:

$$
f(X) \sim GP(M(X), K(X, X)).
$$

In this blog, we assume our GP has zero mean and replace $M(X)$ by $0$. This
is not a limitation of GPs, but it makes the notation easier. The covariance
function, also referred to as kernel, is where we can apply any knowledge we
have of the data we are modelling. A common choice is the squared exponential
kernel:

$$k(x, x') = \sigma^2\exp\left(-\frac{(x - x')^2}{2\ell^2}\right)$$

which has two hyper parameters: the length scale $\ell$ and the output variance
$\sigma^2$. The length scale represents an assumption of the amount of variation
in your inputs, the higher the value the smoother the underlying function is
assumed to be. The output variance is a scale factor, if it is high then we
scale the squared difference down, if it is low then we scale it up. The core
assumption in the squared exponential kernel is that data points that are close
in input ("similar"), have high covariance ("similar" labels). There are many
other options, such as linear kernels, periodic kernels and combinations are
also possible. See David Duvenaud's excellent [kernel
cookbook](https://www.cs.toronto.edu/~duvenaud/cookbook/) for more.

Further we assume that we can only observe noisy labels $Y$ of data, and define
$F$ as the function values at $X$: $F = f(X)$:

$$
Y \sim N(Y | F, \sigma^2I).
$$

And we are interested in finding the predictive distribution for new data point
$x_*$, because it will allow us to make prediction for unseen data:

$$
P(f_*|x_*, X, Y).
$$

The joint distribution of the noisy labels and unseen data is:

$$
\begin{bmatrix}
Y \\
f_*
\end{bmatrix}
\sim N\left(0, \begin{bmatrix}
K(X,X)+\sigma^2I & K(X,x_*) \\
K(x_*,X) & K(x_*,x_*)
\end{bmatrix}\right).
$$

With $K(x_*, X)$ a vector of kernel distances between the test data point and
the training data. For more detail refer to equation 2.21 of the [GPML
book](http://www.gaussianprocess.org/gpml/chapters/RW2.pdf). If we condition
on $Y$ (the observed data), we obtain the equation for the mean and covariance
of the predictive distribution:

$$
\begin{aligned}
\mu &= K(x_*,X)[K(X,X) + \sigma^2_nI]^{-1}Y \\
\Sigma &= K(x_*, x_*)-K(x_*, X)[K(X,X) + \sigma^2I]^{-1}K(X, x_*).
\end{aligned}
$$

The equation for $\Sigma$ contains an inverse of the covariance matrix of the
data set. The complexity of that computation grows cubically, $O(n^3)$, with the
data set. This is infeasible to compute for many datasets. One way to go around
this limitation is to exploit the fact that there is often a lot of redundancy
in the training set. By using only the points that are most informative we can
drastically reduce computation. GPs that work on a reduced data set are called
"Sparse Gaussian Processes".


### Sparse Gaussian Processes

The first Sparse GPs relied on smartly picking a subset of the training data,
this subset is also referred to as the "inducing points", and computing the
inverse on those. This allows us to reduce the computation from depending on the
number of data points $O(n^3)$ to depending on the number of $m$ induced points
$O(m^3)$. See [Quiñonero-Candela  et
al.](http://www.jmlr.org/papers/volume6/quinonero-candela05a/quinonero-candela05a.pdf)
for a nice overview.

Later methods removed the reliance on selecting an actual subset of the data
points ([Snelson et al.](http://mlg.eng.cam.ac.uk/zoubin/papers/nips05spgp.pdf))
by introducing the concept of "pseudo" inducing points. It was
[Titsias](http://proceedings.mlr.press/v5/titsias09a/titsias09a.pdf) who
introduced the first variational approximation to optimising sparse GPs, where
the pseudo inducing points are optimised as variational parameters. In contrast
to previous approaches, the "Titsias approximation" made optimising the pseudo
inducing points part of the objective, providing a unified and rigorous way to
optimise them.

The definition of sparse GPs is a bit different from full GPs. Below we follow the
derivation of [Hensman et
al.](http://www.auai.org/uai2013/prints/papers/244.pdf):

$$
\begin{aligned}
p(Y|F) &= N(Y|F,\sigma^2 I) \\
p(F|U) &= N(F| K(X,Z)K(Z,Z)^{-1}U,\tilde{K}) \\
p(U) &= N(U| 0, K(Z,Z))
\end{aligned}
$$

With $U$ containing values of the function $f(\cdot)$ at the (pseudo-) inducing
points $Z = \{z_i\}_{i=1}^m$ and $\tilde{K} = K(X,X) - K(X,Z)K(Z,Z)^{-1}K(Z,X)$.
Notice that the matrix inverse is on $K(Z,Z)$ now, making it dependent on the
number of inducing points we pick and it no longer grows with the data set. The
difference between this approximation and the full GP is given by the following
KL divergence:

$$
\begin{aligned}
KL(p(F|U) || p(F|U,Y))
\end{aligned}
$$

If $U$ contains all $n$ training function values, meaning that $U = F$, then
$K(X,Z) = K(Z,Z) = K(X,X)$ and so $\tilde{K} = 0$. In that case the KL
divergence is zero, because both distributions are degenerate and we recover the
full GP. However we lost the computational gain, because the matrix inverse is
of the order of the original data set again.

## Finding an approximate posterior

We introduce an approximate posterior distribution $q(F|U)$ and attempt to
minimise its distance to the true posterior $p(F|U,Y)$. We start from a similar
divergence as in the VAE and show we can find a bound we can directly optimise with
respect to the pseudo inducing points.

The KL we are interested in minimising is:

$$
KL(q(F|U) || p(F|U,Y))
$$

Intuitively, we can interpret this as the difference between having only access to
the inducing points as compared to having access to both the inducing and the
original data points. If this difference is small, then q holds all the
information we need.

Like before, we will write out the KL divergence in (now) familiar components:

$$
\begin{aligned}
=& - \int_F q(F|U) \log \frac{p(F|U,Y)}{q(F|U)} \\
=& - \int_F q(F|U) \log \frac{p(Y|U,F)p(F)}{q(F|U)p(Y)} \\
=& - \int_F q(F|U) \left [ \log p(Y|U,F) + \log p(F) - \log q(F|U) - \log p(Y) \right ] \\
=& \log p(Y) - \mathbb{E}_{q(F|U)}[\log p(Y|U,F)] - KL[q(F|U) || p(F)]
\end{aligned}
$$

We end up with three terms: the marginal log-likelihood over $Y$ instead of $X$
in the VAE, the likelihood, and the KL divergence between our approximate
posterior and the prior. To evaluate the likelihood, we note that given the
inducing points, the function values factorise:

$$
\begin{aligned}
p(Y|U, F) = \prod_i p(Y_i|F_i)
\end{aligned}
$$

The approximate posterior $q(F|U)$ is itself a Gaussian Process, with a mean and
variance function that depend on the inducing variables. Following this
[blog](https://www.prowler.io/blog/sparse-gps-approximate-the-posterior-not-the-model)
by James Hensman, we pick a particular mean and variance function for q, such
that the ELBO is computable:

$$
\begin{aligned}
\mu &= K(x_*, Z)K(Z,Z)^{-1}m \\
\sigma(x_*, x_*) &= k(x_*,x_*) - k(x_*, Z) \left[ k(Z,Z)^{-1} -k(Z,Z)^{-1} \Sigma k(Z,Z)^{-1} \right ] k(Z, x_*)
\end{aligned}
$$

With $m$, $\Sigma$, and $Z$, all variational parameters that can be optimised
through gradient descent. Interestingly, this means that the inducing points
themselves, $Z$, have become part of the objective. Making predictions using
$q(F|U)$ now depends explicitly on the set of inducing points, which is very
different from deep neural networks where we only update the parameters of the
network itself. For a detailed derivation, see also [this
tutorial](https://arxiv.org/abs/1402.1412) by Yarin Gal and Mark van der Wilk.


## Conclusion

In this blog I have shown that performing variational inference in VAEs and GPs
is very similar. In both cases we start out with an intractable
equation and introduce a new distribution to create a tractable approximation.
We try to find the best approximation to the original problem (or alternatively
minimise the KL divergence!).

If you found this interesting and are curious to find out more about GPs, then
there is a considerable amount of literature available. I can recommend the
[deep GP](http://proceedings.mlr.press/v31/damianou13a.pdf) paper and
[variational
extensions](http://papers.nips.cc/paper/7045-doubly-stochastic-variational-inference-for-deep-gaussian-processes)
of it. There are many interesting opportunities of combining the scalability of
deep learning with the mathematical foundations of GPs and no doubt a lot more
research will be done towards that goal!
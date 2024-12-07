
<!-- README.md is generated from README.Rmd. Please edit that file -->

# epiwave.mapping

<!-- badges: start -->

[![Lifecycle:
experimental](https://img.shields.io/badge/lifecycle-experimental-orange.svg)](https://lifecycle.r-lib.org/articles/stages.html#experimental)
<!-- badges: end -->

Prototype R package and example code for computationall-yefficient,
fully-Bayesian semi-mechanistic spatio-temporal mapping of disease
infection incidence simultaneously from spatially-aggregated (e.g. at
health facility level) clinical case count timeseries, and targeted
infection prevalence surveys at specific point locations. This model and
software is based on ideas in the epiwave R package and uses some
functions implemented there, but it currently relies on the user to
develop their own model using the greta R package, rather than providing
wrapper functions to set up the model for the user.

## Installation

You can install the package from github, using the `remotes` package:

``` r
remotes::install_github("idem-lab/epiwave.mapping",
                        dependencies = TRUE)
```

## Model

Our main aim is to model, and generate spatio-temporal predictive maps
of, variation in *infection incidence*: the average number of new
infections per time period, per member of the population. We model the
logarithm of infection incidence with a space-time Gaussian process.
Since infection incidence is rarely perfectly observed, we infer it from
two independent and complementary datastreams:

- **Clinical case timeseries**: Counts of the numbers of cases reported
  over time, but potentially spatially aggregated across e.g. the
  catchment areas of health facilities, or the entire region, rather
  than at spatially precise coordinates

- **Infection prevalence survey points**: Random samples of individuals
  at a specific point location, with the number tested and number
  positive recorded. Typically these data are available only very
  infrequently.

Since neither datastream provides a direct estimate of infection
incidence, we model the generative observation process yielding each
datastream, using informative priors for key parameters wherever
possible. A number of options are available for
computationally-efficient space-time Gaussian process (GP) modelling for
applications of this type. Below we implement an approach well-suited to
our health facility catchment observation model and fully Bayesian
inference using Hamiltonian Monte Carlo: a separable space-time GP
employing a block-circulant embedding for the spatial process. This is
similar to the approach used in the `lgcp` R package, and detailed in
Diggle et al. (2013). We note that the clinical incidence observation
model we employ is a particular case of a Log-Gaussian Cox process. Our
contribution is linking this to prevalence survey data in a practical
and easily-extensible way.

### Clinical incidence model

We model the observed number of clinical cases $C_{h,t}$ of our disease
of interest in health facility $h$ during discrete time period $t$ as a
Poisson random variable:

$$
C_{h,t} \sim \text{Poisson}(\hat{C}_{h,t})
$$ The expectation of this poisson random variable (the
modelled/expected number of clinical cases) is given by a weighted sum
of (unobserved but modelled) expected pixel-level clinical case counts
$\hat{C}_{l,t}$ at each of $L$ pixel locations $l$:

$$
\hat{C}_{h,t} = \sum_{l=1}^{L}{\hat{C}_{l,t}w_{l,h}}
$$

where weights $w_{l,h}$ give the ‘membership’ of the population in each
pixel location to each health facility, such that
$\sum_{h=1}^H w_{l,h} = 1$. In practice this could be either a
proportional (fractions of the population attend different health
facilities) or a discrete (the population of each pixel location attends
only one nearby health facility) mapping.

Within each pixel location, we model the unobserved clinical case count
$\hat{C}_{l,t}$ from the modelled number of new infections
$\hat{I}_{l,t'}$ at the same location, during previous time periods $t'$
to account for for non-negative integer delays ($t-t'$ up to maximum of
$\tau_\pi$) from infection to diagnosis and reporting, and from the
fraction of infections $\gamma_{l,t'}$ at that location and time that
would result in a recorded clinical case:

$$
\hat{C}_{l,t} = \sum_{t-t' = 0}^{\tau_\pi}{\gamma_{l,t'} \, \pi_1(t-t') \, \hat{I}_{l,t'}}
$$

where $\pi(\Delta_t)$ gives the discrete and finite (support on
$\Delta_t \in (0, \tau_\pi)$) probability distribution over delays from
infection to reporting, indexed on the time-periods considered in the
model. This temporal reweighting to account for a distribution over
possible delays can be considered as a ‘discrete-time convolution’ with
$\pi(\Delta_t)$ the ‘kernel’. Below we discuss efficient methods for
computing these convolutions in greta.

### Infection prevalence model

We model the observed number of individuals who test positive for
infection $N^+_{l,t}$ in an infection prevalence survey at location $l$
at time $t$ as a binomial sample, given the number of individuals tested
$N_{l,t}$, and the modelled prevalence of infections $\hat{p}_{l,t}$ in
the entire population at that time/place:

$$
N^+_{l,t} \sim \text{Binomial}(N_{l,t}, \hat{p}_{l,t})
$$ Similarly to clinical incidence, we model the infection prevalence at
a given location and time as a discrete-time convolution over previous
infection counts, divided by the total population in the pixel, $M_l$:

$$
\hat{p}_{l,t} = \frac{1}{M_l}\sum_{t-t' = 0}^{\tau_q}{q(t-t')\hat{I}_{l,t'}}
$$

In this case, the kernel $q(\Delta_t)$ is not a probability
distribution, but the proportion of individuals who would test positive
to the diagnostic used in the survey in the $\Delta_t$ time period
post-infection. This function can be estimated from empirical data on
how the test sensitivity and duration of infection vary over time since
infection. Convolution of the number of new infections in each previous
time period with the fraction that would test positive in the the
contemporary time period gives an estimate of the number of
positive-testing people in the population at location $l$, at time $t$.
Dividing this by $M_l$, the whole population of location $l$, yields an
estimate of the population proportion testing positive - the parameter
of the binomial distribution.

Note that our definition of $\hat{p}_{l,t}$ is as the population
prevalence of infections *that would test positive using that diagnostic
method* rather than the true fraction infected/infectious at any one
time. We also assume here that tests have perfect specificity, though
the model can easily be adapted to situations where that is not the
case.

### Infection incidence model

The expected number of new infections $\hat{I}_{l,t'}$ in location $l$
during time period $t$ is modelled as the product of the population
$M_l$ at that location, and the infection incidence $f_{l,t}$ at that
location and time:

$$
\hat{I}_{l,t'} = M_l \hat{f}_{l,t}
$$

Whilst we mechanistically model the observation processes yielding our
data types, we employ a geostatistical approach to modelling
spatio-temporal variation in infection incidence, with spatio-temporal
covariates $\mathbf{X}_{l,t}$ and a space-time random effect
$\epsilon_{l,t}$ with zero-mean Gaussian process prior:

$$
\text{log}(f_{l,t}) = \alpha +\mathbf{X}_{l,t} \beta + \epsilon_{l,t} \\
\epsilon \sim GP(0, \mathbf{K})
$$

where $\alpha$ is a scalar intercept term, $\beta$ is a vector of
regression coefficients against the environmental covariates, and
$\mathbf{K}$ is the space-time covariance function of the Gaussian
process over random effects $\epsilon_{l,t}$.

There are many choices of space-time covariance structure for
$\mathbf{K}$, though we use a separable combination of an isotropic
spatial covariance function with a temporal covariance function, to
enable the use of a range of computationally efficient simulation and
calculation methods:

$$
K_{l,t,l',t'} = \sigma^2 \, K_{\text{space}}(||l-l'||; \phi) \, K_{\text{time}}(|t-t'|; \theta)
$$ where $\sigma^2$ is the marginal variance (amplitude) of the
resultant Gaussian process, $K_{\text{space}}(.; \phi)$ is an (unit
variance) isotropic spatial kernel on euclidean distances $||l-l'||$
with parameter $\phi > 0$ controlling the range of spatial correlation,
and $K_{\text{time}}(|t-t'|; \theta)$ is a a temporal kernel on time
differences $|t-t'|$, with parameter $\theta$ controlling the range of
temporal correlation. Again for computational reasons, we prefer
Markovian kernels for $K_{\text{time}}$.

## Computational approaches

### Inference

We fit the model via fully Bayesian inference, using MCMC (Hamiltonian
Monte Carlo) in the greta R package. Whilst a number of alternative
inference algorithms have been proposed and widely adopted for Bayesian
and non-Bayesian estimation of spatio-temporal Gaussian process models
(e.g. INLA), in out case, the aggregation of clinical cases across
catchments (a feature required by the model structure) leads to a
non-factorising likelihood, which nullifies many of the computational
advantages of that method. Further, the mapping from the Gaussian
process to the expectations of the two sampling distributions is
non-linear (due to sums of exponents in various places), requiring
computationally costly linearisation of the non-linearities. By
employing a fully-Bayesian MCMC approach, and focussing on
computationally-efficient model and implementation choices, rather than
approximations, we are able to estimate the model parameters relatively
quickly, with full treatment, propagation, and quantification of model
uncertainty, and retaining the ability to easily modify and interrogate
the model.

### Discrete-time convolution

This discrete convolutions (weighted sums over previous time points)
used to compute $\hat{C}_{l,t}$ and $\hat{p}_{l,t}$ can be
computationally intensive, depending on the number of time periods
modelled. A number of efficient computational approaches exist to
overcome these, and the optimal approach to use in greta depends on the
size of $\tau^{max}$: the maximum duration of the delay in terms of the
number of timeperiods being considered. Where $\tau^{max}$ is relatively
large (say, $\tau^{max} > 3$), implementation of the convolution as a
matrix multiply is likely to be the most efficient in greta. If
$\tau^{max} > 3$ and the number of time periods being modelled is much
larger than $\tau^{max}$, implementing this as a sparse matrix multiply
(skipping computation on zero elements of the convolution matrix) is
likely to be most efficient. Where $\tau^{max}$ is small (say,
$\tau^{max} \leq 3$) the convolution can instead be computed with a sum
of dense vectorised additions and subtractions. These convolution
approaches are implemented in this package, and demonstrated below.

### Gaussian process simulation

Naive implementation of the full Gaussian process (GP) model is
typically very computationally intensive, due to the fact that the most
expensive step (inverting a dense covariance matrix) scales cubically
with the number of evaluation points. In the model we propose (and as in
a Log-Gaussian Cox process), we need to evaluate the GP at every pixel
location and time period in the study frame, so that we can aggregate
the expected clinical case count to compute the likelihood. This results
in a computationally impractical algorithm for all but the smallest
study frames.

One solution would be to employ an approximation to the full GP,
evaluated at only a subset of locations and times in the study frame,
and approximating the clinical case count calculation with some smaller
finite sum. Candidate GP approximation approaches include the SPDE
approximation to a Matern-type spatial kernel, on a computational mesh
(as used in INLA, Lindgren *et al.*, 2011), a sparse GP method over a
limited set of inducing points (Quinonera-Candela *et al.*, ?; see
`greta.gp`), or one of the closely-related penalised spline methods (see
`greta.gam`).

An appealing alternative in this case is to compute the full spatial GP
for every pixel in a regular grid (the same we use to record spatial
covariate values), by exploiting the block-circulant structure in the
resulting spatial covariance matrix. This requires using projected
coordinates, and expanding the spatial study frame (by a factor of 2 in
each dimension) to map the projection onto a torus in such a way that
the distances between pairs of pixels are preserved. This approach
enables simulation of the GP across all pixels, for a given time period,
via the fast fourier transform (FFT) - which scales only linearly with
the number of locations considered (? check when have wifi). This
enables us to simulate values of the spatial GP across all pixels very
cheaply, with no need to approximate the GP. In separable combination
with a Markovian temporl kernel, this enables very rapid inference.

## Example application

We demonstrate the model with application to mapping the infection
incidence of malaria in ? In this case, we use clinical incidence data
from ? and infection prevalence data from ?

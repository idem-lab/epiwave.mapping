
- [epiwave.mapping](#epiwavemapping)
  - [Installation](#installation)
  - [Model](#model)
    - [Clinical incidence model](#clinical-incidence-model)
    - [Infection prevalence model](#infection-prevalence-model)
    - [Infection incidence model](#infection-incidence-model)
    - [Infection prevalence - clinical incidence
      relationship](#infection-prevalence---clinical-incidence-relationship)
  - [Computational approaches](#computational-approaches)
    - [Inference](#inference)
    - [Discrete-time convolution](#discrete-time-convolution)
    - [Gaussian process simulation](#gaussian-process-simulation)
  - [Example application](#example-application)
    - [Simulating data](#simulating-data)
    - [Model specification using
      greta](#model-specification-using-greta)
    - [Model fitting and prediction](#model-fitting-and-prediction)
    - [Comparison with simulated
      truth](#comparison-with-simulated-truth)

<!-- README.md is generated from README.Rmd. Please edit that file -->

# epiwave.mapping

<!-- badges: start -->

[![Lifecycle:
experimental](https://img.shields.io/badge/lifecycle-experimental-orange.svg)](https://lifecycle.r-lib.org/articles/stages.html#experimental)
<!-- badges: end -->

Prototype R package and example code for computationally efficient,
fully Bayesian semimechanistic spatio-temporal mapping of disease
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
possible. In order to model the full spatial process generating these
data, the model requires estimation of the infection incidence across
every pixel in a raster grid covering the region of interest - incidence
across all cells is aggregated-up to compute the catchment-level case
counts. In this way, the model fully accounts for spatial variation in
environmental conditions (represented by regression against
environmental predictors) and in the residual space-time process across
all cells. This model structure leads to a non-linear and
non-factorising likelihood which negates many of the computational
tricks used in the INLA software for fitting geostatistical models that
can be structured as Gaussian Markov random fields. Instead we employ
fully Bayesian inference using Hamiltonian Monte Carlo (providing
significant flexibility in model specification) and computational
approximations to evaluating the Gaussian process across the entire
spatial extent, of which a number of options are available.

Below illustrate two methods suited to mapping a separable space-time
Gaussian process: simulation with a block-circulant embedding, and with
a sparse Gaussian process approximation. The former is similar to the
approach used in the `lgcp` R package, and detailed in Diggle et
al. (2013). We note that the clinical incidence observation model we
employ is a particular case of a Log-Gaussian Cox process. Our
contribution is accounting for delays in reporting, and linking model
this to prevalence survey data, in a practical and easily-extensible
framework.

### Clinical incidence model

We model the observed number of clinical cases
![C\_{h,t}](https://latex.codecogs.com/png.latex?C_%7Bh%2Ct%7D "C_{h,t}")
of our disease of interest in health facility
![h](https://latex.codecogs.com/png.latex?h "h") during discrete time
period ![t](https://latex.codecogs.com/png.latex?t "t") as a Poisson
random variable:

![C\_{h,t} \sim \text{Poisson}(\hat{C}\_{h,t})](https://latex.codecogs.com/png.latex?C_%7Bh%2Ct%7D%20%5Csim%20%5Ctext%7BPoisson%7D%28%5Chat%7BC%7D_%7Bh%2Ct%7D%29 "C_{h,t} \sim \text{Poisson}(\hat{C}_{h,t})")

The expectation of this Poisson random variable (the modelled/expected
number of clinical cases) is given by a weighted sum of (unobserved but
modelled) expected pixel-level clinical case counts
![\hat{C}\_{l,t}](https://latex.codecogs.com/png.latex?%5Chat%7BC%7D_%7Bl%2Ct%7D "\hat{C}_{l,t}")
at each of ![L](https://latex.codecogs.com/png.latex?L "L") pixel
locations ![l](https://latex.codecogs.com/png.latex?l "l"):

![\hat{C}\_{h,t} = \sum\_{l=1}^{L}{\hat{C}\_{l,t}w\_{l,h}}](https://latex.codecogs.com/png.latex?%5Chat%7BC%7D_%7Bh%2Ct%7D%20%3D%20%5Csum_%7Bl%3D1%7D%5E%7BL%7D%7B%5Chat%7BC%7D_%7Bl%2Ct%7Dw_%7Bl%2Ch%7D%7D "\hat{C}_{h,t} = \sum_{l=1}^{L}{\hat{C}_{l,t}w_{l,h}}")

where weights
![w\_{l,h}](https://latex.codecogs.com/png.latex?w_%7Bl%2Ch%7D "w_{l,h}")
give the ‘membership’ of the population in each pixel location to each
health facility, where each case at a given location
![l](https://latex.codecogs.com/png.latex?l "l") has probability
![w\_{l,h}](https://latex.codecogs.com/png.latex?w_%7Bl%2Ch%7D "w_{l,h}")
of reporting at health facility
![h](https://latex.codecogs.com/png.latex?h "h"), and therefore
![\sum\_{l=1}^L w\_{l,h} = 1](https://latex.codecogs.com/png.latex?%5Csum_%7Bl%3D1%7D%5EL%20w_%7Bl%2Ch%7D%20%3D%201 "\sum_{l=1}^L w_{l,h} = 1").
In practice this could be either a proportional (fractions of the
population attend different health facilities) or a discrete (the
population of each pixel location attends only one nearby health
facility) mapping.

At each pixel location, we model the unobserved clinical case count
![\hat{C}\_{l,t}](https://latex.codecogs.com/png.latex?%5Chat%7BC%7D_%7Bl%2Ct%7D "\hat{C}_{l,t}")
from the modelled number of new infections
![\hat{I}\_{l,t'}](https://latex.codecogs.com/png.latex?%5Chat%7BI%7D_%7Bl%2Ct%27%7D "\hat{I}_{l,t'}")
at the same location, during previous time periods
![t'](https://latex.codecogs.com/png.latex?t%27 "t'"), from the fraction
of infections
![\gamma\_{l,t'}](https://latex.codecogs.com/png.latex?%5Cgamma_%7Bl%2Ct%27%7D "\gamma_{l,t'}")
at that location and previous time period that would result in a
recorded clinical case, and a probability distribution
![\pi(t-t')](https://latex.codecogs.com/png.latex?%5Cpi%28t-t%27%29 "\pi(t-t')")
over delays ![t-t'](https://latex.codecogs.com/png.latex?t-t%27 "t-t'")
from infection to diagnosis and reporting:

![\hat{C}\_{l,t} = \sum\_{t-t' = 0}^{\tau\_\pi}{\hat{I}\_{l,t'} \\ \gamma\_{l,t'} \\ \pi(t-t')}](https://latex.codecogs.com/png.latex?%5Chat%7BC%7D_%7Bl%2Ct%7D%20%3D%20%5Csum_%7Bt-t%27%20%3D%200%7D%5E%7B%5Ctau_%5Cpi%7D%7B%5Chat%7BI%7D_%7Bl%2Ct%27%7D%20%5C%2C%20%5Cgamma_%7Bl%2Ct%27%7D%20%5C%2C%20%5Cpi%28t-t%27%29%7D "\hat{C}_{l,t} = \sum_{t-t' = 0}^{\tau_\pi}{\hat{I}_{l,t'} \, \gamma_{l,t'} \, \pi(t-t')}")

The distribution
![\pi(\Delta_t)](https://latex.codecogs.com/png.latex?%5Cpi%28%5CDelta_t%29 "\pi(\Delta_t)")
gives the discrete and finite (support on
![\Delta_t \in (0, \tau\_\pi)](https://latex.codecogs.com/png.latex?%5CDelta_t%20%5Cin%20%280%2C%20%5Ctau_%5Cpi%29 "\Delta_t \in (0, \tau_\pi)"))
probability distribution over delays from infection to reporting,
indexed on the time-periods considered in the model. This temporal
reweighting to account for a distribution over possible delays can be
considered as a ‘discrete-time convolution’ with
![\pi(\Delta_t)](https://latex.codecogs.com/png.latex?%5Cpi%28%5CDelta_t%29 "\pi(\Delta_t)")
the ‘kernel’. Below we discuss efficient methods for computing these
convolutions in greta.

### Infection prevalence model

We model the observed number of individuals who test positive for
infection
![N^+\_{l,t}](https://latex.codecogs.com/png.latex?N%5E%2B_%7Bl%2Ct%7D "N^+_{l,t}")
in an infection prevalence survey at location
![l](https://latex.codecogs.com/png.latex?l "l") at time
![t](https://latex.codecogs.com/png.latex?t "t") as a binomial sample,
given the number of individuals tested
![N\_{l,t}](https://latex.codecogs.com/png.latex?N_%7Bl%2Ct%7D "N_{l,t}"),
and the modelled prevalence of infections
![\hat{p}\_{l,t}](https://latex.codecogs.com/png.latex?%5Chat%7Bp%7D_%7Bl%2Ct%7D "\hat{p}_{l,t}")
across the population at that time/place:

![N^+\_{l,t} \sim \text{Binomial}(N\_{l,t}, \hat{p}\_{l,t})](https://latex.codecogs.com/png.latex?N%5E%2B_%7Bl%2Ct%7D%20%5Csim%20%5Ctext%7BBinomial%7D%28N_%7Bl%2Ct%7D%2C%20%5Chat%7Bp%7D_%7Bl%2Ct%7D%29 "N^+_{l,t} \sim \text{Binomial}(N_{l,t}, \hat{p}_{l,t})")

Similarly to clinical incidence, we model the infection prevalence at a
given location and time as a discrete-time convolution over previous
infection counts, divided by the total population in the pixel
![M_l](https://latex.codecogs.com/png.latex?M_l "M_l"), and the duration
of the time-period in days
![d](https://latex.codecogs.com/png.latex?d "d"):

![\hat{p}\_{l,t} = \frac{1}{M_l}\sum\_{t-t' = 0}^{\tau_q}{q(t-t') \\ \hat{I}\_{l,t'} \\ d^{-1}}](https://latex.codecogs.com/png.latex?%5Chat%7Bp%7D_%7Bl%2Ct%7D%20%3D%20%5Cfrac%7B1%7D%7BM_l%7D%5Csum_%7Bt-t%27%20%3D%200%7D%5E%7B%5Ctau_q%7D%7Bq%28t-t%27%29%20%5C%2C%20%5Chat%7BI%7D_%7Bl%2Ct%27%7D%20%5C%2C%20d%5E%7B-1%7D%7D "\hat{p}_{l,t} = \frac{1}{M_l}\sum_{t-t' = 0}^{\tau_q}{q(t-t') \, \hat{I}_{l,t'} \, d^{-1}}")

In this case, the kernel
![q(t-t')](https://latex.codecogs.com/png.latex?q%28t-t%27%29 "q(t-t')")
is not a probability distribution, but the kernel of a convolution
operation that maps the daily average number of new infections in
previous time periods
![t'](https://latex.codecogs.com/png.latex?t%27 "t'") to the number of
individuals who would test positive (to the diagnostic used) on a
randomly-selected day in the timeperiod `t`. This kernel, which is
specific to the tim eperiod chosen for modelling the temporal process,
can be computed from a related function:
![q\_{\text{daily}}(\Delta_t)](https://latex.codecogs.com/png.latex?q_%7B%5Ctext%7Bdaily%7D%7D%28%5CDelta_t%29 "q_{\text{daily}}(\Delta_t)")
which gives the probability that an individual will test positive
![\Delta_t](https://latex.codecogs.com/png.latex?%5CDelta_t "\Delta_t")
days after infection. The integral of this function is the average
number of days an infected person would test positive for. The function
![q\_{\text{daily}}()](https://latex.codecogs.com/png.latex?q_%7B%5Ctext%7Bdaily%7D%7D%28%29 "q_{\text{daily}}()")
can be estimated from empirical data on how the test sensitivity and
duration of infection vary over time since infection. This package
provides functionality: `transform_convolution_kernel()` to construct
timeperiod-specific kernels such as
![q()](https://latex.codecogs.com/png.latex?q%28%29 "q()") from daily
kernels like
![q\_{\text{daily}}()](https://latex.codecogs.com/png.latex?q_%7B%5Ctext%7Bdaily%7D%7D%28%29 "q_{\text{daily}}()")
for an arbitrary modelling timeperiod.

By convolving of the number of new infections *per day*
![\hat{I}\_{l,t'} \\ d^{-1}](https://latex.codecogs.com/png.latex?%5Chat%7BI%7D_%7Bl%2Ct%27%7D%20%5C%2C%20d%5E%7B-1%7D "\hat{I}_{l,t'} \, d^{-1}")
in all previous time periods with the fraction of those that would test
positive in a survey in time period
![t](https://latex.codecogs.com/png.latex?t "t"), gives an estimate of
the *number* of positive-testing people in the population at location
![l](https://latex.codecogs.com/png.latex?l "l"), on any given day
within the time period ![t](https://latex.codecogs.com/png.latex?t "t").
Dividing this by the population of that location,
![M_l](https://latex.codecogs.com/png.latex?M_l "M_l"), therefore yields
an estimate of the population proportion testing positive on any given
day; the parameter of the binomial distribution that captures the
prevalence survey data.

Note that our definition of
![\hat{p}\_{l,t}](https://latex.codecogs.com/png.latex?%5Chat%7Bp%7D_%7Bl%2Ct%7D "\hat{p}_{l,t}")
is the population prevalence of infections *that would test positive
using that diagnostic method* rather than the true fraction
infected/infectious at any one time. We also assume here that tests have
perfect specificity, though the model can easily be adapted to
situations where that is not the case.

### Infection incidence model

The expected number of new infections
![\hat{I}\_{l,t'}](https://latex.codecogs.com/png.latex?%5Chat%7BI%7D_%7Bl%2Ct%27%7D "\hat{I}_{l,t'}")
in location ![l](https://latex.codecogs.com/png.latex?l "l") during time
period ![t](https://latex.codecogs.com/png.latex?t "t") is modelled as
the product of the population
![M_l](https://latex.codecogs.com/png.latex?M_l "M_l") at that location,
the length of the timeperiod in days
![d](https://latex.codecogs.com/png.latex?d "d"), and the daily
infection incidence
![f\_{l,t}](https://latex.codecogs.com/png.latex?f_%7Bl%2Ct%7D "f_{l,t}")
at that location and time:

![\hat{I}\_{l,t'} = d \\ M_l \\ \hat{f}\_{l,t}](https://latex.codecogs.com/png.latex?%5Chat%7BI%7D_%7Bl%2Ct%27%7D%20%3D%20d%20%5C%2C%20M_l%20%5C%2C%20%5Chat%7Bf%7D_%7Bl%2Ct%7D "\hat{I}_{l,t'} = d \, M_l \, \hat{f}_{l,t}")

Whilst we mechanistically model the observation processes yielding our
data types, we employ a geostatistical approach to modelling
spatio-temporal variation in infection incidence, with spatio-temporal
covariates
![\mathbf{X}\_{l,t}](https://latex.codecogs.com/png.latex?%5Cmathbf%7BX%7D_%7Bl%2Ct%7D "\mathbf{X}_{l,t}")
and a space-time random effect
![\epsilon\_{l,t}](https://latex.codecogs.com/png.latex?%5Cepsilon_%7Bl%2Ct%7D "\epsilon_{l,t}")
with zero-mean Gaussian process prior:

![\text{log}(f\_{l,t}) = \alpha +\mathbf{X}\_{l,t} \beta + \epsilon\_{l,t}](https://latex.codecogs.com/png.latex?%5Ctext%7Blog%7D%28f_%7Bl%2Ct%7D%29%20%3D%20%5Calpha%20%2B%5Cmathbf%7BX%7D_%7Bl%2Ct%7D%20%5Cbeta%20%2B%20%5Cepsilon_%7Bl%2Ct%7D "\text{log}(f_{l,t}) = \alpha +\mathbf{X}_{l,t} \beta + \epsilon_{l,t}")

![\epsilon \sim GP(0, \mathbf{K})](https://latex.codecogs.com/png.latex?%5Cepsilon%20%5Csim%20GP%280%2C%20%5Cmathbf%7BK%7D%29 "\epsilon \sim GP(0, \mathbf{K})")

where ![\alpha](https://latex.codecogs.com/png.latex?%5Calpha "\alpha")
is a scalar intercept term,
![\beta](https://latex.codecogs.com/png.latex?%5Cbeta "\beta") is a
vector of regression coefficients against the environmental covariates,
and
![\mathbf{K}](https://latex.codecogs.com/png.latex?%5Cmathbf%7BK%7D "\mathbf{K}")
is the space-time covariance function of the Gaussian process over
random effects
![\epsilon\_{l,t}](https://latex.codecogs.com/png.latex?%5Cepsilon_%7Bl%2Ct%7D "\epsilon_{l,t}").

There are many choices of space-time covariance structure for
![\mathbf{K}](https://latex.codecogs.com/png.latex?%5Cmathbf%7BK%7D "\mathbf{K}"),
though we use a separable combination of an isotropic spatial covariance
function with a temporal covariance function, to enable the use of a
range of computationally efficient simulation and calculation methods:

![K\_{l,t,l',t'} = \sigma^2 \\ K\_{\text{space}}(\|\|l-l'\|\|; \phi) \\ K\_{\text{time}}(\|t-t'\|; \theta)](https://latex.codecogs.com/png.latex?K_%7Bl%2Ct%2Cl%27%2Ct%27%7D%20%3D%20%5Csigma%5E2%20%5C%2C%20K_%7B%5Ctext%7Bspace%7D%7D%28%7C%7Cl-l%27%7C%7C%3B%20%5Cphi%29%20%5C%2C%20K_%7B%5Ctext%7Btime%7D%7D%28%7Ct-t%27%7C%3B%20%5Ctheta%29 "K_{l,t,l',t'} = \sigma^2 \, K_{\text{space}}(||l-l'||; \phi) \, K_{\text{time}}(|t-t'|; \theta)")

where
![\sigma^2](https://latex.codecogs.com/png.latex?%5Csigma%5E2 "\sigma^2")
is the marginal variance (amplitude) of the resultant Gaussian process,
![K\_{\text{space}}(.; \phi)](https://latex.codecogs.com/png.latex?K_%7B%5Ctext%7Bspace%7D%7D%28.%3B%20%5Cphi%29 "K_{\text{space}}(.; \phi)")
is an (unit variance) isotropic spatial kernel on euclidean distances
![\|\|l-l'\|\|](https://latex.codecogs.com/png.latex?%7C%7Cl-l%27%7C%7C "||l-l'||")
with parameter
![\phi \> 0](https://latex.codecogs.com/png.latex?%5Cphi%20%3E%200 "\phi > 0")
controlling the range of spatial correlation, and
![K\_{\text{time}}(\|t-t'\|; \theta)](https://latex.codecogs.com/png.latex?K_%7B%5Ctext%7Btime%7D%7D%28%7Ct-t%27%7C%3B%20%5Ctheta%29 "K_{\text{time}}(|t-t'|; \theta)")
is a a temporal kernel on time differences
![\|t-t'\|](https://latex.codecogs.com/png.latex?%7Ct-t%27%7C "|t-t'|"),
with parameter
![\theta](https://latex.codecogs.com/png.latex?%5Ctheta "\theta")
controlling the range of temporal correlation. Again for computational
reasons, we prefer Markovian kernels for
![K\_{\text{time}}](https://latex.codecogs.com/png.latex?K_%7B%5Ctext%7Btime%7D%7D "K_{\text{time}}").

### Infection prevalence - clinical incidence relationship

As described above, the spatially- and temporally-varying parameter
![\gamma\_{l,t}](https://latex.codecogs.com/png.latex?%5Cgamma_%7Bl%2Ct%7D "\gamma_{l,t}")
gives the fraction of new infections that would result in a recorded
clinical case. In the absence of additional data to that described
above, it is unlikely that spatio-temporal variation in this parameter
can be inferred. For some diseases - e.g. those where previous
infections confer little immune protection from symptoms of future
infections - it may be sufficient to model this as a constant, in which
case it can be inferred statistically. However for many diseases, there
is likely to be a non-linear relationship between the infection
incidence in a given population, and the fraction of those infections
that result in a clinical case. Where repeat infections decrease the
clinical severity of subsequent infections,
![\gamma\_{l,t}](https://latex.codecogs.com/png.latex?%5Cgamma_%7Bl%2Ct%7D "\gamma_{l,t}")
is likely to manifest as a monotone decreasing function of infection
incidence
![f\_{l,t}](https://latex.codecogs.com/png.latex?f_%7Bl%2Ct%7D "f_{l,t}").
In this scenario, it would be preferable to account for this
relationship using external information. In the case of malaria, this
relationship is often explored using empirical data in terms of the
relationship between infection prevalence and clinical incidence. Where
a previously-estimated relationship between infection prevalence and
clinical incidence is available, the relationship between infection
incidence and clinical incidence can be inferred as follows.

We assume that a pre-determined function
![g()](https://latex.codecogs.com/png.latex?g%28%29 "g()") is available
that maps from the ‘true’ population infection prevalence
![p\_{i}](https://latex.codecogs.com/png.latex?p_%7Bi%7D "p_{i}") to the
‘true’ underlying clinical incidence
![c\_{i}](https://latex.codecogs.com/png.latex?c_%7Bi%7D "c_{i}") in
some population and setting
![i](https://latex.codecogs.com/png.latex?i "i"), for which the
corresponding infection incidence is held constant:

![c\_{i} = g(p\_{i})](https://latex.codecogs.com/png.latex?c_%7Bi%7D%20%3D%20g%28p_%7Bi%7D%29 "c_{i} = g(p_{i})")

In this case,
![c\_{i}](https://latex.codecogs.com/png.latex?c_%7Bi%7D "c_{i}") gives
the incidence of all clinical episodes regardless whether those cases
would be recorded in a health system. We therefore assume that
![g()](https://latex.codecogs.com/png.latex?g%28%29 "g()") is estimated
against empirical estimates of
![p\_{i}](https://latex.codecogs.com/png.latex?p_%7Bi%7D "p_{i}") and of
![c\_{i}](https://latex.codecogs.com/png.latex?c_%7Bi%7D "c_{i}") that
would be observed under some intensive surveillance (e.g. a prospective
study), or other high-quality source of information. We therefore
capture imperfect reporting in the health system under which observed
case counts
![C\_{i}](https://latex.codecogs.com/png.latex?C_%7Bi%7D "C_{i}") are
collected via some constant reporting rate
![r](https://latex.codecogs.com/png.latex?r "r"):

![C\_{i} = r \\ d \\ M_i \\ c\_{i}](https://latex.codecogs.com/png.latex?C_%7Bi%7D%20%3D%20r%20%5C%2C%20d%20%5C%2C%20M_i%20%5C%2C%20c_%7Bi%7D "C_{i} = r \, d \, M_i \, c_{i}")

where ![d](https://latex.codecogs.com/png.latex?d "d") is the duration
of time periods over which the counts are made, and
![M_i](https://latex.codecogs.com/png.latex?M_i "M_i") is the size of
this population (converting from clinical incidence to the number of
clinical cases, as above). Under the assumption (from
![g()](https://latex.codecogs.com/png.latex?g%28%29 "g()")) that
infection incidence
![f\_{i}](https://latex.codecogs.com/png.latex?f_%7Bi%7D "f_{i}") is
constant over the period to which
![c\_{i}](https://latex.codecogs.com/png.latex?c_%7Bi%7D "c_{i}") and
![p\_{i}](https://latex.codecogs.com/png.latex?p_%7Bi%7D "p_{i}")
correspond, these two quantities can therefore be related as:

![c\_{i} = \kappa_i f\_{i}](https://latex.codecogs.com/png.latex?c_%7Bi%7D%20%3D%20%5Ckappa_i%20f_%7Bi%7D "c_{i} = \kappa_i f_{i}")

where
![\kappa_i](https://latex.codecogs.com/png.latex?%5Ckappa_i "\kappa_i")
is the fraction of infections at location
![i](https://latex.codecogs.com/png.latex?i "i") resulting in a clinical
case (recorded or otherwise), and:

![p\_{i} = q^\* \\ f\_{i}](https://latex.codecogs.com/png.latex?p_%7Bi%7D%20%3D%20q%5E%2A%20%5C%2C%20f_%7Bi%7D "p_{i} = q^* \, f_{i}")

where ![q^\*](https://latex.codecogs.com/png.latex?q%5E%2A "q^*") is the
average number of subsequent days on which each new infection would test
positive, if tested once per day (following from the convolution above,
in the case that
![f\_{l,t}](https://latex.codecogs.com/png.latex?f_%7Bl%2Ct%7D "f_{l,t}")
and therefore
![I\_{l,t}](https://latex.codecogs.com/png.latex?I_%7Bl%2Ct%7D "I_{l,t}")
are constant over time); the integral of the detectability function
![q()](https://latex.codecogs.com/png.latex?q%28%29 "q()"):

![q^\* = \int\_{0}^{\tau_q}{q(t) dt}](https://latex.codecogs.com/png.latex?q%5E%2A%20%3D%20%5Cint_%7B0%7D%5E%7B%5Ctau_q%7D%7Bq%28t%29%20dt%7D "q^* = \int_{0}^{\tau_q}{q(t) dt}")

This scalar value
![q^\*](https://latex.codecogs.com/png.latex?q%5E%2A "q^*") can equally
be interpretted as the average number of previous days worth of
infections that are detected on the day of a prevalence survey. In
practice, ![q^\*](https://latex.codecogs.com/png.latex?q%5E%2A "q^*")
can be estimated by a sufficiently-fine resolution discrete sum over
![q(t)](https://latex.codecogs.com/png.latex?q%28t%29 "q(t)").

Combining the above, we have that:

![c_i = \kappa_i \\ f_i = g(q^\* f_i)](https://latex.codecogs.com/png.latex?c_i%20%3D%20%5Ckappa_i%20%5C%2C%20f_i%20%3D%20g%28q%5E%2A%20f_i%29 "c_i = \kappa_i \, f_i = g(q^* f_i)")

and therefore:

![\kappa_i = g(q^\* f_i) / f_i](https://latex.codecogs.com/png.latex?%5Ckappa_i%20%3D%20g%28q%5E%2A%20f_i%29%20%2F%20f_i "\kappa_i = g(q^* f_i) / f_i")

simce
![\gamma_i = r \\ \kappa_i](https://latex.codecogs.com/png.latex?%5Cgamma_i%20%3D%20r%20%5C%2C%20%5Ckappa_i "\gamma_i = r \, \kappa_i"),
and applying this relationship to our spatio-temporally varying
infection incidences
![f\_{l,t}](https://latex.codecogs.com/png.latex?f_%7Bl%2Ct%7D "f_{l,t}"),
we can compute
![\gamma\_{l,t}](https://latex.codecogs.com/png.latex?%5Cgamma_%7Bl%2Ct%7D "\gamma_{l,t}")
as:

![\gamma\_{l,t} = r \\ g(q^\* f\_{l,t}) \\ / \\ f\_{l,t}](https://latex.codecogs.com/png.latex?%5Cgamma_%7Bl%2Ct%7D%20%3D%20r%20%5C%2C%20g%28q%5E%2A%20f_%7Bl%2Ct%7D%29%20%5C%2C%20%2F%20%5C%2C%20f_%7Bl%2Ct%7D "\gamma_{l,t} = r \, g(q^* f_{l,t}) \, / \, f_{l,t}")

where ![q^\*](https://latex.codecogs.com/png.latex?q%5E%2A "q^*") is a
scalar constant that can be computed *a priori* from
![q](https://latex.codecogs.com/png.latex?q "q"),
![g()](https://latex.codecogs.com/png.latex?g%28%29 "g()") is a
deterministic function, also known *a priori*,
![f\_{l,t}](https://latex.codecogs.com/png.latex?f_%7Bl%2Ct%7D "f_{l,t}")
is our inference target, the daily infection incidence, and
![r](https://latex.codecogs.com/png.latex?r "r") is the only remaining
free parameter in this relationship, which is identifiable in the model.

In practice, the functional form of
![g()](https://latex.codecogs.com/png.latex?g%28%29 "g()") may well
permit the computational complexity of this term to be reduced, either
by computing a simpler analytic solution, or approximating it with a
functionally simpler form with near-equivalent shape. This will likely
aid in reducing the computational cost of fitting the model.

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
used to compute
![\hat{C}\_{l,t}](https://latex.codecogs.com/png.latex?%5Chat%7BC%7D_%7Bl%2Ct%7D "\hat{C}_{l,t}")
and
![\hat{p}\_{l,t}](https://latex.codecogs.com/png.latex?%5Chat%7Bp%7D_%7Bl%2Ct%7D "\hat{p}_{l,t}")
can be computationally intensive, depending on the number of time
periods modelled. A number of efficient computational approaches exist
to overcome these, and the optimal approach to use in greta depends on
the size of
![\tau^{max}](https://latex.codecogs.com/png.latex?%5Ctau%5E%7Bmax%7D "\tau^{max}"):
the maximum duration of the delay in terms of the number of timeperiods
being considered. Where
![\tau^{max}](https://latex.codecogs.com/png.latex?%5Ctau%5E%7Bmax%7D "\tau^{max}")
is relatively large (say,
![\tau^{max} \> 3](https://latex.codecogs.com/png.latex?%5Ctau%5E%7Bmax%7D%20%3E%203 "\tau^{max} > 3")),
implementation of the convolution as a matrix multiply is likely to be
the most efficient in greta. If
![\tau^{max} \> 3](https://latex.codecogs.com/png.latex?%5Ctau%5E%7Bmax%7D%20%3E%203 "\tau^{max} > 3")
and the number of time periods being modelled is much larger than
![\tau^{max}](https://latex.codecogs.com/png.latex?%5Ctau%5E%7Bmax%7D "\tau^{max}"),
implementing this as a sparse matrix multiply (skipping computation on
zero elements of the convolution matrix) is likely to be most efficient.
Where
![\tau^{max}](https://latex.codecogs.com/png.latex?%5Ctau%5E%7Bmax%7D "\tau^{max}")
is small (say,
![\tau^{max} \leq 3](https://latex.codecogs.com/png.latex?%5Ctau%5E%7Bmax%7D%20%5Cleq%203 "\tau^{max} \leq 3"))
the convolution can instead be computed with a sum of dense vectorised
additions and subtractions. These convolution approaches are implemented
in this package, and demonstrated below.

### Gaussian process simulation

Naive implementation of the full Gaussian process (GP) model is
typically very computationally intensive, due to the fact that the most
expensive step (inverting a dense covariance matrix) scales cubically
with the number of evaluation points. In the model we propose (and as in
a Log-Gaussian Cox process), we need to evaluate the GP at every pixel
location and time period in the study frame, so that we can aggregate
the expected clinical case count to compute the likelihood. This results
in a computationally impractical algorithm for all but the smallest
study frames. Some form of computational approximation is required.

One straightforward approximation would be to employ a full
(non-approximate) GP, but to evaluate the combined process (GP and fixed
effects regression) at only a subset of locations and times in the study
frame and then to approximate the health-facility level clinical case
count calculation (a set of integrals over space) with some smaller
finite sum. Whilst general, this would also result in an approximation
to the fixed-effects environmental regression part of the model. While
the GP could feasibly be smooth and well approximated in this way, the
environmental covariates are likely to be more complex and thus
potentially quite poorly approximated by a finite sum.

An alternative approach would be to evaluate the GP and the fixed
effects at all pixels, but to replace the full GP with an approximation.
Candidate GP approximation approaches include the SPDE approximation to
a Matern-type spatial kernel, on a computational mesh (as used in INLA,
[Lindgren *et al.*,
2011](https://doi.org/10.1111/j.1467-9868.2011.00777.x)), a sparse GP
method over a limited set of inducing points ([Quinonera-Candela &
Rasmussen, 2005](https://jmlr.org/papers/v6/quinonero-candela05a.html) -
see `greta.gp` and note the similarity to a Guassian predictive process
per [Banerjee *et al.*,
2009](https://doi.org/10.1111/j.1467-9868.2008.00663.x)), or one of the
closely-related penalised spline methods (see `greta.gam`).
Implementation of the SPDE approximation in an HMC setting requires
implementation of sparse matrix methods adapted to automatic
differentiation ([Durrande *et al.*
2019](https://doi.org/10.48550/arXiv.1902.10078), greta version in
development), but would likely be preferable where the underlying
Gaussian process has short lengthscale. In the case of a comparatively
smooth underlying GP, the sparse o spline-based methods are likely to be
a better fit, since they have lower computational complexity in this
case. Given the spatially-aggregated observation model, it seems likely
that only comparatively smooth GP realisations could be identified in
this model, and therefore favour sparse or spline methods.

An appealing third alternative in this case is to compute the full
spatial GP for every pixel in a regular grid (the same we use to record
spatial covariate values), by exploiting the block-circulant structure
in the resulting spatial covariance matrix. This requires using
projected coordinates, and expanding the spatial study frame (by a
factor of 2 in each dimension) to map the projection onto a torus in
such a way that the distances between pairs of pixels are preserved.
This approach enables simulation of the GP across all pixels, for a
given time period, via the fast fourier transform (FFT) - which scales
only linearly with the number of locations considered (? check when have
wifi). This enables us to simulate values of the spatial GP across all
pixels very cheaply, with no need to approximate the GP. In separable
combination with a Markovian temporal kernel, this enables very rapid
inference. However the very large number of random variables required to
be sampled (at least four times as many as pixels and time points under
study) leads to a very large parameter space for sampling, and can slow
down efficient MCMC sampling considerably.

## Example application

We demonstrate the model with application to mapping the (simulated)
infection incidence of malaria in Kenya from simulated clinical
incidence and infection prevalence data. Using the `sim_data()` function
from this package, we use a real grid of environmental covariates, but
simulate the locations of health facilities, prevalence survey
locations, and all datasets to which we will then fit the model.

### Simulating data

First we load the bioclim covariates using the `terra` and `geodata` R
package:

``` r
library(terra)
library(geodata)
# download at a five-degree resolution. See ?geodata_path for how to save the data between sessions
bioclim_kenya <- geodata::worldclim_country(
  country = "KEN",
  var = "bio",
  res = 0.5,
  path = file.path(tempdir(), "bioclim")
)

pop <- geodata::population(
  year = 2020,
  res = 0.5,
  path = file.path(tempdir(), "population")
)

# crop and mask to Kenya
pop_kenya <- mask(pop, bioclim_kenya)
```

    #> Warning: package 'terra' was built under R version 4.2.3

The Gaussian process implementations we use below assume that the
coordinates are on a plane - they don’t account for the curvature of the
earth (unlike INLA which is able to model GPs on a sphere). We therefore
first need to ensure that the raster we use to define this computational
setup is appropriately projected. For the block-circulant matrix
approach, this is especially important since the computation relies on
the compute locations being arranged in a regular gird. We therefore
project our matrix to an appropriate projected coordinate reference
system:

``` r
# define a template raster representing our compute grid: 0 in land cells, NA
# elsewhere
grid_raster <- pop_kenya * 0
names(grid_raster) <- NULL

# Pick projection as UTM zone 36S (Uganda, Kenya, Tanzania)
new_crs <- "epsg:21036"

# project this template raster, the bioclim layers, and the population raster
grid_raster <- terra::project(grid_raster, new_crs)
bioclim_kenya <- terra::project(bioclim_kenya, new_crs)
pop_kenya <- terra::project(pop_kenya, new_crs)
```

Next we prepare our covariates. We will lower the spatial resolution a
little to make the example run faster. This is a good idea when
initially setting up and debugging a model, but you’ll likely want to
increase the spatial resolution later. Then we simulate some fake data,
using the first 5 covariates:

``` r
# aggregate rasters
grid_raster <- terra::aggregate(grid_raster, 20)
bioclim_raster <- terra::aggregate(bioclim_kenya, 20)
pop_raster <- terra::aggregate(pop_kenya, 20)

# subset and scale the bioclim layers to make our covariates
covariates_raster <- scale(bioclim_raster[[1:5]])

# set time periods
start_year <- 2020
n_years <- 3

# simulate fake data
library(epiwave.mapping)
set.seed(1)
data <- sim_data(
  covariates_rast = covariates_raster,
  population_rast = pop_raster, 
  years = seq(start_year, start_year + n_years - 1),
  n_health_facilities = 30,
  n_prev_surveys = 15,
  # assume some fraction of case counts are missing
  case_missingness = 0.3)
```

Let’s visualise the fake data.

Plot the fake health facility locations over the population raster:

``` r
library(tidyverse)
#> Warning: package 'ggplot2' was built under R version 4.2.3
#> Warning: package 'tidyr' was built under R version 4.2.3
#> Warning: package 'dplyr' was built under R version 4.2.3
library(tidyterra)

# plot the health facility locations
ggplot() +
  geom_spatraster(data = pop_raster) +
  scale_fill_gradient(
    low = grey(0.9),
    high = grey(0.6),
    transform = "log1p",
    na.value = "transparent",
    guide = "none") +
  geom_point(
    aes(
      y = y,
      x = x
    ),
    data = data$surveillance_information$health_facilities$health_facility_loc,
    shape = 16
  ) +
  xlab("") +
  ylab("") +
  theme_minimal() +
  ggtitle("Health facility locations")
```

![](README_files/figure-gfm/vis_data_1-1.png)<!-- -->

Plot the clinical case timeseries for all health facilities over each
year:

``` r
# add months and years onto the clinical cases, for plotting
cases <- data$epi_data$clinical_cases %>%
  mutate(
    month = 1 + (time - 1) %% 12,
    year = start_year + (time - 1) %/% 12,
    health_facility = factor(health_facility)
  )

ggplot(
  aes(
    x = month,
    y = cases,
    colour = health_facility,
    group = health_facility
  ),
  data = cases) +
  facet_wrap(~year) +
  geom_line(
    linewidth = 0.3
  ) + 
  theme_minimal() +
  scale_colour_discrete(
    guide = "none"
  ) +
  scale_x_continuous(breaks = 1:12) +
  ggtitle("Clinical case timeseries")
#> Warning: Removed 21 rows containing missing values or values outside the scale range
#> (`geom_line()`).
```

![](README_files/figure-gfm/vis_data_2-1.png)<!-- -->

Plot the prevalence survey results at the prevalence survey locations:

``` r
prev_surveys <- data$epi_data$prevalence_surveys %>%
  mutate(
    prevalence = n_positive / n_sampled
  )

ggplot() +
  geom_spatraster(data = pop_raster) +
  scale_fill_gradient(
    low = grey(0.9),
    high = grey(0.9),
    na.value = "transparent",
    guide = "none") +
  geom_point(
    aes(
      y = y,
      x = x,
      colour = prevalence
    ),
    data = prev_surveys,
    size = 3,
    shape = 16
  ) +
  scale_colour_gradient(
    labels = scales::label_percent(),
    low = "skyblue",
    high = "darkred"
  ) +
  xlab("") +
  ylab("") +
  theme_minimal() +
  ggtitle("Prevalence surveys")
```

![](README_files/figure-gfm/vis_data_3-1.png)<!-- -->

Plot the fake delay distribution (probability mass function over days)
and prevalence survey detectability functions:

``` r
case_delay_plot <- tibble(
  days = seq(0, data$surveillance_information$case_delay_max_days)) %>%
  mutate(
    pmf = data$surveillance_information$case_delay_distribution_daily_fun
(days)
  )

ggplot(
  aes(x = days,
      y = pmf),
  data = case_delay_plot) +
  geom_step() +
  theme_minimal() +
  ggtitle("Case reporting delay distribution",
          "Probability of reporting X days after infection")
```

![](README_files/figure-gfm/vis_data_4-1.png)<!-- -->

``` r
detectability_plot <- tibble(
  days = seq(0, data$surveillance_information$prev_detectability_max_days)) %>%
  mutate(
    detectability = data$surveillance_information$prev_detectability_daily_fun
(days)
  )

ggplot(
  aes(x = days,
      y = detectability),
  data = detectability_plot) +
  geom_step() +
  theme_minimal() +
  ggtitle("Prevalence survey detectability function",
          "Probability of registering a positive if tested X days after infection")
```

![](README_files/figure-gfm/vis_data_5-1.png)<!-- -->

Plot the assumed infection prevalence - clinical case incidence
relationship:

``` r
prev_inc_plot <- tibble(prevalence = seq(0, 0.5, length.out = 100)) %>%
  mutate(
    clinical_incidence = data$surveillance_information$prev_inc_function
(prevalence)
  )

ggplot(
  aes(x = prevalence,
      y = 365 * clinical_incidence),
  data = prev_inc_plot) +
  geom_line() +
  theme_minimal() +
  xlab("Prevalence") +
  ylab("Clinical incidence (annual)") +
  ggtitle("Prevalence - clinical incidence function")
```

![](README_files/figure-gfm/vis_data_6-1.png)<!-- -->

### Model specification using greta

To specify the model in greta, first we define the model hyperparameters
with priors, then we create any random effects (here the spatiotemporal
random effects `epsilon`) and other model objects that depend on them.
Then we can define the observation model - linking our generative model
to data.

First we will define the hyperparameters and spatio-temporal process
structure and for the spatio-temporal random effects epsilon. We provide
two options: the block-circulant embedding approach, and the sparse GP
approach, and we demonstrate how to set up objects for both.

We can use the `epiwave.mapping::deifne_bcb_setup()` function to define
the objects required for the block-circulant basis approach to
simulating IID Guassian processes over space, and converting them to
spatio-temporal Gaussian processes:

``` r
bcb_setup <- define_bcb_setup(grid_raster)
```

For the sparse GP approach, we instead need to construct a set of
spatial ‘inducing points’ or ‘knots’ across the region of interest. The
number of inducing points controls the accuracy of the approximation -
with one inducing point per pixel, this is no longer an approximation.
An insufficient number of inducing points will bias inference towards
smoother Gaussian processes, though trial and error is required to
ensure the inducing point scheme is sufficient to accurately capture the
underlying process. For now, we will use a small number of points. We
generate points evenly across space in a somewhat ad-hoc way using
kmeans clustering. Note that improved efficiency of the approximation
could be achieved by weighting inducing points towards more populous
areas.

``` r
n_inducing <- 30
pts_sim <- terra::spatSample(grid_raster,
                             n_inducing * 1e3,
                             replace = TRUE,
                             xy = TRUE,
                             values = FALSE,
                             na.rm = TRUE)
                  
inducing <- kmeans(pts_sim, n_inducing)$centers
```

We can plot these to see where they fall and check their coverage:

``` r
# plot the health facility locations
ggplot() +
  geom_spatraster(data = pop_raster) +
  scale_fill_gradient(
    low = grey(0.9),
    high = grey(0.6),
    transform = "log1p",
    na.value = "transparent",
    guide = "none") +
  geom_point(
    aes(
      y = y,
      x = x
    ),
    data = inducing,
    shape = 16
  ) +
  xlab("") +
  ylab("") +
  theme_minimal() +
  ggtitle("Inducing point locations")
```

![](README_files/figure-gfm/plot_inducing-1.png)<!-- -->

Now we can define a space-time Gaussian process over `epsilon`, by
creating an isotropic spatial kernel object using the greta.gp R
package, a parameter for the AR(1) temporal kernel parameter, and a
marginal variance parameter, and passing these to the `epiwave.mapping`
functions `sim_gp_bcb()` (for the block-circulant embedding) or
`sim_gp_sparse()`.

We first need to define priors for the three hyperparameters of the
spatio-temporal process. We define a penalised complexity prior
(half-normal) for the marginal variance parameter `sigma`, which shrinks
the degree of spatio-temporal correlation towards 0. Since we believe
negative residual correlation between time periods (oscillation between
negative and positive on a monthly timesteps) to be incredibly unlikely,
we define a standard uniform prior on the temporal correlation parameter
`theta` so that is positive, but otherwise not informative (the spatial
pattern could be either constant, or constantly changing). For the
spatial range parameter `phi`, we need to take more care to consider
reasonable values in terms of the units of the map (metres). We set up
an informative prior to control both the lower and the upper tail of the
`phi` - making very short and very long lengthscales (very short-range
and long-range spatial correlations) *a priori* unlikely. This helps to
resolve the various types of non-identifiability inherent in model-based
geostatistics: between the lengthscale and marginal variance parameters,
between shorter lengthscales and the spatial covariate parameters `beta`
(spatial confounding), and between longer lengthscales and the intercept
parameter `alpha`. We do so, by having the user specify upper and lower
bounds on the degree of spatial correlation that would be plausible, in
terms of the spatial extent of the country under consideration.

``` r
library(greta)
library(greta.gp)

# Variance of space-time process shrunk towards 0
sigma <- normal(0, 1, truncation = c(0, Inf))

# Temporal correlation forced to be positive, but equally as likely to be strong
# as weak
theta <- uniform(0, 1)

# get the maximum spatial extent of each dimension of the raster, in metres
grid_extent <- as.vector(ext(grid_raster))
x_range <- grid_extent["xmax"] - grid_extent["xmin"]
y_range <- grid_extent["ymax"] - grid_extent["ymin"]
max_distance <- max(x_range, y_range)

# First, find a lower bound for the lengthscale parameter phi, a reasonable
# maximum level of complexity, such that spatial correlation would be 0.1 at
# some fraction of the maximum distance, under a Matern 5/2 covariance
# structure:
phi_lower <- matern_lengthscale_from_correl(distance = max_distance / 5,
                                            correlation = 0.1,
                                            nu = 5/2)

# Do the same for an upper bound. A reasonable spatial correlation for
# some longer distance.
phi_upper <- matern_lengthscale_from_correl(distance = max_distance / 2,
                                            correlation = 0.1,
                                            nu = 5/2)

# Now define a gamma prior on the lengthscale parameter phi, such that the mass
# is low on lengthscales below the lower bound (very complex spatial effects)
# and low on lengthscales above the upper bound (completely flat spatial
# effects). Specifically, we find gamma parameters such that there is only a 10%
# chance of *more complex* spatial correlations than the one above. Note that
# whilst an inverse gamma has the inherent property that it has no mass at 0,
# that is taken care of here by our lower bound, and the gamma is easier to
# solve for these constraints.
gamma_prior_params <- find_gamma_parameters(upper_value = phi_upper,
                                            upper_prob = 0.1,
                                            lower_value = phi_lower,
                                            lower_prob = 0.1)

# # we can double check this converged OK:
# round(pgamma(phi_lower,
#              shape = gamma_prior_params$shape,
#              rate = gamma_prior_params$rate), 2)
# round(1 - pgamma(phi_upper,
#                  shape = gamma_prior_params$shape,
#                  rate = gamma_prior_params$rate), 2)

# define the prior over the lengthscale parameter phi
phi <- gamma(shape = gamma_prior_params$shape,
             rate = gamma_prior_params$rate)
```

To use the block-circulant approach we need to define a spatial kernel
in greta.gp, with only a single lengthscale parameter, and pass this to
`epiwave.mapping::sim_gp_bcb()` along with our setup for the BCB
approach. I’ve commented this out (as well as not evaluating the block),
since running this block in addition. to the sparse approach we use
below will make the model run very slowly.

``` r
# # We define an isotropic kernel in greta.gp. We fix the variance of this
# # kernel to 1, since we define that parameter at the level of the combined
# # space-time kernel
# k_space <- mat52(lengthscales = phi,
#                  variance = 1)
# 
# # Now we can pass this kernel and the other hyperparameters to the function to
# # create space-time GPs over all cells
# epsilon <- epiwave.mapping::sim_gp_bcb(
#   bcb_setup = bcb_setup,
#   n_times = n_years * 12,
#   space_kernel = k_space,
#   time_correlation = theta,
#   sigma = sigma
# )
```

Alternatively, we can construct the space-time process using a sparse GP
with `epiwave.mapping::sim_gp_sparse()`. In this case (*for reasons*),
to specify an isotropic spatial kernel we need to pass the lengthscale
parameter to the `greta.gp` kernel function twice, once for each
dimension:

``` r
# note we pass the lengthscale argument twice
k_space <- mat52(lengthscales = c(phi, phi),
                 variance = 1)

epsilon <- epiwave.mapping::sim_gp_sparse(
  inducing_points = inducing,
  grid_raster = grid_raster,
  n_times = n_years * 12,
  space_kernel = k_space,
  time_correlation = theta,
  sigma = sigma
)
```

We can visually check that our prior is giving us reasonable values of
epsilon by doing some prior simulations and then plotting them:

``` r
n_sims <- 3
times_plot <- 1:5
prior_sims <- calculate(epsilon,
                        phi,
                        theta,
                        sigma,
                        nsim = n_sims)

# for each simulation, create an empty multilayer raster, fill it in with the
# simulations for each of the first three years, and plot it
plot_list <- list() 
for (sim in seq_len(n_sims)) {
  
  plot_raster <- do.call(c,
                         replicate(length(times_plot),
                                   grid_raster,
                                   simplify = FALSE))
  names(plot_raster) <- paste("timestep", times_plot)
  
  for (i in seq_along(times_plot)) {
    plot_raster[[i]][cells(grid_raster)] <- prior_sims$epsilon[sim, , i]
  }

  plot_list[[sim]] <- ggplot() +
    geom_spatraster(data = plot_raster) +
    scale_fill_distiller(
      palette = "PiYG",
      na.value = "transparent",
      guide = "none") +
    facet_wrap(~lyr, nrow = 1) +
    theme_minimal() +
    xlab("") +
    ylab("") +
    ggtitle(label = paste("Simulation", sim),
            subtitle = sprintf(
              "phi (space) = %s, theta (time) = %s, sigma = %s",
              round(prior_sims$phi[sim, 1, 1]),
              round(prior_sims$theta[sim, 1, 1], 2),
              round(prior_sims$sigma[sim, 1, 1], 2)
            ))
}

library(patchwork)
#> Warning: package 'patchwork' was built under R version 4.2.3
combined_plot <- plot_list[[1]]
for (sim in 2:n_sims) {
  combined_plot <- combined_plot / plot_list[[sim]]
}
combined_plot
```

![](README_files/figure-gfm/gp_prior_check-1.png)<!-- --> You can run
that block multiple times to see different realisations, and check they
conform to you prior expectation about the spatio-temporal random
process.

Now we have the object for the spatio-temporal random effects, we define
our remaining hyperparameters for the covariate effects, and reporting
fraction.

Here we use an arbitrary set of priors, that happen to be those used to
simulate the ‘true’ parameters in `epiwave.mapping::sim_data()`. In a
practical application, the user should give these a lot more thought.

``` r
# intercept and slope for fixed-effects terms on daily infection incidence
alpha <- normal(log(1e-4), 1)
beta <- normal(0, 1, dim = terra::nlyr(covariates_raster))
# fraction of all clinical cases reported in case counts
r <- beta(10, 5)
```

Then we can define the deterministic mapping from these, through
infection incidence, to our observed data. First we compute the daily
incidence of new infections, per cell, per timestep.

``` r

# get an index to all pixel values
all_cells <- terra::cells(grid_raster)

# extract the covariate design matrix (scale)
X <- terra::extract(covariates_raster, all_cells)

# matrix-multiply with beta, and add alpha, to get the spatial fixed effects
fixef <- alpha + X %*% beta

# add on epsilon, and convert to infection incidence
eta <- sweep(epsilon, 1, fixef, FUN = "+")
infection_incidence <- ilogit(eta)

# multiply by population and timeperiod to get the count of new infections per
# time period
timeperiod_days <- 365/12
pop_all_cells <- terra::extract(pop_raster, all_cells)[[1]]

daily_new_infections <- sweep(infection_incidence, 1, pop_all_cells, FUN = "*")
timeperiod_new_infections <- daily_new_infections * timeperiod_days
```

Note that we can similarly simulate other objects like
`timeperiod_new_infections` from the model priors and plot them as
rasters to check the model and prior specification seems reasonable.

Next we will compute the expected prevalence at our prevalence survey
locations. We need to find the index to the rows in
`timeperiod_new_infections` where the prevalence surveys were performed,
and then apply the appropriate convolution on these to compute the
expeccted prevalences at those times and places.

``` r
# get the prevalence survey coordinates (already projected int he same CRS as the rasters)
prev_coords <- data$epi_data$prevalence_surveys %>%
  select(x, y) %>%
  as.matrix()
# find an index to the cells in the full grid
prev_cells <- terra::cellFromXY(grid_raster, prev_coords)
# get the index to rows in our compute objects
prev_index <- match(prev_cells, all_cells)

# pull out the number of infections per day in these locations
daily_new_infections_prev_locs <- daily_new_infections[prev_index, ]

# now to do the convolution, we first need to translate the detectability
# function from a daily timestep to our monthly timestep
q_daily <- data$surveillance_information$prev_detectability_daily_fun
q_max_days <- data$surveillance_information$prev_detectability_max_days
q_timeperiod <- transform_convolution_kernel(kernel_daily = q_daily,
                                  max_diff_days = q_max_days,
                                  timeperiod_days = timeperiod_days)

# now we can do the convolution to get the number of detectable infections, if
# measured each day
detectable_infections_daily_prev_locs <- convolve_matrix(
  matrix = daily_new_infections_prev_locs,
  kernel = q_timeperiod)

# compute prevalence at these locations over time by dividing by population of
# the locations where prevalence surveys were done
population_prev_locs <- pop_all_cells[prev_index]
prevalence_prev_locs <- sweep(detectable_infections_daily_prev_locs,
      1, population_prev_locs, FUN = "/")
```

Now we model the expected clinical incidence at each pixel, accounting
for the prevalence-incidence relationship, delays between infection and
reporting, imperfect reporting rates, and aggregating the counts over
health facility catchments.

``` r
# first, we define gamma: the ratio of clinical cases to infections

# We estimate q_star, the integral of the daily detectability function
q_star <- sum(q_daily(seq(0, q_max_days)))

# As per equations above, we can get the instantaneous prevalence using this,
# and the provided prevalence-incidence relationship to map to the incidence
# rate of all clinical cases (reported or not)
g <- data$surveillance_information$prev_inc_function
clinical_case_incidence <- g(q_star * infection_incidence)
# multiplying by the reporting rate r gives us the reported clinical incidence rate
reported_clinical_case_incidence <- r * clinical_case_incidence

# to convert this to counts of reported clinical cases per pixel (before
# temporal convolution), we compute and use gamma like in the above equations,
# like this:
gamma <- reported_clinical_case_incidence / infection_incidence
reported_clinical_cases <- gamma * timeperiod_new_infections

# note we could also multiply reported_clinical_case_incidence by the
# populations and the monthly timeperiod

# next, we convolve these through time to account for delays between infection
# and reporting. This follows the same process as for prevalence:

pi_daily <- data$surveillance_information$case_delay_distribution_daily_fun
pi_max_days <- data$surveillance_information$case_delay_max_days
pi_timeperiod <- transform_convolution_kernel(kernel_daily = pi_daily,
                                              max_diff_days = pi_max_days,
                                              timeperiod_days = timeperiod_days)

# now we can convolve to compute prevalence at all pixels over time
reported_clinical_cases_shifted <- convolve_matrix(reported_clinical_cases,
                                        kernel = pi_timeperiod)
```

This gives us the expected number of clinical cases reported in each
timeperiod, in each pixel. Next we need to aggregate them at health
facility level to link them to the available data.

``` r
# First, we'll need to reproject the catchment model, and then extract all
# catchment information into a matrix
catchment_rast_raw <- data$surveillance_information$health_facilities$catchments_rast
catchment_rast <- terra::project(catchment_rast_raw, new_crs) 
catchment_weights <- terra::extract(catchment_rast, all_cells)

# after reprojecting, we will need to ensure that the weights sum to 1 across
# all rows(pixels), so that the weights are probabilities of attending each
# health facility. If they don't sum to 1, we'll be losing/adding cases in an
# unpredictable way
catchment_weight_sums <- rowSums(catchment_weights)
catchment_weights <- sweep(catchment_weights, 1, catchment_weight_sums, FUN = "/")

# now we just need a matrix multiplication to assign the reported cases in each
# pixel and time to health facility level, resulting in a matrix with rows
# giving health facilities and columns giving time periods
reported_clinical_cases_hf <- t(catchment_weights) %*% reported_clinical_cases_shifted
```

Now we define likelihoods for the two types of observation data. First
we define the likelihood for prevalence survey data by extracting the
prevalence values from the modelled timeseries, at the times and
locations where the surveys were performed.

``` r
# Now we find the timeperiod in which each survey was done, and extract the
# corresponding prevalence estimates
n_surveys <- nrow(data$epi_data$prevalence_surveys)
idx <- cbind(
  seq_len(n_surveys), # rows
  data$epi_data$prevalence_surveys$time # columns
)

prevalence_estimates <- prevalence_prev_locs[idx]

# now we can define a binomial likelihood over the data for this part of the
# model
positive <- data$epi_data$prevalence_surveys$n_positive
tested <- data$epi_data$prevalence_surveys$n_sampled
distribution(positive) <- binomial(tested, prevalence_estimates)
```

Now we add the likelihood for the clinical cases reported at health
facility level, accounting for missingness of data in these timeseries.

``` r
# find the non-missing data
valid_idx <- !is.na(data$epi_data$clinical_cases$cases)

# pull out these places, times, and cases
cases_valid <- data$epi_data$clinical_cases[valid_idx, ]
cases_observed <- cases_valid$cases

# extract the corresponding estimates of cases
cases_extract_index <- cbind(cases_valid$health_facility, cases_valid$time)
cases_expected <- reported_clinical_cases_hf[cases_extract_index]

# define the poisson observation model
distribution(cases_observed) <- poisson(cases_expected)
```

### Model fitting and prediction

Now we have defined all of the components of the model, we can run MCMC
to estimate parameters.

In general, it would be a good idea to use calculate to simulate some of
these quantities (predicted prevalences etc) from the priors, to make
sure they are somewhat reasonable, and there are no numerical issues or
mistakes in the model code. I’ll skip that for now in this example,
since the model has already been checked.

``` r
# list only the quantities/parameters we want to trace the posteriors for in the
# first instance. We can always recompute other quantities later, so keep this
# to the hyperparameters.
m <- model(
  alpha, beta,
  r,
  phi, theta, sigma
)

# the model has trouble initialising when the infection incidence is high
# (because the number of people detectable in the prevalence surveys can exceed
# the population), so initialise it to be low

# # later: work out how to cap it without ruining the gradients
n_chains <- 4
inits <- replicate(n_chains,
                   initials(
                     alpha = runif(1, -10, -9)
                   ),
                   simplify = FALSE)

draws <- mcmc(m,
              chains = n_chains,
              initial_values = inits)

r_hats <- coda::gelman.diag(draws, autoburnin = FALSE, multivariate = FALSE)
```

Now we have sampled the model, we can plot posterior maps of quantities
we are interested in. Here we will the infection incidence rate (annual,
per person) and the numbers of clinical cases expected per pixel, per
timeperiod. To get posterior samples of quantities of interest for greta
arrays we pass the posterior samples to the `values` argument of tha
`greta::calculate()` function. Then we summarise these to get the
posterior means before putting them in the rasters to visualise them.

``` r

# first, we calculate the number of clinical cases (reported or otherwise) at the time of reporting, per timeperiod, per pixel
# clinical_cases_pixel <- reported_clinical_cases_shifted / r
clinical_cases_pixel <- reported_clinical_cases / r

posterior_sims <- calculate(infection_incidence,
                            clinical_cases_pixel,
                            values = draws,
                            nsim = 1000)

# in these, the first dimension is the posterior sample, the second is the pixel location, and the third is the timeperiod. Sowe calculate the mean across the first dimension
infection_incidence_post_mean <- apply(posterior_sims$infection_incidence, 2:3, mean)
clinical_cases_pixel_post_mean <- apply(posterior_sims$clinical_cases_pixel, 2:3, mean)

# now we can put these in rasters
n_times <- ncol(infection_incidence)
plot_raster <- do.call(c,
                       replicate(n_times,
                                 grid_raster,
                                 simplify = FALSE))
names(plot_raster) <- paste("timestep", seq_len(n_times))

infection_incidence_post_raster <- plot_raster
clinical_cases_pixel_post_raster <- plot_raster
for (i in seq_len(n_times)) {
  infection_incidence_post_raster[[i]][all_cells] <- infection_incidence_post_mean[ , i]
  clinical_cases_pixel_post_raster[[i]][all_cells] <- clinical_cases_pixel_post_mean[ , i]
}

# plot for 5 timeperiods
times_plot <- round(seq(1, n_times, length.out = 5))

infection_incidence_plot <- ggplot() +
    geom_spatraster(data = 365 * infection_incidence_post_raster[[times_plot]]) +
    scale_fill_distiller(
      name = "Incidence",
      palette = "YlGnBu",
      direction = 1,
      na.value = "transparent") +
    facet_wrap(~lyr, nrow = 1) +
    theme_minimal() +
    xlab("") +
    ylab("") +
    ggtitle("Infection incidence",
            "per person per year")

clinical_cases_pixel_plot <- ggplot() +
    geom_spatraster(data = clinical_cases_pixel_post_raster[[times_plot]]) +
    scale_fill_distiller(
      name = "Cases",
      palette = "YlOrRd",
      direction = 1,
      na.value = "transparent") +
    facet_wrap(~lyr, nrow = 1) +
    theme_minimal() +
    xlab("") +
    ylab("") +
    ggtitle("Clinical cases",
            "number per pixel per timeperiod")

clinical_cases_pixel_plot / infection_incidence_plot
```

![](README_files/figure-gfm/plot_posteriors-1.png)<!-- -->

### Comparison with simulated truth

We can compare these estimates with the ‘true’ simulated surfaces from
the `data` object.

``` r
infection_incidence_truth_raster <- data$truth$rasters$infection_incidence_rast

clinical_cases_truth_raster <- data$truth$rasters$clinical_cases_rast


# plot for 5 timeperiods
times_plot <- round(seq(1, n_times, length.out = 5))

infection_incidence_truth_plot <- ggplot() +
    geom_spatraster(data = 365 * infection_incidence_truth_raster[[times_plot]]) +
    scale_fill_distiller(
      name = "Incidence",
      palette = "YlGnBu",
      direction = 1,
      na.value = "transparent") +
    facet_wrap(~lyr, nrow = 1) +
    theme_minimal() +
    xlab("") +
    ylab("") +
    ggtitle("Infection incidence",
            "per person per year")

clinical_cases_truth_plot <- ggplot() +
    geom_spatraster(data = clinical_cases_truth_raster[[times_plot]]) +
    scale_fill_distiller(
      name = "Cases",
      palette = "YlOrRd",
      direction = 1,
      na.value = "transparent") +
    facet_wrap(~lyr, nrow = 1) +
    theme_minimal() +
    xlab("") +
    ylab("") +
    ggtitle("Clinical cases",
            "number per pixel per timeperiod")

clinical_cases_truth_plot / infection_incidence_truth_plot
```

![](README_files/figure-gfm/plot_truth-1.png)<!-- -->

We can also compare the posterior estimates of the hyperparameters
against their true values in the simulation

``` r

# get true values
param_names <- c("alpha", "beta", "r", "phi", "theta", "sigma")
param_truth <- unlist(data$truth$parameters[param_names])

# compute summaries of parameters in the same order
param_post <- calculate(
  alpha,
  beta,
  r,
  phi,
  theta,
  sigma,
  values = draws,
  nsim = 1e3
)

param_prior <- calculate(
  alpha,
  beta,
  r,
  phi,
  theta,
  sigma,
  nsim = 1e3
)

post_mean <- unlist(lapply(param_post, colMeans))
post_lower <- unlist(lapply(param_post, function(x) apply(x, 2:3, quantile, 0.025)))
post_upper <- unlist(lapply(param_post, function(x) apply(x, 2:3, quantile, 0.975)))

prior_mean <- unlist(lapply(param_prior, colMeans))
prior_lower <- unlist(lapply(param_prior, function(x) apply(x, 2:3, quantile, 0.025)))
prior_upper <- unlist(lapply(param_prior, function(x) apply(x, 2:3, quantile, 0.975)))

tibble(
  parameter = names(param_truth),
  truth = round(param_truth, 2),
  posterior_mean = round(post_mean, 2),
  posterior_ci = paste(
    round(post_lower, 1),
    round(post_upper, 1),
    sep = " to "
  ),
  prior_ci = paste(
    round(prior_lower, 1),
    round(prior_upper, 1),
    sep = " to "
  ),
)
#> # A tibble: 10 × 5
#>    parameter     truth posterior_mean posterior_ci      prior_ci           
#>    <chr>         <dbl>          <dbl> <chr>             <chr>              
#>  1 alpha         -9.84          -9.78 -10.2 to -9.4     -11.2 to -7.2      
#>  2 beta1          0.18          -0.02 -1.1 to 0.9       -1.9 to 1.9        
#>  3 beta2         -0.84          -0.68 -1.2 to -0.2      -1.9 to 2          
#>  4 beta3          1.6            0.86 0.1 to 2          -1.9 to 1.9        
#>  5 beta4          0.33          -0.19 -0.8 to 0.7       -2 to 1.9          
#>  6 beta5         -0.82          -0.59 -1.6 to 0.4       -2 to 2.1          
#>  7 r              0.6            0.58 0.4 to 0.8        0.4 to 0.9         
#>  8 phi       214688.        155049.   93028 to 224909.3 82197.7 to 339101.8
#>  9 theta          0.6            0.54 0.3 to 0.7        0 to 1             
#> 10 sigma          0.5            0.36 0.3 to 0.5        0 to 2.2
```

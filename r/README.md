# R code for PMH tutorial

This R code implements the Kalman filter (KF), particle filter (PF) and particle Metropolis-Hastings (PMH) algorithm for two different dynamical models: a linear Gaussian state-space (LGSS) model and a stochastic volatilty (SV) model. Note that the Kalman filter can only be employed for the first of these two models. The details of the code is described in the tutorial paper available at: < http://arxiv.org/pdf/1511.01707 >.

Requirements
--------------
The code is written and tested for R 3.2.2 and makes use of the packages Quandl and mvtnorm. These can be installed in R by executing the command:
``` R
install.packages(c("mvtnorm", "Quandl")) 
```

Main script files
--------------
These are the main script files that implement the various algorithms discussed in the tutorial.

**example1-lgss.R** State estimation in a LGSS model using the KM and a fully-adapted PF (faPF). The code is discussed in Section 3.1 and the results are presented in Section 3.2 as Figure 4 and Table 1.

**example2-lgss.R** Parameter estimation of one parameter in the LGSS model using PMH with the faPF as the likelihood estimator. The code is discussed in Section 4.1 and the results are presented in Section 4.2 as Figure 5.

**example3-sv.R** Parameter estimation of three parameters in the SV model using PMH with the bootstrap PF as the likelihood estimator. The code is discussed in Section 5.1 and the results are presented in Section 5.2 as Figure 6. The code takes about an hour to run.

**example4-sv.R** Modified version of the code in *example3-sv.R* to make use of a better tailored parameter proposal. The details are discussed in Section 6.3.2 and the results are presented in the same section as Figures 7 and 8. Note that the only difference in the code is that the variable stepSize is changed.

**example5-sv.R** Modified version of the code in *example3-sv.R* to make use of another parameterisation of the model and a better tailored parameter proposal. The details are discussed in Section 6.3.3 and the results are presented in the same section. Note that the differences in the code is the use of another implemenation of PMH ant that the variable stepSize is changed.


Additional script files for creating plots for tutorial (codeForPaper/)
--------------
These are some additional files to recreate some extra results discussed in the tutorial.

**example1-lgss-plotData.R** Some sample code for generate data and recreate the plot of the data presented as Figure 3.

**example2-lgss-varyingT.R** An extended version of *example2-lgss.R* and makes several runs while changing the number of observations. The results are presented in Section 3.2 as Table 1.

**example4-sv-plotProposals.R** Some (ugly) code to plot the estimates of the posterior distribution and the proposal distribution using the output from a run of *example3-sv.R*. This code generates Figure 7 in Section 6.3.2.


Supporting files (helpers/)
--------------
**stateEstimation.R**
Implements data generation for the LGSS model (generateData), the faPF for the LGSS model (particleFilter), the Kalman filter for the LGSS model (kalmanFilter) and the bPF for the SV model (paticleFilterSVmodel).

**parameterEstimation.R**
Implements the PMH algorithm for the LGSS model (particleMetropolisHastings), the SV model (particleMetropolisHastingsSVModel) and the reparameterised SV model (particleMetropolisHastingsSVModelReparameterised). Moreover, a script (makePlotsParticleMetropolisHastingsSVModel) is included to generate the Figures presented in the paper using the output of the PMH algorithm.


Saved results (savedWorkspaces/ and figures/)
--------------
**savedWorkspaces/** Saved copies of the workspace after running the corresponding code. These outputs are used to generate all the results in the aper. Can be used to directly recreate the plots in the tutorial by setting the flags loadSavedWorkspace and savePlotToFile to TRUE.


Adapting the code for another model
--------------
The code provided in *helpers/stateInference.R* and *helpers/parameterInferernce.R* is quite general. To adapt this code for your own model, you can start with the code in *example3-sv.R* together with the functions *particleFilterSVmodel* and *particleMetropolisHastings* from the helpers. 

### Particle filter
In the particle filter, you need to change the lines connected to: (i) the sampling of the initial state, (ii) the propagation of particles and (iii) the computation of the weights. For (i), you need to rewrite:
``` R
  particles[, 1] <- rnorm(noParticles, mu, sigmav / sqrt(1 - phi^2))
```
to fit your model. Two simple choices are to make use of the stationary distribution of the state (as is done for the SV model) computed by hand or to initialise all particles to some value (as is done in the LGSS model) by:
``` R
  particles[, 1] <- initialState
```
where *initialState* is provided by the user. The particle filter usually quite rapidly converges to the state in this case if the state process quickly forgets its past (mixes well).

For (ii), you need to change:
``` R
    part1 <- mu + phi * (particles[newAncestors, t - 1] - mu)
    part2 <- rnorm(noParticles, 0, sigmav)
    particles[, t] <- part1 + part2
```
to something else. For the bPF, this corresponds to the state process of your state-space model.

For (iii), you need to change:
``` R
    weights[, t] <- dnorm(y[t - 1], 0, exp(particles[, t] / 2), log = TRUE)
```
to something else. For the bPF, this corresponds to the observation process of your state-space model.

Finally, note that the particle filter implemetation can only be used for state-space models where the state and observation are scalar. However, it is quite straightforward to make use of particle filtering when the state and/or observations are multivariate. It is basically only bookkeeping. If the dimension of the state is larger than say 5, good proposals are usually required to not run into the curse of dimensionality. This is a hot current research topic in the particle filtering literature.

### Particle Metropolis-Hastings
The implemenation of the PMH algorithm is general and does not require any larger changes if the model is changed. The dimensionality of the variables *xHatFiltered*, *xHatFilteredProposed*, *theta* and *thetaProposed* needs to be altered to match the dimensionality of the state and the number of parameters in the new state-space model. Moreover, the initial value of theta and the proposal distribution need to be calibrated for your new model. The simplest way to do this is by so-called pilot runs. Set the initial value to something reasonable and stepSize to a diagonal matrix with quite small elements, so that you get at least some accepted proposed values. After the pilot run, adapt the proposal as is discussed in 6.3.2 and initialise the PMH algorithm in the estimated posterior mean. Repeat this one or two more times or until you are satisfied. 

It is known that this simple version of PMH performs bad when the number of parameters is larger than about 5. To circumvent this problem, see the suggestions in Sections 4.3 and 6. It is also discussed there how to choose the number of particles *noParticles* and the number of iterations *noIterations* to use in the PMH algorithm. *noBurnInIterations* can be selected by looking at the trace plot for when the Markov chain has reached its steady-state/stationarity. I usually use *noIterations* as 10,000 or 30,000 (with *noBurnInIterations* as 3,000 or 10,0000) to get good posterior estimates but these runs take time. Also, using *noParticles* as somewhere between *T* and 2*T* is a good place to start.
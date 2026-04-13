# Power_analysis


- [<span class="toc-section-number">1</span> *Power
  Analysis*](#power-analysis)
- [<span class="toc-section-number">2</span> Running power
  analysis](#running-power-analysis)
- [<span class="toc-section-number">3</span> output](#output)
- [<span class="toc-section-number">4</span> ](#section)

## *Power Analysis*

*A simulation-based power analysis was conducted for a Bayesian
multilevel mediation model fit with `brms` to evaluate the sample size
required to detect direct and indirect effects in a 14-day intensive
longitudinal design. The model included two daily predictors, **stress**
and **importance of the intention**, two daily mediators, **offloading**
and **monitoring**, and a binary end-of-day outcome, **prospective
memory**.*

*The data-generating model assumed Bernoulli outcomes and incorporated
both fixed and random effects. Predictor-to-mediator paths were set to
moderate effects on the log-odds scale (`a1 = -0.40`, `a2 = 0.40`,
`b1 = -0.40`, `b2 = 0.40`). Mediator-to-outcome paths were also set to
moderate effects (`d1 = 0.40`, `d2 = 0.40`), whereas direct
predictor-to-outcome effects were set to smaller values (`c1 = -0.20`,
`c2 = 0.20`).  
This setting produced indirect effects of approximately `0.16` for each
mediated pathway (i.e., `0.40 × 0.40 = 0.16`), corresponding to
small-to-moderate indirect effects on the log-odds scale.*

*The simulation further included person-level random intercepts for
offloading, monitoring, and prospective memory (`SD = 1.0`), as well as
modest random slopes (`SD = 0.10`) for the effects of offloading,
monitoring, and stress on prospective memory, and for the effects of
stress and importance on offloading and monitoring. This structure was
chosen to allow between-person variability while maintaining an
estimable multilevel model.*

*The entire code used to generate is showed here:*

``` r
# ---- Packages ----
library(MASS)    # mvrnorm
library(dplyr)
library(tidyr)
library(brms)

# ---- 1) Data simulator for 14-day intensive longigudinatl study ----

# the three dependent vars, PM, offloading, and monitoring are binary. 
# The approach that I am using is considering them as continuous first, 
# and then binarize them depending on their probability of occurring (marginal prevalence) -
simulate_ild_pm <- function( 
                            T_per = 14, # days
                            seed = 123,
                            # fixed effects (within-person daily)
                            intercept_pm = 0, # intercept pm (log odds) = p = 0.24 to convert, exp(log-odds)/(1(1+exp(log-odds)). this is chance  p =0.5 
                            # to convert probability to logit: log(p/(1-p))
                            
                            # coefficients
                            a1 = - 0.40,   # stress -> offloading
                            a2 = 0.40,   # importance -> offloading
                            b1 = - 0.40,   # stress -> monitoring
                            b2 = 0.40,   # importance -> monitoring
                            c1 = - 0.20,   # direct stress -> pm (log odds - medium = 0,.5)
                            c2 = 0.20,   # direct importance -> pm (log odds)
                            d1 = 0.40,  # offloading -> pm 
                            d2 = 0.40,   # monitoring -> pm
                            
                            # person-level random effect SDs a
                            re_sds = c(off = 1.0, mon = 1.0, pm = 1.0, # random intercepts
                                       pm_off_slope = 0.1, pm_mon_slope = 0.1, 
                                       pm_stress_slope = 0.1, a1_slope = 0.1, b1_slope = 0.1, 
                                       a2_slope = 0.1, b2_slope = 0.1),
                            
                            # covariance structure between person-level random effects (4x4)
                            re_corr_mat = NULL,
                            
                            # whether to include a random slope for offloading -> pm
                            include_pm_off_slope = TRUE,
                            
                            # AR structure for stress/importance (within-person temporal autocorr)
                            ar_stress = 0.35,
                            ar_importance = 0.35,
                            miss_prob = 0.0) {
  
  set.seed(seed)
  TT <- T_per
  
  # ---- build list of random effects 
  re_names_expected <- c("u_off","u_mon","u_pm", # random intercepts
                         "u_pm_off_slope","u_pm_mon_slope","u_pm_stress_slope", # slopes
                         "u_a1_slope","u_b1_slope","u_a2_slope","u_b2_slope") # slopes
  
  # ensure re_sds named; fill missing with 0 and warn
  if (is.null(names(re_sds))) names(re_sds) <- rep("", length(re_sds))
  
  # Map missing expected names to 0
  for (nm in re_names_expected) if (!(nm %in% names(re_sds))) {
    # try to accept shorter keys (e.g., "off" -> u_off)
    alt <- sub("^u_", "", nm)
    if (alt %in% names(re_sds)) {
      names(re_sds)[names(re_sds) == alt] <- nm
    } else {
      re_sds[[nm]] <- 0
      warning(sprintf("re_sds did not contain '%s' — defaulting SD=0 for this RE.", nm))
    }
  }
  
  # now construct sds_vec in the expected order
  sds_vec <- as.numeric(re_sds[re_names_expected])
  names(sds_vec) <- re_names_expected
  
  kRE <- length(sds_vec)
  
  # correlation matrix
  re_corr_mat <- diag(kRE)
  
  # create covariance matrix from sds and correlation matrix
  RE_cov <- diag(sds_vec) %*% re_corr_mat %*% diag(sds_vec)
  
  # eign value
  ev <- eigen(RE_cov, symmetric = TRUE)$values
  
  # intercept handling
  if (is.null(intercept_pm)) intercept_pm <- qlogis(intercept_prob)
  
  # Draw person-level random effects jointly: each person has vector (u_off, u_mon, u_pm, u_pm_off_slope)
  REs_full <- MASS::mvrnorm(N, mu = rep(0, length(sds_vec)), Sigma = RE_cov)                                                  
  
  colnames(REs_full) <- re_names_expected
  
  # storage
  dat_list <- vector("list", N)
  
  
  for (i in seq_len(N)) { # loop through particiapnts
    # simulate time series for exogenous predictors (stress & importance) AR(1)
    stress <- numeric(TT)
    importance <- numeric(TT)
    
    for (t in seq_len(TT)) {
      if (t == 1) {
        stress[t] <- rnorm(1, 0, 1)
        importance[t] <- rnorm(1, 0, 1)
      } else {
        stress[t] <- ar_stress * stress[t-1] + rnorm(1, 0, sqrt(1 - ar_stress^2))
        importance[t] <- ar_importance * importance[t-1] + rnorm(1, 0, sqrt(1 - ar_importance^2))
      }
    }
    
    # person-level REs
    # extract REs for person i (0 if SD==0)
    u_off <- REs_full[i,"u_off"]; u_mon <- REs_full[i,"u_mon"]; u_pm <- REs_full[i,"u_pm"]
    u_pm_off_slope <- REs_full[i,"u_pm_off_slope"]; u_pm_mon_slope <- REs_full[i,"u_pm_mon_slope"]
    u_pm_stress_slope <- REs_full[i,"u_pm_stress_slope"]
    u_a1_slope <- REs_full[i,"u_a1_slope"]; u_b1_slope <- REs_full[i,"u_b1_slope"]
    u_a2_slope <- REs_full[i,"u_a2_slope"]; u_b2_slope <- REs_full[i,"u_b2_slope"]
    
    off_v <- integer(T_per); mon_v <- integer(T_per); pm_v <- integer(T_per)
    
    
    for (t in seq_len(TT)) { # loop through the trials
      
      # Mediator linear predictors (logit scale) include person intercept
      eta_off <- u_off + (a1 + u_a1_slope) * stress[t] + (a2 + u_a2_slope) * importance[t]
      p_off <- plogis(eta_off)
      off_t <- rbinom(1, 1, p_off)
      
      eta_mon <- u_mon + (b1 + u_b1_slope) * stress[t] + (b2 + u_b2_slope) * importance[t]
      p_mon <- plogis(eta_mon)
      mon_t <- rbinom(1, 1, p_mon)
      
      # PM linear predictor uses observed mediators (binary) and person REs
      # include random slope u_pm_off_slope if requested
      # PM eqn with person-specific slopes for stress and both mediators
      slope_off_term <- d1 + u_pm_off_slope
      slope_mon_term <- d2 + u_pm_mon_slope
      stress_term    <- c1 + u_pm_stress_slope
      
      eta_pm <- intercept_pm + u_pm + stress_term * stress[t] + c2 * importance[t] +
        slope_off_term * off_t + slope_mon_term * mon_t
      p_pm <- plogis(eta_pm)
      pm_t <- rbinom(1, 1, p_pm)
      
      
      # store
      off_v[t] <- off_t
      mon_v[t] <- mon_t
      pm_v[t]  <- pm_t
    }
    
    dat_list[[i]] <- data.frame(id = i, time = 1:T_per,
                                stress = stress, importance = importance,
                                offloading = off_v, monitoring = mon_v,
                                prospective_memory = pm_v)
  }
  
  dat <- bind_rows(dat_list)
  
  # optionally drop person-level RE columns for analysis (keeps only observed variables)
  dat <- dat %>% select(-starts_with("u_"))
  
  if (miss_prob > 0) {
    set.seed(seed + 1)
    dat <- dat[runif(nrow(dat)) > miss_prob, ]
  }
  
  return(dat)
}
```

*Power was evaluated across a range of sample sizes (from N=20 to
N=220), by repeatedly simulating datasets, fitting the model, and
calculating the proportion of replications in which each target
parameter was detected. On each iteration, a target parameter was
considered detected if the posterior probability of the effect being in
the expected direction exceeded 0.95. Specifically, for positive
directional hypotheses, detection was defined as the proportion of
posterior draws greater than zero exceeding 0.95, and for negative
directional hypotheses, detection was defined as the proportion of
posterior draws less than zero exceeding 0.95.*

## Running power analysis

A code running the simulations and calculating the power is the
following:

## output

*The resulting power curves showed that power increased with sample size
for all paths, but the mediator-to-outcome effects generally required
larger samples than the direct effects and predictor-to-mediator
effects.*

![](power_analysis_files/figure-commonmark/unnamed-chunk-3-1.png)

## 

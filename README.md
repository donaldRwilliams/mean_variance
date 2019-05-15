
<!-- README.md is generated from README.Rmd. Please edit that file -->

Beneath the Surface: Unearthing Within-Person Variability and Mean Relations with Bayesian Mixed Models
=======================================================================================================

This document reproduces the analyses, plots, model comparisons, etc. included in the paper. The following is in order of appearance.

The Mixed-Effects Location Scale Model
--------------------------------------

### Random Intercepts Only Model

A random intercepts model was first fitted to both the location and scale. For the *i*th person and *j*th trial in the congruent condition, the mean structure is defined as

*y*<sub>*i**j*</sub> = *β*<sub>0</sub> + *u*<sub>0*i*</sub> + *ϵ*<sub>*i**j*</sub>,

where *β*<sub>0</sub> is the fixed effect and *u*<sub>0*i*</sub> the individual deviation. More specifically, *β*<sub>0</sub> is the average of the individual means and for, say, the first subject (*i* = 1), the respective mean response time is *β*<sub>0</sub> + *u*<sub>01</sub>. This is not equivalent to estimating the empirical means, because of the hierarchical structure of the model. Before describing this aspect of the model, we must account for the \`\`errors''. While they are typically assumed to be normally distributed with a constant variance, this is not the case for the MELSM--i.e.,

*σ*<sub>*ϵ*<sub>*i**j*</sub></sub><sup>2</sup> = exp\[*η*<sub>0</sub> + *u*<sub>1*i*</sub>\].

Load the following packages and data:

``` r
library(brms)
library(dplyr)
library(ggplot2)
library(cowplot)
library(reshape)
library(devtools)
library(rethinking)

# read the data
ur <- "https://raw.githubusercontent.com/PerceptionCognitionLab/data0/master/contexteffects/FlankerStroopSimon/cleaning.R"
source_url(ur)
```

Estimate the following models. Note that two are mixed effects models, and are used for model comparison.

``` r
# melsm: mixed effects location scale model
# me: mixed effects model
fit_1 <- brm(bf(rt ~ 1 + (1|c|ID), sigma ~ 1 + (1|c|ID)), 
             data = stroop %>% filter(congruency == "incongruent"),
             inits = 0, cores = 2, 
             chains = 2, iter = 2000, 
             warmup = 1000)

# remove scale model
fit_2 <- brm(bf(rt ~ 1 + (1|c|ID)), 
             data = stroop %>% filter(congruency == "incongruent"),
             inits = 0, cores = 2, 
             chains = 2, iter = 2000, 
             warmup = 1000)

# congruent intercept only
fit_3 <- brm(bf(rt ~ 1 + (1|c|ID), sigma ~ 1 + (1|c|ID)), 
             data = stroop %>% filter(congruency == "congruent"),
             inits = 0, cores = 2, chains = 2, iter = 2000, 
             warmup = 1000)


# check to easy worry of ``over-fitting''
# remove scale model
fit_4 <- brm(bf(rt ~ 1 + (1|c|ID)), 
             data = stroop %>% filter(congruency == "congruent"),
             inits = 0, cores = 2, 
             chains = 2, iter = 2000, 
             warmup = 1000)
```

Compare with WAIC.

``` r
incong_waic <- waic(fit_1, fit_2)
cong_waic <- waic(fit_3, fit_4)
```

Incongruent comparison:

``` r
# difference
incong_waic$ic_diffs__
#>                    WAIC       SE
#> fit_1 - fit_2 -427.4586 76.32362

# SEs away from zero
abs(incong_waic$ic_diffs__[1]) / incong_waic$ic_diffs__[2]
#> [1] 5.600607
```

Congruent comparison:

``` r
# difference
cong_waic$ic_diffs__
#>                    WAIC       SE
#> fit_3 - fit_4 -810.2488 113.5126

# SEs away from zero
abs(cong_waic$ic_diffs__[1]) / cong_waic$ic_diffs__[2]
#> [1] 7.137966
```

Make plot 1 from the fitted objects. First for the incongruent model. Figure 1 (bottom):

``` r
# use "Times" font
windowsFonts(Times = windowsFont("Times New Roman"))

re_mean_in <- fit_1 %>% 
  data.frame() %>% 
  select(contains("r_ID")) %>% 
  select(-contains("sigma"))

fe_mean_in <- fit_1 %>% 
  data.frame() %>% 
  select(contains("b_Intercept")) 

re_mean_in <- (re_mean_in + fe_mean_in[,1]) 
colnames(re_mean_in) <- 1:121

# empirical estimates
emp_est_in <- stroop %>% 
  filter(congruency == "incongruent") %>% 
  group_by(ID) %>% 
  summarise(sd_emp = sd(rt), mean_emp = mean(rt))

set.seed(1)
random_draw <- sample(1:121, 60, replace = F)

plot_1A <- melt(re_mean_in) %>% 
  group_by(variable) %>% 
  # compute mean and intervals
  summarise(mu_mu = mean(value), 
            low = quantile(value, 0.05),
            up = quantile(value, 0.95)) %>% 
  # order small to large
  mutate(mu_emp = emp_est_in$mean_emp) %>%
  arrange(mu_mu) %>%
  mutate(index = as.factor(1:121), 
         dist_param = "mean", 
         outcome = "Incongruent") %>%
  filter(index %in% random_draw) %>%
  ggplot() +
  # fixed effect line
  geom_hline(yintercept = round(posterior_summary(fit_1, "b_")[1,1],2), 
             linetype = "twodash",
             alpha = 0.50) +
  # error bars
  geom_errorbar(aes(x = index, 
                    ymin = low, 
                    ymax = up), 
                width = 0.05) +
  # model based estimates
  geom_point(aes(x = index, 
                 y = mu_mu), 
             size = 2, 
             color = "#0072B2", 
             alpha = 0.75) +
  # empirical estimates
  geom_point(aes(x = index, 
                 y = mu_emp), 
             size = 2, 
             color = "#D55E00",
             alpha = 0.75) +
  # times font
  theme_bw(base_family = "Times") +
  # plot options
  theme(panel.grid.major.x =   element_blank(), 
        panel.grid.minor.y = element_blank(),
        axis.text.x = element_blank(),
        axis.title = element_text(size = 14),
        title = element_text(size = 14)) +
  scale_x_discrete(expand = c(0.01, 0.01)) +
  ylab("Mean") +
  xlab("Ascending Index") +
  ggtitle("Incongruent") +
  scale_y_continuous(breaks = c(0.6, 0.7, 0.77, 0.9, 1))

plot_1A
```

<img src="figures/README-unnamed-chunk-9-1.png" width="80%" style="display: block; margin: auto;" />

Plot 1B:

``` r
# extract posterior estimates
re_sigma_in <- fit_1 %>% 
  data.frame() %>% 
  select(contains("r_ID__sigma")) 

fe_sigma_in <- fit_1 %>% 
  data.frame() %>% 
  select(contains("b_sigma_Intercept")) 

re_sigma_in <- exp(re_sigma_in + fe_sigma_in[,1]) 
colnames(re_sigma_in) <- 1:121

# randomly sample 60 eople
set.seed(1)
random_draw <- sample(1:121, 60, replace = F)

# plot 1a
plot_1B <- melt(re_sigma_in) %>% 
  group_by(variable) %>% 
  # compute mean and intervals
  summarise(mu_sigma = mean(value), 
            low = quantile(value, 0.05),
            up = quantile(value, 0.95)) %>% 
  # order small to large
  mutate(sd_emp = emp_est_in$sd_emp) %>%
  arrange(mu_sigma) %>%
  mutate(index = as.factor(1:121), 
         dist_param = "SD", 
         outcome = "Incongruent") %>%
  filter(index %in% random_draw) %>%
  ggplot() +
  # fixed effect line
  geom_hline(yintercept = round(exp(posterior_summary(fit_1, "b_")[2,1]),2), 
             linetype = "twodash",
             alpha = 0.50) +
  # error bars
  geom_errorbar(aes(x = index, 
                    ymin = low, 
                    ymax = up), 
                width = 0.05) +
  # model based estimates
  geom_point(aes(x = index, 
                 y = mu_sigma), size = 2, 
             color = "#0072B2", 
             alpha = 0.75) +
  # empirical estimates
  geom_point(aes(x = index, 
                 y = sd_emp), 
             size = 2, 
             color = "#D55E00", 
             alpha = 0.75) +
  # times font
  theme_bw(base_family = "Times") +
  # plot options
  theme(panel.grid.major.x =   element_blank(), 
        panel.grid.minor.y = element_blank(),
        axis.text.x = element_blank(),
        axis.title = element_text(size = 14),
        title = element_text(size = 16)) +
  scale_x_discrete(expand = c(0.01, 0.01)) +
  ylab("Standard Deviation") +
  xlab("Ascending Index") +
  ggtitle("")

plot_1B
```

<img src="figures/README-unnamed-chunk-10-1.png" width="80%" style="display: block; margin: auto;" />

The following has the empirical and model based estimates in the same plot. Plot 1C:

``` r
dat_model_in <- data.frame(type = "Hierarchical",  
                           mean = colMeans(re_mean_in), 
                           sd = colMeans(re_sigma_in))


dat_data_in <- data.frame(type = "Empirical",  
                          mean = emp_est_in$mean_emp, 
                          sd = emp_est_in$sd_emp)


dat_plt_in <- rbind.data.frame(dat_model_in, dat_data_in)
dat_plt_in$ID <- 1:121

plot_1C <- ggplot(dat_plt_in, aes(y = sd, 
                                  x = mean, 
                                  color = type)) +
  geom_line(aes( group = ID), 
            color =  "black", 
            alpha = 0.25, 
            size = 1) +
  geom_point(size = 2, alpha = 0.75) +
  geom_smooth(method = "lm", se = F, show.legend = F) +
  scale_color_manual(values =c( "#0072B2", "#D55E00")) +
  theme_bw(base_family = "Times") +
  # plot options
  theme(panel.grid.minor.x =   element_blank(), 
        panel.grid.minor.y = element_blank(),
        axis.title = element_text(size = 14),
        title = element_text(size = 14),
        legend.text = element_text(size = 13),
        legend.justification=c(1,0), 
        legend.position=c(1,0),
        legend.title = element_blank(), 
        legend.background = element_rect(color = "black"),
        legend.margin=margin(c(1,1,1,1))) +
  xlab("Mean") +
  ylab("Standard Deviation") +
  guides(colour = guide_legend(override.aes = list(alpha = 1))) +
  ggtitle("") 

plot_incong <- plot_grid(plot_1A, plot_1B, plot_1C, nrow = 1)

# ggsave(plot = plot_incong, filename = "plot_1.pdf", width = 8, height = 5)
```

Continue making plot 1 from the fitted objects. Now for the congruent model. Figure 1 (top):

``` r
re_mean_con <- fit_3 %>% 
  data.frame() %>% 
  select(contains("r_ID")) %>% 
  select(-contains("sigma"))

fe_mean_con <- fit_3 %>% 
  data.frame() %>% 
  select(contains("b_Intercept")) 

re_mean_con <- (re_mean_con + fe_mean_con[,1]) 
colnames(re_mean_con) <- 1:121

# empirical estimates
emp_est_con <- stroop %>% 
  filter(congruency == "congruent") %>% 
  group_by(ID) %>% 
  summarise(sd_emp = sd(rt), mean_emp = mean(rt))

set.seed(1)
random_draw <- sample(1:121, 60, replace = F)


set.seed(1)
random_draw <- sample(1:121, 60, replace = F)

plot_1D <- melt(re_mean_con) %>% 
  group_by(variable) %>% 
  # compute mean and intervals
  summarise(mu_mu = mean(value), 
            low = quantile(value, 0.05),
            up = quantile(value, 0.95)) %>% 
  # order small to large
  mutate(mu_emp = emp_est_con$mean_emp) %>%
  arrange(mu_mu) %>%
  mutate(index = as.factor(1:121), 
         dist_param = "mean", 
         outcome = "Congruent") %>%
  filter(index %in% random_draw) %>%
  ggplot() +
  # fixed effect line
  geom_hline(yintercept = round(posterior_summary(fit_3, "b_")[1,1],2), 
             linetype = "twodash",
             alpha = 0.50) +
  # error bars
  geom_errorbar(aes(x = index, 
                    ymin = low, 
                    ymax = up), 
                width = 0.05) +
  
  # model based estimates
  geom_point(aes(x = index, 
                 y = mu_mu), 
             size = 2, 
             color = "#0072B2", 
             alpha = 0.75) +
  
  # empirical estimates
  geom_point(aes(x = index, 
                 y = mu_emp), 
             size = 2, 
             color = "#D55E00",
             alpha = 0.75) +
  
  # times font
  theme_bw(base_family = "Times") +
  
  # plot options
  theme(panel.grid.major.x =   element_blank(), 
        panel.grid.minor.y = element_blank(),
        axis.text.x = element_blank(),
        axis.title = element_text(size = 14),
        title = element_text(size = 14)) +
  scale_x_discrete(expand = c(0.01, 0.01)) +
  ylab("Mean") +
  xlab("Ascending Index") +
  ggtitle("Congruent") +
  scale_y_continuous(breaks = c(0.6, 0.71, 0.8, 0.9, 1))

plot_1D
```

<img src="figures/README-unnamed-chunk-12-1.png" width="80%" style="display: block; margin: auto;" />

``` r
re_sigma_con <- fit_3 %>% 
  data.frame() %>% 
  select(contains("r_ID__sigma")) 

fe_sigma_con <- fit_3 %>% 
  data.frame() %>% 
  select(contains("b_sigma_Intercept")) 

re_sigma_con <- exp(re_sigma_con + fe_sigma_con[,1]) 
colnames(re_sigma_con) <- 1:121

# empirical estimates
emp_est_con <- stroop %>% 
  filter(congruency == "congruent") %>% 
  group_by(ID) %>% 
  summarise(sd_emp = sd(rt), mean_emp = mean(rt))

set.seed(1)
random_draw <- sample(1:121, 60, replace = F)

plot_1E <- melt(re_sigma_con) %>% 
  group_by(variable) %>% 
  # compute mean and intervals
  summarise(mu_sigma = mean(value), 
            low = quantile(value, 0.05),
            up = quantile(value, 0.95)) %>% 
  # order small to large
  mutate(sd_emp = emp_est_con$sd_emp) %>%
  arrange(mu_sigma) %>%
  mutate(index = as.factor(1:121), 
         dist_param = "SD", 
         outcome = "Congruent") %>%
  filter(index %in% random_draw) %>%
  ggplot() +
  # fixed effect line
  geom_hline(yintercept = round(exp(posterior_summary(fit_3, "b_")[2,1]),2), 
             linetype = "twodash",
             alpha = 0.50) +
  # error bars
  geom_errorbar(aes(x = index, 
                    ymin = low, 
                    ymax = up),width = 0.05) +
  # model based estimates
  geom_point(aes(x = index, 
                 y = mu_sigma), size = 2, 
             color = "#0072B2", 
             alpha = 0.75) +
  # empirical estimates
  geom_point(aes(x = index, 
                 y = sd_emp), 
             size = 2, 
             color = "#D55E00", 
             alpha = 0.75) +
  # times font
  theme_bw(base_family = "Times") +
  # plot options
  theme(panel.grid.major.x =   element_blank(), 
        panel.grid.minor.y = element_blank(),
        axis.text.x = element_blank(),
        axis.title = element_text(size = 14),
        title = element_text(size = 14)) +
  scale_x_discrete(expand = c(0.01, 0.01)) +
  ylab("Standard Deviation") +
  xlab("Ascending Index") +
  ggtitle("") +
  scale_y_continuous(breaks = c(0.1, 0.17, 0.3))

plot_1E
```

<img src="figures/README-unnamed-chunk-13-1.png" width="80%" style="display: block; margin: auto;" />

Model based and emprical plot:

``` r
dat_model_con <- data.frame(type = "Hierarchical",  
                            mean = colMeans(re_mean_con), 
                            sd = colMeans(re_sigma_con))


dat_data_con <- data.frame(type = "Empirical",  
                           mean = emp_est_con$mean_emp, 
                           sd = emp_est_con$sd_emp)


dat_plt_con <- rbind.data.frame(dat_model_con, dat_data_con)
dat_plt_con$ID <- 1:121

plot_1F <- ggplot(dat_plt_con, aes(y = sd, 
                                   x = mean, 
                                   color = type)) +
  geom_line(aes( group = ID), 
            color =  "black", 
            alpha = 0.25, 
            size = 1) +
  # points
  geom_point(size = 2, 
             alpha = 0.75) +
  # add fitted line
  geom_smooth(method = "lm", 
              se = F, 
              show.legend = F) +
  # color blind pallette
  scale_color_manual(values =c( "#0072B2", "#D55E00"), 
                     name = "") +
  # fav theme with times font
  theme_bw(base_family = "Times") +
  # plot options
  theme(panel.grid.minor.x =   element_blank(), 
        panel.grid.minor.y = element_blank(),
        axis.title = element_text(size = 14),
        title = element_text(size = 16),
        legend.position = "none") +
  xlab("Mean") +
  ylab("Standard Deviation") +
  # remove transparency from legend points
  guides(colour = guide_legend(override.aes = list(alpha=1))) +
  ggtitle("") 


plot_cong <- plot_grid(plot_1D, plot_1E, plot_1F, nrow = 1)
plot_1 <- plot_grid(plot_cong, plot_incong, nrow = 2)

plot_1
```

<img src="figures/README-unnamed-chunk-14-1.png" width="80%" style="display: block; margin: auto;" />

``` r
ggsave(filename = "plot_1.pdf", plot = plot_1, width = 9.5, height = 6.5)
```

We also breifly compared the correlation between the model based and emprical means and standard deviations. For the incongruent responses:

``` r
# correlate incongruent M's and SD's

# emprical
emp_cor_in <- cor(dat_data_in[,2:3])[1,2]

# bootstrap (n = 121)
boot_in <- replicate(1000, cor(dat_data_in[sample(1:121, 121, replace = T),2:3])[1,2])

# boostrap SE
emp_cor_sd_in <- sd(boot_in)

# model mean
model_cor_in <- mean(posterior_samples(fit_1, pars = "cor")[,1])

# model SE (post SD)
model_cor_sd_in <- sd(posterior_samples(fit_1, pars = "cor")[,1])

# data frame
dat_in <- data.frame(type = c("emprical", "model"), 
           cor = c(emp_cor_in, model_cor_in),
           cor_sd = c(emp_cor_sd_in, model_cor_sd_in))
# results
dat_in
#>       type       cor     cor_sd
#> 1 emprical 0.6123496 0.05915010
#> 2    model 0.6684826 0.06196056
```

Now for the congruent responses:

``` r
# correlate congruent M's and SD's

# empirical
emp_cor_con <- cor(dat_data_con[,2:3])[1,2]

# boostrap (n = 121)
boot_con <- replicate(1000, cor(dat_data_con[sample(1:121, 121, replace = T),2:3])[1,2])

# boostrap SE
emp_cor_sd_con <- sd(boot_con)

# model mean
model_cor_con <- mean(posterior_samples(fit_3, pars = "cor")[,1])

# model SE (post SD)
model_cor_sd_con <- sd(posterior_samples(fit_3, pars = "cor")[,1])

# data frame
dat_con <- data.frame(type = c("emprical", "model"), 
           cor = c(emp_cor_con, model_cor_con),
           cor_sd = c(emp_cor_sd_con, model_cor_sd_con))

# results
dat_con
#>       type       cor     cor_sd
#> 1 emprical 0.6626116 0.05772447
#> 2    model 0.7146246 0.05241439
```

Thus far the work showed how the mixed-effects location scale model can estimate mean-variance relations. This lead naturally to testing the correlations. The following visualizes the LKJ prior distribution for the partial correlations:

``` r
n_samps <- 10000
prior_samples  <- lapply(1:4, function(x) rethinking::rlkjcorr(n_samps, K = 4, eta = x)[,,1][,2])

lkj_dat <- data.frame(nu = as.factor(rep(1:4, each = n_samps)), 
                      sample = unlist(prior_samples))

lkj_dat %>% 
  ggplot() +
  # for linetype
  geom_line(stat = "density", 
            aes(linetype = nu, x = sample), 
            adjust = 1.5) +
  theme_bw(base_family = "Times") +
  # plot options
  theme(panel.grid.minor.x = element_blank(), 
        panel.grid.minor.y = element_blank(),
        axis.title = element_text(size = 14),
        # top right legend
        legend.justification=c(1,1), 
        legend.position=c(1,1),
        legend.background = element_rect(color = "black")) +
  xlab( expression("Marginal Prior Distribution"~ italic(rho[i][j]))) +
  ylab("Density") +
  scale_linetype_manual(name = expression("  "~italic(nu)), 
                        values = c("solid", "longdash", "dotted", "dotdash"))
```

<img src="figures/README-unnamed-chunk-17-1.png" width="80%" style="display: block; margin: auto;" />

We proceed to fit the full models:

``` r
form <- brmsformula(rt ~ congruency + (congruency |c| ID), 
                           sigma  ~ congruency + (congruency |c| ID))



priors <- c(
  # correlations
  set_prior("lkj(2)", class = "cor"),
  # location random SD
  set_prior("normal(0,0.25)", 
            class = "sd", 
            coef = "congruencyincongruent", 
            group = "ID"),
  # location random SD
  set_prior("normal(0,0.25)",
            class = "sd", 
            coef = "Intercept",
            group = "ID"),
  # scale random SD
  set_prior("normal(0,1)", 
            class = "sd",
            coef = "congruencyincongruent", 
            group = "ID", 
            dpar = "sigma"), 
  # scale random SD
  set_prior("normal(0,1)", 
            class = "sd", 
            coef = "Intercept", 
            group = "ID", 
            dpar = "sigma"), 
  # fixed effect priors
  set_prior("normal(0,5)", 
            class = "b"),
  set_prior("normal(0,5)", 
            class = "b", 
            dpar = "sigma"),
  set_prior("normal(0,5)", 
            class = "Intercept"),
  set_prior("normal(0,5)", 
            class = "Intercept", 
            dpar = "sigma"))

fit_stroop <- brm(form, data = stroop, 
                  inits = 0, cores = 4, 
                  chains = 4, iter = 3500, 
                  warmup = 1000, prior = priors)


fit_flank <- brm(form_stroop, data = flanker, 
                 inits = 0, cores = 4, 
                 chains = 4, iter = 3500, 
                 warmup = 1000,  
                 prior = priors)

save(fit_stroop, file = "fit_stroop.Rdata")
save(fit_flank, file = "fit_flank.Rdata")
```

Now the customary mixed-effects model (no scale model):

``` r
form_me <- brmsformula(rt ~ congruency + (congruency |c| ID))


# ME priors 
priors_me <- c(
  set_prior("lkj(2)", class = "cor"),
  # location random SD
  set_prior("normal(0,0.25)", 
            class = "sd", 
            coef = "Intercept", 
            group = "ID"),
  set_prior("normal(0,0.25)", 
            class = "sd",
            coef = "congruencyincongruent", 
            group = "ID"), 
                  # fixed effect priors
                  set_prior("normal(0,5)", 
                            class = "b"),
                  set_prior("normal(0,5)", 
                            class = "Intercept"))

fit_stroop_me <- brm(form_me, data = stroop, 
                  inits = 0, cores = 4, 
                  chains = 4, iter = 3500, 
                  warmup = 1000, prior = priors_me)


fit_flank_me <- brm(form_me, data = flanker, 
                    inits = 0, cores = 4, 
                    chains = 4, iter = 3500, 
                    warmup = 1000,  
                    prior = priors_me)
```

To address concerns of overfitting, we compared these models with WAIC. The Stroop task:

``` r
# stroop task
stroop_waic$ic_diffs__
#>                                 WAIC       SE
#> fit_stroop - fit_stroop_me -1370.327 138.5454

# SEs away from zero
abs(stroop_waic$ic_diffs__[1]) / stroop_waic$ic_diffs__[2]
#> [1] 9.89082
```

The Flanker task:

``` r
# flanker task
flank_waic$ic_diffs__
#>                               WAIC       SE
#> fit_flank - fit_flank_me -1142.353 203.2134

# SEs away from zero
abs(flank_waic$ic_diffs__[1]) / flank_waic$ic_diffs__[2]
#> [1] 5.621443
```

Note that this was done to show the MELSM is preferred over the customary mixed-effects model. In our experience, we have never come across an example in which the MELSM did not have (much) lower WAIC (or LOO).

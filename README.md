
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

The following contructs Figure 3 (Panel A). Start with the Stroop task. The MELSM is plotted with:

``` r
# point estimates from linear regression
coefs_nonreg <- unlist(lapply(1:121, function(x) coefficients(lm(rt ~ congruency, data = subset(stroop, ID == x)))[2]))

# "Stroop effect"
melsm_stroop <- round(posterior_summary(fit_stroop, pars = "b_")[3,1], 3)



dat_stroop <- data.frame(y = coefs_nonreg, 
                         index= 1:121) %>% 
              arrange(y) %>%
              mutate(index2 = as.factor(1:121))

# melsm
melsm_stroop_plot <- coef(fit_stroop, probs = c(0.05, 0.95))$ID[,,2] %>% 
  data.frame() %>%
  arrange(Estimate) %>%
  mutate(sig = as.factor(ifelse(Q5 < melsm_stroop & Q95 > melsm_stroop, 0, 1)),
         index = as.factor(1:121),
         type = "MELSM") %>%
  ggplot() +
  facet_grid(~ type ) +
  geom_errorbar(aes(x = index, 
                    ymin = Q5, 
                    ymax = Q95, 
                    color = sig), 
                show.legend = F,
                width = 0) +
  geom_hline(yintercept = melsm_stroop,
             alpha = 0.50,
             linetype = "twodash") +
  geom_hline(yintercept = 0,
             alpha = 0.50,
             linetype = "dotted") +
  geom_point(aes(x = index, 
                 y = Estimate,
                 group = sig),
             size = 2, 
             alpha = 0.75) +
  theme_bw(base_family = "Times") +
  # plot options
  theme(panel.grid.major.x = element_blank(), 
        panel.grid.minor.y = element_blank(),
        axis.text.x = element_blank(),
        axis.title = element_text(size = 14),
        title = element_text(size = 14),
        strip.background = element_rect(fill = "grey94"),
        strip.text = element_text(size = 14)) +
  ylab(expression(atop(italic(beta[1])* " + " *italic(u[1][i])*"  (ms)", "Stroop Effect on RT"))) +
  scale_color_manual(values = c("#009E73", "#CC79A7")) +
  xlab("") +
  xlab("") + 
  geom_path(inherit.aes = F, 
            data = dat_stroop, 
            aes(x = as.numeric(index2), 
                y = y), 
            color = "grey",
            size = 1.5, alpha = 0.75) +
  scale_y_continuous(breaks = seq(-.1, .2, 0.05), 
                     labels = seq(-.1, .2, 0.05) * 1000) +
  scale_x_discrete(expand = c(0.015, 0.015)) +
  ggtitle("")

#melsm_stroop_plot
```

The ME (mixed-effect model) is plotted with:

``` r
# Stroop effect
me_stroop <- round(posterior_summary(fit_stroop_me, pars = "b_")[2,1], 3)

# plot
me_stroop_plot <- coef(fit_stroop_me, probs = c(0.05, 0.95))$ID[,,2] %>% 
  data.frame() %>%
  arrange(Estimate) %>%
  mutate(sig = as.factor(ifelse(Q5 < me_stroop & Q95 > me_stroop, 0, 1)),
         index = as.factor(1:121),
         type = "MEM") %>%
  ggplot() +
  facet_grid(~ type ) +
  geom_errorbar(aes(x = index, 
                    ymin = Q5, 
                    ymax = Q95, 
                    color = sig), 
                show.legend = F,
                width = 0) +
  geom_hline(yintercept = me_stroop,
             alpha = 0.50,
             linetype = "twodash") +
  geom_hline(yintercept = 0,
             alpha = 0.50,
             linetype = "dotted") +
  geom_point(aes(x = index, 
                 y = Estimate,
                 group = sig),
             size = 2, 
             alpha = 0.75) +
  theme_bw(base_family = "Times") +
  # plot options
  theme(panel.grid.major.x = element_blank(), 
        panel.grid.minor.y = element_blank(),
        axis.text.x = element_blank(),
        axis.title = element_text(size = 14),
        title = element_text(size = 14),
        strip.background = element_rect(fill = "grey94"),
        strip.text = element_text(size = 14)) +
  ylab(expression(atop(italic(beta[1])* " + " *italic(u[1][i])*"  (ms)", "Stroop Effect on RT"))) +
  scale_color_manual(values = c("#009E73", "#CC79A7")) +
  xlab("") +
  xlab("") + 
  geom_path(inherit.aes = F, 
            data = dat_stroop, 
            aes(x = as.numeric(index2), 
                y = y), 
            color = "grey",
            size = 1.5, 
            alpha = 0.75) +

  scale_y_continuous(breaks = seq(-.1, .2, 0.05), 
                     labels = seq(-.1, .2, 0.05) * 1000) +
  ggtitle("Stroop") +
  scale_x_discrete(expand = c(0.015, 0.015)) 

#me_stroop_plot
```

Combine plots:

``` r
plot_stroop <- plot_grid(me_stroop_plot, melsm_stroop_plot)

plot_stroop
```

<img src="figures/README-unnamed-chunk-26-1.png" width="80%" style="display: block; margin: auto;" />

Next the Flanker task. The MELSM is plotted with:

``` r
coefs_nonreg <- unlist(lapply(1:121, function(x) coefficients(lm(rt ~ congruency, data = subset(flanker, ID == x)))[2]))

# "flanker effect"
melsm_flanker <- round(posterior_summary(fit_flank, pars = "b_")[3,1], 3)



dat_flanker <- data.frame(y = coefs_nonreg, 
                         index= 1:121) %>% 
              arrange(y) %>%
              mutate(index2 = as.factor(1:121))

# melsm
melsm_flanker_plot <- coef(fit_flank, probs = c(0.05, 0.95))$ID[,,2] %>% 
  data.frame() %>%
  arrange(Estimate) %>%
  mutate(sig = as.factor(ifelse(Q5 < melsm_flanker & Q95 > melsm_flanker, 0, 1)),
         index = as.factor(1:121),
         type = "MELSM") %>%
  ggplot() +
  facet_grid(~ type ) +
  geom_errorbar(aes(x = index, 
                    ymin = Q5, 
                    ymax = Q95, 
                    color = sig), 
                show.legend = F,
                width = 0) +
  geom_hline(yintercept = melsm_flanker,
             alpha = 0.50,
             linetype = "twodash") +
  geom_hline(yintercept = 0,
             alpha = 0.50,
             linetype = "dotted") +
  geom_point(aes(x = index, 
                 y = Estimate,
                 group = sig),
             size = 2, 
             alpha = 0.75) +
  theme_bw(base_family = "Times") +
  # plot options
  theme(panel.grid.major.x = element_blank(), 
        panel.grid.minor.y = element_blank(),
        axis.text.x = element_blank(),
        axis.title = element_text(size = 14),
        title = element_text(size = 14),
        strip.background = element_rect(fill = "grey94"),
        strip.text = element_text(size = 14)) +
  ylab(expression(atop(italic(beta[1])* " + " *italic(u[1][i])*"  (ms)", "Flanker Effect on RT"))) +
  scale_color_manual(values = c("#009E73", "#CC79A7")) +
  xlab("") +
  xlab("") + 
  geom_path(inherit.aes = F, 
            data = dat_flanker, 
            aes(x = as.numeric(index2), 
                y = y), 
            color = "grey",
            size = 1.5, alpha = 0.75) +
  scale_y_continuous(breaks = seq(-.1, .2, 0.05), 
                     labels = seq(-.1, .2, 0.05) * 1000) +
  scale_x_discrete(expand = c(0.015, 0.015)) +
  ggtitle("")

# melsm_flanker_plot
```

Now the Flanker mixed model:

``` r
me_flanker <- round(posterior_summary(fit_flank_me, pars = "b_")[2,1], 3)

# me
me_flanker_plot <- coef(fit_flank_me, probs = c(0.05, 0.95))$ID[,,2] %>% 
  data.frame() %>%
  arrange(Estimate) %>%
  mutate(sig = as.factor(ifelse(Q5 < me_flanker & Q95 > me_flanker, 0, 1)),
         index = as.factor(1:121),
         type = "MELSM") %>%
  ggplot() +
  facet_grid(~ type ) +
  geom_errorbar(aes(x = index, 
                    ymin = Q5, 
                    ymax = Q95, 
                    color = sig), 
                show.legend = F,
                width = 0) +
  geom_hline(yintercept = me_flanker,
             alpha = 0.50,
             linetype = "twodash") +
  geom_hline(yintercept = 0,
             alpha = 0.50,
             linetype = "dotted") +
  geom_point(aes(x = index, 
                 y = Estimate,
                 group = sig),
             size = 2, 
             alpha = 0.75) +
  theme_bw(base_family = "Times") +
  # plot options
  theme(panel.grid.major.x = element_blank(), 
        panel.grid.minor.y = element_blank(),
        axis.text.x = element_blank(),
        axis.title = element_text(size = 14),
        title = element_text(size = 14),
        strip.background = element_rect(fill = "grey94"),
        strip.text = element_text(size = 14)) +
  ylab(expression(atop(italic(beta[1])* " + " *italic(u[1][i])*"  (ms)", "Flanker Effect on RT"))) +
  scale_color_manual(values = c("#009E73", "#CC79A7")) +
  xlab("") +
  xlab("") + 
  geom_path(inherit.aes = F, 
            data = dat_flanker, 
            aes(x = as.numeric(index2), 
                y = y), 
            color = "grey",
            size = 1.5, alpha = 0.75) +
  scale_y_continuous(breaks = seq(-.1, .2, 0.05), 
                     labels = seq(-.1, .2, 0.05) * 1000) +
  scale_x_discrete(expand = c(0.015, 0.015)) +
  ggtitle("Flanker")
```

Combine plots:

``` r
# flanker plot
plot_flanker <-  plot_grid(me_flanker_plot, melsm_flanker_plot)

# panel A
A <- plot_grid(plot_stroop, plot_flanker, nrow = 2)

plot_flanker
```

<img src="figures/README-unnamed-chunk-29-1.png" width="80%" style="display: block; margin: auto;" />

The above is Panel A. Panel B highlights shrinkage from the Flanker model:

``` r
coefs_nonreg <- lapply(1:121, function(x) coefficients(lm(rt ~ congruency, data = subset(flanker, ID == x))))

non_pooled <- do.call(rbind.data.frame, coefs_nonreg)
colnames(non_pooled) <- c("Congruent RT", "Flanker Effect on RT")
non_pooled$type <- "Empirical"
non_pooled$id <- 1:121

#melsm
pooled_melsm <- coef(fit_flank)$ID[,1,1:2] %>% data.frame()
colnames(pooled_melsm) <- c("Congruent RT", "Flanker Effect on RT")
pooled_melsm$type <- "Hierarchical"
pooled_melsm$id <- 1:121

#me
pooled_me <- coef(fit_flank_me)$ID[,1,1:2] %>% data.frame()
colnames(pooled_me) <-  c("Congruent RT", "Flanker Effect on RT")
pooled_me$type <- "Hierarchical"
pooled_me$id <- 1:121

pooled_melsm <- rbind.data.frame(non_pooled, pooled_melsm)
pooled_melsm$model <- "MELSM"

pooled_me <- rbind.data.frame(non_pooled, pooled_me)
pooled_me$model <- "MEM"

# for legend
leg <- ggplot(pooled_melsm, aes(y = pooled_melsm$`Congruent RT`, 
                                 x = pooled_melsm$`Flanker Effect on RT`)) +
  geom_line(aes(group = id), 
            alpha = 0.25, 
            size = 1) +
  geom_point(aes(group = id, 
                 color = type), 
             size = 2.5, 
             alpha = 0.75) +
  facet_grid(~ model) +
  theme_bw(base_family = "Times") +
  theme(panel.grid.major.x = element_blank(), 
        panel.grid.minor.y = element_blank(),
        axis.title = element_text(size = 14),
        title = element_text(size = 14),
        strip.background = element_rect(fill = "grey94"),
        strip.text = element_text(size = 14),
        legend.text = element_text(size = 14),
        legend.position = "top") +
  guides(colour = guide_legend(override.aes = list(alpha=1))) +
  scale_color_manual(values =c( "#0072B2", "#D55E00"), name = "")

leg_pooled <- get_legend(leg)


melsm_pooled_plot <- ggplot(pooled_melsm, aes( y = pooled_melsm$`Congruent RT`, 
                                               x = pooled_melsm$`Flanker Effect on RT`)) +
  geom_line(aes(group = id), alpha = 0.25, size = 1) +
  geom_point(aes(group = id, color = type), size = 2.5, alpha = 0.75) +
  facet_grid(~ model) +
  theme_bw(base_family = "Times") +
  theme(panel.grid.major.x = element_blank(), 
        panel.grid.minor.y = element_blank(),
        axis.title = element_text(size = 14),
        title = element_text(size = 14),
        strip.background = element_rect(fill = "grey94"),
        strip.text = element_text(size = 14),
        legend.text = element_text(size = 14),
        legend.position = "none") +
  guides(colour = guide_legend(override.aes = list(alpha=1))) +
  scale_color_manual(values =c( "#0072B2", "#D55E00"), name = "") +
  scale_y_continuous(limits = c(0.4, 0.9)) +
  ggtitle("") +
  ylab("Congruent RT") +
  xlab("Flanker Effect on RT")


me_pooled_plot <- ggplot(pooled_me, aes( y = pooled_me$`Congruent RT`, 
                                         x = pooled_me$`Flanker Effect on RT`)) +
  geom_line(aes(group = id), alpha = 0.25, size = 1) +
  geom_point(aes(group = id, color = type), size = 2.5, alpha = 0.75) +
  facet_grid(~ model) +
  theme_bw(base_family = "Times") +
  theme(panel.grid.major.x = element_blank(), 
        panel.grid.minor.y = element_blank(),
        axis.title = element_text(size = 14),
        title = element_text(size = 14),
        strip.background = element_rect(fill = "grey94"),
        strip.text = element_text(size = 14),
        legend.text = element_text(size = 14),
        legend.position = "none") +
  guides(colour = guide_legend(override.aes = list(alpha=1))) +
  scale_color_manual(values =c( "#0072B2", "#D55E00"), name = "") +
  ggtitle("Flanker") +
  ylab("Congruent RT") +
  xlab("Flanker Effect on RT")

# plot B
B <- plot_grid("", me_pooled_plot, 
               melsm_pooled_plot, "", 
               nrow = 1, 
               rel_widths = c(1.5,8, 8, 1))

# add legend to plot B
B <- plot_grid(leg_pooled, B, 
               nrow = 2, 
               rel_heights = c(1,10))

# combined plot in paper
A_and_B <- plot_grid(A, "", B, nrow = 3, rel_heights = c(1.5,.1,1))
```

Panel A:

``` r
A
```

<img src="figures/README-unnamed-chunk-31-1.png" width="80%" style="display: block; margin: auto;" />

Panel B:

``` r
B
```

<img src="figures/README-unnamed-chunk-32-1.png" width="80%" style="display: block; margin: auto;" />

Now Figure 4. First scale intercept for the Stroop task:

``` r

# scale intercepts
int_scale_stroop <- posterior_summary(fit_stroop, pars = "b_")[2,1]
int_scale_flank <- posterior_summary(fit_flank, pars = "b_")[2,1]

# scale effects
stroop_scale_stroop <- posterior_summary(fit_stroop, pars = "b_")[4,1]
flank_scale_flank <- posterior_summary(fit_flank, pars = "b_")[4,1]

int_scale_stroop_plot <- coef(fit_stroop, probs = c(0.05, 0.95))$ID[,,3] %>% exp(.) %>%
  data.frame() %>%
  arrange(Estimate) %>%
  mutate(sig = as.factor(ifelse(Q5 < exp(int_scale_stroop) & Q95 > exp(int_scale_stroop), 0, 1)),
         index = as.factor(1:121),
         type = "Stroop") %>%
  ggplot() +
  facet_grid(~ type ) +
  geom_errorbar(aes(x = index, 
                    ymin = Q5, 
                    ymax = Q95, 
                    color = sig), 
                show.legend = F,
                width = 0) +
  geom_hline(yintercept = exp(int_scale_stroop),
             alpha = 0.50,
             linetype = "twodash") +
  geom_point(aes(x = index, 
                 y = Estimate,
                 group = sig),
             size = 2, 
             alpha = 0.75) +
  theme_bw(base_family = "Times") +
  # plot options
  theme(panel.grid.major.x = element_blank(), 
        panel.grid.minor.y = element_blank(),
        axis.text.x = element_blank(),
        axis.title = element_text(size = 14),
        title = element_text(size = 14),
        strip.background = element_rect(fill = "grey94"),
        strip.text = element_text(size = 14)) +
  ylab(expression(atop(italic(eta[0])* " + " *italic(u[2][i])* "  (SD)", "Congruent IIV"))) +
  scale_color_manual(values = c("#009E73", "#CC79A7")) +
  xlab("") +
  scale_y_continuous(limits = c(0, .6), 
                     labels = seq(0, 0.5, .1), 
                     breaks = seq(0, 0.5, .1))

int_scale_stroop_plot
```

<img src="figures/README-unnamed-chunk-33-1.png" width="80%" style="display: block; margin: auto;" />

Second scale intercept for the Flanker task:

``` r
# plot
int_scale_flank_plot <- coef(fit_flank, probs = c(0.05, 0.95))$ID[,,3] %>% exp(.) %>%
  data.frame() %>%
  arrange(Estimate) %>%
  mutate(sig = as.factor(ifelse(Q5 < exp(int_scale_flank) & Q95 > exp(int_scale_flank), 0, 1)),
         index = as.factor(1:121),
         type = "Flanker") %>%
  ggplot() +
  facet_grid(~ type ) +
  geom_errorbar(aes(x = index, 
                    ymin = Q5, 
                    ymax = Q95, 
                    color = sig), 
                show.legend = F,
                width = 0) +
  geom_hline(yintercept = exp(int_scale_flank),
             alpha = 0.50,
             linetype = "twodash") +
  geom_point(aes(x = index, 
                 y = Estimate,
                 group = sig),
             size = 2, 
             alpha = 0.75) +
  theme_bw(base_family = "Times") +
  # plot options
  theme(panel.grid.major.x = element_blank(), 
        panel.grid.minor.y = element_blank(),
        axis.text.x = element_blank(),
        axis.title = element_text(size = 14),
        title = element_text(size = 14),
        strip.background = element_rect(fill = "grey94"),
        strip.text = element_text(size = 14)) +
  ylab(expression(atop(italic(eta[0])* " + " *italic(u[2][i])* "  (SD)", "Congruent IIV"))) +
  scale_color_manual(values = c("#009E73", "#CC79A7")) +
  xlab("") +
  scale_y_continuous(limits = c(0, .6), 
                     labels = seq(0, 0.5, .1), 
                     breaks = seq(0, 0.5, .1))

int_scale_flank_plot
```

<img src="figures/README-unnamed-chunk-34-1.png" width="80%" style="display: block; margin: auto;" />

Now the "Stroop/Flanker" effects on IIV:

``` r
# % change
labs <- c("-63 %", "-39 %", "0 %", "+65 %", "+172 %")

stroop_scale_stroop_plot <- coef(fit_stroop, probs = c(0.05, 0.95))$ID[,,4] %>% 
  data.frame() %>%
  arrange(Estimate) %>%
  mutate(sig = as.factor(ifelse(Q5 < stroop_scale_stroop & Q95 > stroop_scale_stroop, 0, 1)),
         index = as.factor(1:121),
         type = "MELSM") %>%
  ggplot() +
  facet_grid(~ type ) +
  geom_errorbar(aes(x = index, 
                    ymin = Q5, 
                    ymax = Q95, 
                    color = sig), 
                show.legend = F,
                width = 0) +
  geom_hline(yintercept = stroop_scale_stroop,
             alpha = 0.50,
             linetype = "twodash") +
  geom_hline(yintercept = 0,
             alpha = 0.50,
             linetype = "dotted") +
  geom_point(aes(x = index, 
                 y = Estimate,
                 group = sig),
             size = 2, 
             alpha = 0.75) +
  theme_bw(base_family = "Times") +
  # plot options
  theme(panel.grid.major.x =element_blank(), 
        panel.grid.minor.y = element_blank(),
        axis.text.x = element_blank(),
        axis.title = element_text(size = 14),
        title = element_text(size = 14),
        strip.background = element_blank(),
        strip.text = element_blank()) +
  ylab(expression(atop(italic(eta[1])* " + " *italic(u[3][i]), "Stroop Effect on IIV"))) +
  scale_color_manual(values = c("#009E73", "#CC79A7")) +
  xlab("")  +
  scale_y_continuous(limits = c(-1, 1.1),  
                     labels = labs) +
 xlab("Ascending Index") 

stroop_scale_stroop_plot
```

<img src="figures/README-unnamed-chunk-35-1.png" width="80%" style="display: block; margin: auto;" />

Next the "flanker effect" on IIV:

``` r
flank_scale_flank_plot <- coef(fit_flank, probs = c(0.05, 0.95))$ID[,,4] %>% 
  data.frame() %>%
  arrange(Estimate) %>%
  mutate(sig = as.factor(ifelse(Q5 < flank_scale_flank & Q95 > flank_scale_flank, 0, 1)),
         index = as.factor(1:121),
         type = "MELSM") %>%
  ggplot() +
  facet_grid(~ type ) +
  geom_errorbar(aes(x = index, 
                    ymin = Q5, 
                    ymax = Q95, 
                    color = sig), 
                show.legend = F,
                width = 0) +
  geom_hline(yintercept = flank_scale_flank,
             alpha = 0.50,
             linetype = "twodash") +
  geom_hline(yintercept = 0,
             alpha = 0.50,
             linetype = "dotted") +

  geom_point(aes(x = index, 
                 y = Estimate,
                 group = sig),
             size = 2, 
             alpha = 0.75) +
  theme_bw(base_family = "Times") +
  # plot options
  theme(panel.grid.major.x =   element_blank(), 
        panel.grid.minor.y = element_blank(),
        axis.text.x = element_blank(),
        axis.title = element_text(size = 14),
        title = element_text(size = 14),
        strip.background = element_blank(),
        strip.text = element_blank()) +
  ylab(expression(atop(italic(eta[1])* " + " *italic(u[3][i]), "Flanker Effect on IIV"))) +
  scale_color_manual(values = c("#009E73", "#CC79A7")) +
  xlab("") +
  scale_y_continuous(limits = c(-1, 1.1), 
                     labels = labs) +
  xlab("Ascending Index") 

flank_scale_flank_plot
```

<img src="figures/README-unnamed-chunk-36-1.png" width="80%" style="display: block; margin: auto;" />

The above is panel A in the paper:

``` r
# combine intercept plots
int_scale_plot <- plot_grid(int_scale_stroop_plot, int_scale_flank_plot)

# combine slope plots
cong_scale_plot <- plot_grid(stroop_scale_stroop_plot, flank_scale_flank_plot)

# panel A
scale_plot <- plot_grid(int_scale_plot, cong_scale_plot, nrow = 2, rel_heights = c(1, 1))

scale_plot
```

<img src="figures/README-unnamed-chunk-37-1.png" width="80%" style="display: block; margin: auto;" />

Panel B follows:

``` r
stroop_res <- coef(fit_stroop)$ID[,1,3:4] %>% 
  data.frame() %>% 
  mutate(cong = exp(sigma_Intercept), 
         incong = exp(sigma_Intercept + sigma_congruencyincongruent)) %>% 
  mutate(diff =  incong - cong) %>% 
  arrange(diff)

# select from tails of distribution
stroop_res_sub <-stroop_res[c(1:10, 112:121),]

dat_stroop_plot <- data.frame(iiv = c(stroop_res_sub$cong, stroop_res_sub$incong ), 
                             Condition = rep(c("Congruent", "Incongruent"), each = 20),
                             ID = as.factor(rep(1:20, times = 2)), 
                             data = "Stroop")



stroop_iiv_plot <- dat_stroop_plot %>% 
  
  ggplot(aes(x = Condition, y = iiv)) +
  geom_line(aes(group = ID, x = Condition), 
            alpha = 0.25, size = 1.25, 
            position = position_dodge(0.05)) +
  geom_point(aes(color = ID), 
             show.legend = F, 
             size = 6, 
             position = position_dodge(.05)) +
  ylab("Within-Person Variability  (SD)") +
  facet_grid(~ data) +
  theme_bw(base_family = "Times") +
  # plot options
  theme(panel.grid.major.x = element_blank(), 
        panel.grid.minor.y = element_blank(),
        axis.text.x = element_text(size =14),
        axis.title = element_text(size = 14),
        title = element_text(size = 14),
        strip.background = element_rect(fill = "grey94"),
        strip.text = element_text(size = 14)) +
  xlab("") +
  scale_color_grey()

stroop_iiv_plot
```

<img src="figures/README-unnamed-chunk-38-1.png" width="80%" style="display: block; margin: auto;" />

Now the Flaner IIV plot:

``` r
flank_res <- coef(fit_flank)$ID[,1,3:4] %>% 
  data.frame() %>% 
  mutate(cong = exp(sigma_Intercept), incong = exp(sigma_Intercept +sigma_congruencyincongruent )) %>% 
  mutate(diff =  incong - cong) %>% 
  arrange(diff)
  
flank_res_sub <- flank_res[c(1:10, 112:121),]

dat_flank_plot <- data.frame(iiv = c(flank_res_sub$cong, flank_res_sub$incong), 
                              Condition = rep(c("Congruenct", "Incongruent"), 
                                              each = 20),
                              ID = as.factor(rep(1:20, times = 2)), 
                              data = "Flanker")

flank_iiv_plot <- 
  dat_flank_plot %>% ggplot(aes(x = Condition, 
                                y = iiv)) +
  geom_line(aes(group = ID, 
                x = Condition), 
            alpha = 0.25, 
            size = 1.25, 
            position = position_dodge(.05)) +
  geom_point(aes(color = ID), 
             show.legend = F, 
             size = 6, 
             position = position_dodge(.05)) +
  ylab("Within-Person Variability  (SD)") +
  facet_grid(~ data) +
  theme_bw(base_family = "Times") +
  # plot options
  theme(panel.grid.major.x = element_blank(), 
        panel.grid.minor.y = element_blank(),
        axis.text.x = element_text(size =14),
        axis.title = element_text(size = 14),
        title = element_text(size = 14),
        strip.background = element_rect(fill = "grey94"),
        strip.text = element_text(size = 14)) +
  xlab("") +
  scale_color_grey() 

flank_iiv_plot
```

<img src="figures/README-unnamed-chunk-39-1.png" width="80%" style="display: block; margin: auto;" />

Next combine them:

``` r
iiv_plot <- plot_grid("", stroop_iiv_plot, 
                      flank_iiv_plot, "", 
                      rel_widths = c(1.5,8, 8, 1), 
                      nrow = 1)
iiv_plot
```

<img src="figures/README-unnamed-chunk-40-1.png" width="80%" style="display: block; margin: auto;" />

Figure 5:

``` r
mu_con_stroop <- fit_stroop %>%
  coef() %>%
  .$ID %>%
  .[,,"Intercept"] %>%
  data.frame() %>%
  select(Estimate)

sigma_con_stroop <- fit_stroop %>%
  coef() %>%
  .$ID %>%
  .[,,"sigma_Intercept"] %>%
  data.frame() %>%
  select(Estimate)


dat_31 <- data.frame(mu_con_stroop = mu_con_stroop[,1],
                     sigma_con_stroop = sigma_con_stroop[,1])

dat_31$facet <- "Stroop"

plot_31 <- dat_31 %>%
  ggplot(aes(y = sigma_con_stroop,
             x = mu_con_stroop)) +
  facet_grid( ~ facet) +
  theme_bw(base_family = "Times") +
  theme(panel.grid = element_blank(),
        axis.title = element_text(size = 14),
        title = element_text(size = 14),
        strip.background = element_rect(fill = "grey94"),
        strip.text = element_text(size= 14)) +
  stat_density_2d(aes(fill = ..density..),
                  geom = "raster",
                  contour = FALSE,
                  alpha = .75,
                  show.legend = F) +
  scale_fill_distiller(palette= "Spectral",
                       direction=1) +
  geom_smooth(method = "lm",
              color = "white",
              se = FALSE) +
  geom_point(aes(y = sigma_con_stroop,
                 x = mu_con_stroop),
             size = 2,
             color = "black",
             alpha = 0.5) +
  scale_x_continuous(expand = c(0, 0)) +
  scale_y_continuous(expand = c(0,0)) +
  xlab(expression(atop(italic(beta[0])* " + " *italic(u[0][i]), "Congruent RT"))) +
  ylab(expression(atop(italic(eta[0])* " + " *italic(u[2][i]), "Congruent IIV" )))


sigma_effect_stroop <- fit_stroop %>%
  coef() %>%
  .$ID %>%
  .[,,"sigma_congruencyincongruent"] %>%
  data.frame() %>%
  select(Estimate)


dat_41 <- data.frame(mu_con_stroop = mu_con_stroop[,1],
                     sigma_effect_stroop = sigma_effect_stroop[,1])

plot_41 <-
  dat_41 %>%
  ggplot(aes(y = sigma_effect_stroop,
             x = mu_con_stroop)) +
  theme_bw(base_family = "Times") +
  theme(panel.grid = element_blank(),
        axis.title = element_text(size = 14),
        title = element_text(size = 14)) +
  stat_density_2d(aes(fill = ..density..),
                  geom = "raster",
                  contour = FALSE,
                  alpha = .75,
                  show.legend = F) +
  # spectral gradient
  scale_fill_distiller(palette = "Spectral",
                       direction = 1) +
  # fitted line
  geom_smooth(method = "lm",
              color = "white",
              # no standard error
              se = FALSE) +
  # add points
  geom_point(aes(y = sigma_effect_stroop,
                 x = mu_con_stroop),
             size = 2,
             color = "black",
             alpha = 0.5) +
  # more all space in plot
  scale_x_continuous(expand = c(0, 0)) +
  scale_y_continuous(expand = c(0,0)) +
  # y label
  xlab(expression(atop(italic(beta[0])* " + " *italic(u[0][i]), "Congruent RT"))) +
  # x label
  ylab(expression(atop(italic(eta[1])* " + " *italic(u[3][i]),  "Stroop Effect on IIV"))) 


mu_effect_stroop <- fit_stroop %>%
  coef() %>%
  .$ID %>%
  .[,,"congruencyincongruent"] %>%
  data.frame() %>%
  select(Estimate)


dat_32 <- data.frame(mu_effect_stroop = mu_effect_stroop[,1],
                     sigma_con_stroop = sigma_con_stroop[,1])

dat_32$facet <- "Stroop"
plot_32 <- dat_32 %>%
  ggplot(aes(x = mu_effect_stroop,
             y = sigma_con_stroop)) +
  facet_grid(~ facet ) +
  theme_bw(base_family = "Times") +
  theme(panel.grid = element_blank(),
        axis.title = element_text(size = 14),
        title = element_text(size = 14),
        strip.background = element_rect(fill = "grey94"),
        strip.text = element_text(size= 14)) +
  stat_density_2d(aes(fill = ..density..),
                  geom = "raster",
                  contour = FALSE,
                  alpha = .75,
                  show.legend = F) +
  scale_fill_distiller(palette= "Spectral",
                       direction=1) +
    geom_smooth(method = "lm",
                color = "white",
                se = FALSE) +
    geom_point(aes(x = mu_effect_stroop,
                 y = sigma_con_stroop),
               size = 2,
               color = "black",
               alpha = 0.5) +
  scale_x_continuous(expand = c(0, 0)) +
  scale_y_continuous(expand = c(0,0)) +
  xlab(expression(atop(italic(beta[1])* " + " *italic(u[1][i]),  "Stroop Effect on RT" ))) +
  ylab(expression(atop(italic(eta[0])* " + " *italic(u[2][i]), "Congruent IIV"))) 



dat_42 <- data.frame(mu_effect_stroop = mu_effect_stroop[,1],
                     sigma_effect_stroop = sigma_effect_stroop[,1])

plot_42 <- dat_42 %>%
  ggplot(aes(y = mu_effect_stroop,
             x =  sigma_effect_stroop )) +
  theme_bw(base_family = "Times") +
  theme(panel.grid = element_blank(),
        axis.title = element_text(size = 14),
        title = element_text(size = 14)) +
  stat_density_2d(aes(fill = ..density..),
                  geom = "raster",
                  contour = FALSE,
                  alpha = .75,
                  show.legend = F) +
  scale_fill_distiller(palette= "Spectral",
                       direction=1) +
  geom_smooth(method = "lm",
              color = "white",
              se = FALSE) +
  geom_point(aes(y = mu_effect_stroop,
                 x = sigma_effect_stroop),
             size = 2,
             color = "black",
             alpha = 0.5) +
  scale_x_continuous(expand = c(0, 0)) +
  scale_y_continuous(expand = c(0,0)) +
  xlab(expression(atop(italic(beta[1])* " + " *italic(u[1][i]), "Stroop Effect on RT"))) +
  ylab(expression(atop(italic(eta[1])* " + " *italic(u[3][i]), "Stroop Effect on IIV"))) 


fig_5 <- cowplot::plot_grid( cowplot::plot_grid(plot_31, plot_41, nrow = 2) , 
                    cowplot::plot_grid(plot_32, plot_42, nrow = 2), nrow = 1)

fig_5
```

<img src="figures/README-unnamed-chunk-41-1.png" width="80%" style="display: block; margin: auto;" />

Figure 6:

``` r
fit_flanker <- fit_flank
mu_con_flanker <- fit_flanker %>%
  coef() %>%
  .$ID %>%
  .[,,"Intercept"] %>%
  data.frame() %>%
  select(Estimate)

sigma_con_flanker <- fit_flanker %>%
  coef() %>%
  .$ID %>%
  .[,,"sigma_Intercept"] %>%
  data.frame() %>%
  select(Estimate)


dat_31 <- data.frame(mu_con_flanker = mu_con_flanker[,1],
                     sigma_con_flanker = sigma_con_flanker[,1])
dat_31$facet <- "Flanker"
plot_31 <- dat_31 %>%
  ggplot(aes(y = sigma_con_flanker,
             x = mu_con_flanker)) +
  facet_grid(~ facet ) +
  theme_bw(base_family = "Times") +
  theme(panel.grid = element_blank(),
        axis.title = element_text(size = 14),
        title = element_text(size = 14),
        strip.background = element_rect(fill = "grey94"),
        strip.text = element_text(size= 14)) +
  stat_density_2d(aes(fill = ..density..),
                  geom = "raster",
                  contour = FALSE,
                  alpha = .75,
                  show.legend = F) +
  scale_fill_distiller(palette= "Spectral",
                       direction=1) +
  geom_smooth(method = "lm",
              color = "white",
              se = FALSE) +
  geom_point(aes(y = sigma_con_flanker,
                 x = mu_con_flanker),
             size = 2,
             color = "black",
             alpha = 0.5) +
  scale_x_continuous(expand = c(0, 0)) +
  scale_y_continuous(expand = c(-0.05,0)) +
  xlab(expression(atop(italic(beta[0])* " + " *italic(u[0][i]), "Congruent RT"))) +
  ylab(expression(atop(italic(eta[0])* " + " *italic(u[2][i]), "Congruent IIV" )))
  



sigma_effect_flanker <- fit_flanker %>%
  coef() %>%
  .$ID %>%
  .[,,"sigma_congruencyincongruent"] %>%
  data.frame() %>%
  select(Estimate)


dat_41 <- data.frame(mu_con_flanker = mu_con_flanker[,1],
                     sigma_effect_flanker = sigma_effect_flanker[,1])

plot_41 <-
  dat_41 %>%
  ggplot(aes(y = sigma_effect_flanker,
             x = mu_con_flanker)) +
  theme_bw(base_family = "Times") +
  theme(panel.grid = element_blank(),
        axis.title = element_text(size = 14),
        title = element_text(size = 14)) +
  stat_density_2d(aes(fill = ..density..),
                  geom = "raster",
                  contour = FALSE,
                  alpha = .75,
                  show.legend = F) +
  # spectral gradient
  scale_fill_distiller(palette = "Spectral",
                       direction = 1) +
  # fitted line
  geom_smooth(method = "lm",
              color = "white",
              # no standard error
              se = FALSE) +
  # add points
  geom_point(aes(y = sigma_effect_flanker,
                 x = mu_con_flanker),
             size = 2,
             color = "black",
             alpha = 0.5) +
  # more all space in plot
  scale_x_continuous(expand = c(0, 0)) +
  scale_y_continuous(expand = c(0,0)) +
  # y label
  xlab(expression(atop(italic(beta[0])* " + " *italic(u[0][i]), "Congruent RT"))) +
  # x label
  ylab(expression(atop(italic(eta[1])* " + " *italic(u[3][i]),  "Flanker Effect on IIV"))) 
  
 
mu_effect_flanker <- fit_flanker %>%
  coef() %>%
  .$ID %>%
  .[,,"congruencyincongruent"] %>%
  data.frame() %>%
  select(Estimate)


dat_32 <- data.frame(mu_effect_flanker = mu_effect_flanker[,1],
                     sigma_con_flanker = sigma_con_flanker[,1])
dat_32$facet <- "Flanker"
plot_32 <- dat_32 %>%
  ggplot(aes(x = mu_effect_flanker,
             y = sigma_con_flanker)) +
  facet_grid(~ facet) +
  theme_bw(base_family = "Times") +
  theme(panel.grid = element_blank(),
        axis.title = element_text(size = 14),
        title = element_text(size = 14),
        strip.background = element_rect(fill = "grey94"),
        strip.text = element_text(size= 14)) +
  stat_density_2d(aes(fill = ..density..),
                  geom = "raster",
                  contour = FALSE,
                  alpha = .75,
                  show.legend = F) +
  scale_fill_distiller(palette= "Spectral",
                       direction=1) +
  geom_smooth(method = "lm",
              color = "white",
              se = FALSE) +
  geom_point(aes(x = mu_effect_flanker,
                 y = sigma_con_flanker),
             size = 2,
             color = "black",
             alpha = 0.5) +
  scale_x_continuous(expand = c(0, 0)) +
  scale_y_continuous(expand = c(0,0)) +
  xlab(expression(atop(italic(beta[1])* " + " *italic(u[1][i]),  "Flanker Effect on RT" ))) +
  ylab(expression(atop(italic(eta[0])* " + " *italic(u[2][i]), "Congruent IIV"))) 



dat_42 <- data.frame(mu_effect_flanker = mu_effect_flanker[,1],
                     sigma_effect_flanker = sigma_effect_flanker[,1])


plot_42 <- dat_42 %>%
  ggplot(aes(y = mu_effect_flanker,
             x =  sigma_effect_flanker )) +
  theme_bw(base_family = "Times") +
  theme(panel.grid = element_blank(),
        axis.title = element_text(size = 14),
        title = element_text(size = 14)) +
  stat_density_2d(aes(fill = ..density..),
                  geom = "raster",
                  contour = FALSE,
                  alpha = .75,
                  show.legend = F) +
  scale_fill_distiller(palette= "Spectral",
                       direction=1) +
  geom_smooth(method = "lm",
              color = "white",
              se = FALSE) +
  geom_point(aes(y = mu_effect_flanker,
                 x = sigma_effect_flanker),
             size = 2,
             color = "black",
             alpha = 0.5) +
  scale_x_continuous(expand = c(0, 0)) +
  scale_y_continuous(expand = c(-0.02,0) ) + 
  xlab(expression(atop(italic(beta[1])* " + " *italic(u[1][i]), "Flanker Effect on RT"))) +
  ylab(expression(atop(italic(eta[1])* " + " *italic(u[3][i]), "Flanker Effect on IIV"))) 


Fig_6 <- cowplot::plot_grid(cowplot::plot_grid(plot_31, plot_41, nrow = 2), 
                   cowplot::plot_grid(plot_32, plot_42, nrow = 2), nrow = 1)

Fig_6
```

<img src="figures/README-unnamed-chunk-42-1.png" width="80%" style="display: block; margin: auto;" />

We then performed a sensitivty analysis for *ν* in the LKJ prior distrbution. The following is the hypothesis testing function:

``` r
# function to compute posterior probabilities
brms_cortest <- function(fit, dimension, eta){
  
 post_samps <- fit %>% 
               posterior_samples(pars = "cor") 
  

 prior_samps <- rlkjcorr(n = 10000, K = dimension, eta = eta)[,,1][,2]
  post <- apply(post_samps, 2, atanh) 
  prior <- atanh(prior_samps)
  col_names <- substring(colnames(post), 9)
  post_dense_0  <- unlist( lapply(1:ncol(post), function(x) dnorm(0, mean(post[, x]), sd(post[,x]))) )
  dense_greater  <- unlist( lapply(1:ncol(post), function(x) (1 - pnorm(0, mean(post[,x]), sd(post[,x]))) * 2))
  dense_lesser  <- unlist( lapply(1:ncol(post), function(x) (pnorm(0, mean(post[,x]), sd(post[,x]))) * 2))
  
  prior_dense_0 <-  dnorm(0, mean(prior), sd(prior))
  BF_01 <- post_dense_0 / prior_dense_0
  BF_1u <- dense_greater * (1/ BF_01)
  BF_2u <- dense_lesser * (1/ BF_01)
  
  BF_helper <- function (BF_null, BF_positive, BF_negative) {
    c(BF_null, BF_positive, BF_negative)/sum(BF_null, BF_positive, 
                                             BF_negative)
  }
  
  
  
  post_prob <- data.frame(round(t(mapply(BF_helper, BF_01, BF_1u, BF_2u)), 3))
  colnames(post_prob) <-c("null_prob", "pos_prob", "neg_prob")
  dat <- cbind(data.frame(parameter = col_names, 
                    post_mean = as.numeric(colMeans(post_samps)),  
                    post_sd = as.numeric(apply(post_samps, 2, sd)),
                    BF_01 = log(BF_01), 
                    BF_10 = log(1/ BF_01)),
               post_prob)
  dat
}
```

Figure 7:

``` r
nu_values <- seq(0.1, 4, length.out = 20)

stroop_sens <- lapply(1:length(nu_values), function(x)  cbind(brms_cortest(fit = fit_stroop, 
                                                                     dimension = 4, 
                                                                     eta = nu_values[x])[c(2:5),c(1,5)], 
                                                              data.frame(nu = (nu_values[x]))))

flank_sens <- lapply(1:length(nu_values), function(x)  cbind(brms_cortest(fit = fit_flank, 
                                                                           dimension = 4, 
                                                                           eta = nu_values[x])[c(2:5),c(1,5)], 
                                                              data.frame(nu = (nu_values[x]))))


names(stroop_sens) <- names(flank_sens) <- nu_values


dat_stroop_sens <- do.call(rbind.data.frame, stroop_sens)
dat_stroop_sens$outcome <- "Stroop"

dat_flank_sens <- do.call(rbind.data.frame, flank_sens)
dat_flank_sens$outcome <- "Flanker"

# legend 
sens_legend <- dat_stroop_sens %>% 
  ggplot() + 
  facet_wrap(~ outcome) +
  geom_hline(yintercept = log(.33), 
             linetype = "dotted") +
  geom_hline(yintercept = log(3), 
             linetype = "dotted") +
  annotate("rect", 
           xmin=-Inf, 
           xmax=Inf, ymin=log(3), ymax=Inf, alpha=0.30, fill="grey90") +
  annotate("rect", 
           xmin=-Inf, 
           xmax=Inf, ymin=-Inf, ymax=log(.33), alpha=0.30, fill="grey90") +
  geom_line(aes( x = nu, color = parameter, y= BF_10), size = 2) +
  scale_color_manual(name = "Relation",
                     breaks = c("Intercept__sigma_Intercept",
                                "Intercept__sigma_congruencyincongruent",
                                "congruencyincongruent__sigma_Intercept",
                               "congruencyincongruent__sigma_congruencyincongruent"),
                     labels = c(expression(italic("cor")*"("*italic( u[0][i]* ","*u[2][i])*")"),
                                expression(italic("cor")*"("*italic( u[0][i]* ","*u[3][i])*")"),
                                expression(italic("cor")*"("*italic( u[1][i]* ","*u[2][i])*")"),
                                expression(italic("cor")*"("*italic( u[1][i]* ","*u[3][i])*")")
                                ),
                     
                     values = c("#999999", "#E69F00", "#56B4E9", "#009E73")) +
  theme_bw(base_family = "Times") +
  theme(panel.grid.minor.x  = element_blank(),
        axis.title = element_text(size = 14),
        title = element_text(size = 14),
        legend.text = element_text(size = 14),
        legend.position = "top",
        strip.background = element_rect(fill = "grey94"),
        strip.text = element_text(size= 14)) +
  xlab(expression("LKJ  " * italic(nu))) +
  ylab(expression("log("*italic(BF[10])*")"))

sens_legend <- get_legend(sens_legend) 

# stroop
sens_stroop <- dat_stroop_sens %>% 
  ggplot() + 
  facet_wrap(~ outcome) +
  geom_hline(yintercept = log(.33), 
             linetype = "dotted") +
  geom_hline(yintercept = log(3), 
             linetype = "dotted") +
  annotate("rect", 
           xmin=-Inf, 
           xmax=Inf, ymin=log(3), ymax=Inf, alpha=0.30, fill="grey90") +
  annotate("rect", 
           xmin=-Inf, 
           xmax=Inf, ymin=-Inf, ymax=log(.33), alpha=0.30, fill="grey90") +
  geom_line(aes( x = nu, color = parameter, y= BF_10), size = 2) +
  scale_color_manual(name = "Relation",
                     breaks = c("Intercept__sigma_Intercept",
                                "Intercept__sigma_congruencyincongruent",
                                "congruencyincongruent__sigma_Intercept",
                                "congruencyincongruent__sigma_congruencyincongruent"),
                     labels = c(expression(italic("cor")*"("*italic( u[0][i]* ","*u[2][i])*")"),
                                expression(italic("cor")*"("*italic( u[0][i]* ","*u[3][i])*")"),
                                expression(italic("cor")*"("*italic( u[1][i]* ","*u[2][i])*")"),
                                expression(italic("cor")*"("*italic( u[1][i]* ","*u[3][i])*")")),
                     values = c("#999999", "#E69F00", "#56B4E9", "#009E73")) +
  theme_bw(base_family = "Times") +
  theme(panel.grid.minor.x  = element_blank(),
        axis.title = element_text(size = 14),
        title = element_text(size = 14),
        legend.text = element_text(size = 14),
        legend.position = "none",
        strip.background = element_rect(fill = "grey94"),
        strip.text = element_text(size= 14)) +
  xlab(expression("LKJ  " * italic(nu))) +
  ylab(expression("log("*italic(BF[10])*")"))


# flanker
sens_flank <- dat_flank_sens %>% 
  ggplot() + 
  facet_wrap(~ outcome) +
  geom_hline(yintercept = log(.33), 
             linetype = "dotted") +
  geom_hline(yintercept = log(3), 
             linetype = "dotted") +
  annotate("rect", 
           xmin=-Inf, 
           xmax=Inf, ymin=log(3), ymax=Inf, alpha=0.30, fill="grey90") +
  annotate("rect", 
           xmin=-Inf, 
           xmax=Inf, ymin=-Inf, ymax=log(.33), alpha=0.30, fill="grey90") +
  geom_line(aes( x = nu, color = parameter, y= BF_10), size = 2) +
  scale_color_manual(name = "Relation",
                     breaks = c("Intercept__sigma_Intercept",
                                "Intercept__sigma_congruencyincongruent",
                                "congruencyincongruent__sigma_Intercept",
                                "congruencyincongruent__sigma_congruencyincongruent"),
                     labels = c(expression(italic("cor")*"("*italic( u[0][i]* ","*u[2][i])*")"),
                                expression(italic("cor")*"("*italic( u[0][i]* ","*u[3][i])*")"),
                                expression(italic("cor")*"("*italic( u[1][i]* ","*u[2][i])*")"),
                                expression(italic("cor")*"("*italic( u[1][i]* ","*u[3][i])*")")),
                     values = c("#999999", "#E69F00", "#56B4E9", "#009E73")) +
  theme_bw(base_family = "Times") +
  theme(panel.grid.minor.x  = element_blank(),
        axis.title = element_text(size = 14),
        title = element_text(size = 14),
        legend.text = element_text(size = 14),
        legend.position = "none",
        strip.background = element_rect(fill = "grey94"),
        strip.text = element_text(size= 14)) +
  xlab(expression("LKJ  " * italic(nu))) +
  ylab(expression("log("*italic(BF[10])*")"))

plot_7 <- plot_grid(sens_legend, 
                    plot_grid(sens_stroop, 
                              sens_flank, 
                              nrow = 1),  
                    nrow = 2, 
                    rel_heights = c(1,10))
plot_7
```

<img src="figures/README-unnamed-chunk-44-1.png" width="80%" style="display: block; margin: auto;" />

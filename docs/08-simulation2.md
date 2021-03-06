# Simulation 2 

In which we figure out some key statistical concepts through simulation and plotting. On the menu we have: 

- Central limit theorem 
- Sampling distributions 
- p-value
- Confidence interval

## Load packages and set plotting theme  




```r
library("knitr")      # for knitting RMarkdown 
library("NHANES")     # data set 
library("kableExtra") # for making nice tables
library("janitor")    # for cleaning column names
library("tidyverse")  # for wrangling, plotting, etc. 

opts_chunk$set(
  comment = "",
  results = "hold",
  fig.show = "hold"
)
```


```r
theme_set(
  theme_classic() + #set the theme 
    theme(text = element_text(size = 20)) #set the default text size
)
```

## The central limit theorem 

> The Central Limit Theorem (CLT) states that the sample mean of a sufficiently large number of independent and identically distributed (i.i.d.) random variables is approximately normally distributed. The larger the sample, the better the approximation.  The theorem is a key ("central") concept in probability theory because it implies that statistical methods that work for normal distributions can be applicable to many problems involving other types of distributions.

Here are some nice interactive illustrations of the CLT: 

- [seeing-theory.brown.edu](https://seeing-theory.brown.edu/probability-distributions/index.html#section3)
- [http://mfviz.com/central-limit/](http://mfviz.com/central-limit/)

### Population distribution 

Let's first put the information we need for our population distribution in a data frame. 


```r
# the distribution from which we want to sample (aka the heavy metal distribution)
df.population = tibble(
  numbers = 1:6,
  probability = c(1/3, 0, 1/6, 1/6, 0, 1/3)
)
```

And then let's plot it: 


```r
# plot the distribution 
ggplot(data = df.population,
       mapping = aes(x = numbers,
                     y = probability)) +
  geom_bar(stat = "identity",
           fill = "lightblue",
           color = "black") +
  scale_x_continuous(breaks = df.population$numbers,
                     labels = df.population$numbers,
                     limits = c(0.1, 6.9)) +
  coord_cartesian(expand = F)
```

<img src="08-simulation2_files/figure-html/simulation2-05-1.png" width="672" />

Here are the true mean and standard deviation of our population distribution: 


```r
# mean and standard deviation (see: https://nzmaths.co.nz/category/glossary/standard-deviation-discrete-random-variable)

df.population %>% 
  summarize(population_mean = sum(numbers * probability),
            population_sd = sqrt(sum(numbers^2 * probability) - population_mean^2)) %>% 
  kable(digits = 2) %>% 
  kable_styling(bootstrap_options = "striped",
                full_width = F)
```

<table class="table table-striped" style="width: auto !important; margin-left: auto; margin-right: auto;">
 <thead>
  <tr>
   <th style="text-align:right;"> population_mean </th>
   <th style="text-align:right;"> population_sd </th>
  </tr>
 </thead>
<tbody>
  <tr>
   <td style="text-align:right;"> 3.5 </td>
   <td style="text-align:right;"> 2.06 </td>
  </tr>
</tbody>
</table>

### Distribution of a single sample 

Let's draw a single sample of size $n = 40$ from the population distribution and plot it: 


```r
# make example reproducible 
set.seed(1)

# set the sample size
sample_size = 40 

# create data frame 
df.sample = sample(df.population$numbers, 
         size = sample_size, 
         replace = T,
         prob = df.population$probability) %>% 
  enframe(name = "draw", value = "number")

# draw a plot of the sample
ggplot(data = df.sample,
       mapping = aes(x = number, y = stat(density))) + 
  geom_histogram(binwidth = 0.5, 
                 fill = "lightblue",
                 color = "black") +
  scale_x_continuous(breaks = min(df.sample$number):max(df.sample$number)) + 
  scale_y_continuous(expand = expand_scale(mult = c(0, 0.01)))
```

<img src="08-simulation2_files/figure-html/simulation2-07-1.png" width="672" />

Here are the sample mean and standard deviation:


```r
# print out sample mean and standard deviation 
df.sample %>% 
  summarize(sample_mean = mean(number),
            sample_sd = sd(number)) %>% 
  kable(digits = 2) %>% 
  kable_styling(bootstrap_options = "striped",
                full_width = F)
```

<table class="table table-striped" style="width: auto !important; margin-left: auto; margin-right: auto;">
 <thead>
  <tr>
   <th style="text-align:right;"> sample_mean </th>
   <th style="text-align:right;"> sample_sd </th>
  </tr>
 </thead>
<tbody>
  <tr>
   <td style="text-align:right;"> 3.73 </td>
   <td style="text-align:right;"> 2.05 </td>
  </tr>
</tbody>
</table>

### The sampling distribution

And let's now create the sampling distribution (making the unrealistic assumption that we know the population distribution). 


```r
# make example reproducible 
set.seed(1)

# parameters 
sample_size = 40 # size of each sample
sample_n = 10000 # number of samples 

# define a function that draws samples from a discrete distribution
fun.draw_sample = function(sample_size, distribution){
  x = sample(distribution$numbers,
       size = sample_size,
       replace = T,
       prob = distribution$probability)
  return(x)
}

# generate many samples 
samples = replicate(n = sample_n,
                    fun.draw_sample(sample_size, df.population))

# set up a data frame with samples 
df.sampling_distribution = matrix(samples, ncol = sample_n) %>%
  as_tibble() %>%
  set_names(str_c(1:ncol(.))) %>%
  gather("sample", "number") %>% 
  mutate(sample = as.numeric(sample)) %>% 
  group_by(sample) %>% 
  mutate(draw = 1:n()) %>% 
  select(sample, draw, number) %>% 
  ungroup()
```

```
Warning: `as_tibble.matrix()` requires a matrix with column names or a `.name_repair` argument. Using compatibility `.name_repair`.
This warning is displayed once per session.
```

```r
# turn the data frame into long format and calculate the means of each sample
df.sampling_distribution_means = df.sampling_distribution %>% 
  group_by(sample) %>% 
  summarize(mean = mean(number)) %>% 
  ungroup()
```

And plot it: 


```r
# plot a histogram of the means with density overlaid 
ggplot(data = df.sampling_distribution_means %>% 
         sample_frac(size = 1, replace = T),
       mapping = aes(x = mean)) + 
  geom_histogram(aes(y = stat(density)),
                 binwidth = 0.05, 
                 fill = "lightblue",
                 color = "black") +
  stat_density(bw = 0.1,
               size = 2,
               geom = "line"
               ) + 
  scale_y_continuous(expand = expand_scale(mult = c(0, 0.01)))
```

<img src="08-simulation2_files/figure-html/simulation2-10-1.png" width="672" />

That's the central limit theorem in action! Even though our population distribution was far from normal (and much more heavy-metal like), the means of that distribution are normally distributed. 

And here are the mean and standard deviation of the sampling distribution: 


```r
# print out sampling distribution mean and standard deviation 
df.sampling_distribution_means %>% 
  summarize(sampling_distribution_mean = mean(mean),
            sampling_distribution_sd = sd(mean)) %>% 
  kable(digits = 2) %>% 
  kable_styling(bootstrap_options = "striped",
                full_width = F)
```

<table class="table table-striped" style="width: auto !important; margin-left: auto; margin-right: auto;">
 <thead>
  <tr>
   <th style="text-align:right;"> sampling_distribution_mean </th>
   <th style="text-align:right;"> sampling_distribution_sd </th>
  </tr>
 </thead>
<tbody>
  <tr>
   <td style="text-align:right;"> 3.5 </td>
   <td style="text-align:right;"> 0.33 </td>
  </tr>
</tbody>
</table>

Here is a data frame that I've used for illustrating the idea behind how a sampling distribution is constructed from the population distribution. 


```r
# data frame for illustration in class 
df.sampling_distribution %>% 
  filter(sample <= 10, draw <= 4) %>% 
  spread(draw, number) %>% 
  set_names(c("sample", str_c("draw_", 1:(ncol(.)-1)))) %>% 
  mutate(sample_mean = rowMeans(.[, -1])) %>% 
    head(10) %>% 
    kable(digits = 2) %>% 
    kable_styling(bootstrap_options = "striped",
                full_width = F)
```

<table class="table table-striped" style="width: auto !important; margin-left: auto; margin-right: auto;">
 <thead>
  <tr>
   <th style="text-align:right;"> sample </th>
   <th style="text-align:right;"> draw_1 </th>
   <th style="text-align:right;"> draw_2 </th>
   <th style="text-align:right;"> draw_3 </th>
   <th style="text-align:right;"> draw_4 </th>
   <th style="text-align:right;"> sample_mean </th>
  </tr>
 </thead>
<tbody>
  <tr>
   <td style="text-align:right;"> 1 </td>
   <td style="text-align:right;"> 1 </td>
   <td style="text-align:right;"> 6 </td>
   <td style="text-align:right;"> 6 </td>
   <td style="text-align:right;"> 4 </td>
   <td style="text-align:right;"> 4.25 </td>
  </tr>
  <tr>
   <td style="text-align:right;"> 2 </td>
   <td style="text-align:right;"> 3 </td>
   <td style="text-align:right;"> 6 </td>
   <td style="text-align:right;"> 3 </td>
   <td style="text-align:right;"> 6 </td>
   <td style="text-align:right;"> 4.50 </td>
  </tr>
  <tr>
   <td style="text-align:right;"> 3 </td>
   <td style="text-align:right;"> 6 </td>
   <td style="text-align:right;"> 3 </td>
   <td style="text-align:right;"> 6 </td>
   <td style="text-align:right;"> 1 </td>
   <td style="text-align:right;"> 4.00 </td>
  </tr>
  <tr>
   <td style="text-align:right;"> 4 </td>
   <td style="text-align:right;"> 4 </td>
   <td style="text-align:right;"> 6 </td>
   <td style="text-align:right;"> 6 </td>
   <td style="text-align:right;"> 1 </td>
   <td style="text-align:right;"> 4.25 </td>
  </tr>
  <tr>
   <td style="text-align:right;"> 5 </td>
   <td style="text-align:right;"> 1 </td>
   <td style="text-align:right;"> 4 </td>
   <td style="text-align:right;"> 6 </td>
   <td style="text-align:right;"> 3 </td>
   <td style="text-align:right;"> 3.50 </td>
  </tr>
  <tr>
   <td style="text-align:right;"> 6 </td>
   <td style="text-align:right;"> 1 </td>
   <td style="text-align:right;"> 1 </td>
   <td style="text-align:right;"> 6 </td>
   <td style="text-align:right;"> 1 </td>
   <td style="text-align:right;"> 2.25 </td>
  </tr>
  <tr>
   <td style="text-align:right;"> 7 </td>
   <td style="text-align:right;"> 1 </td>
   <td style="text-align:right;"> 6 </td>
   <td style="text-align:right;"> 4 </td>
   <td style="text-align:right;"> 1 </td>
   <td style="text-align:right;"> 3.00 </td>
  </tr>
  <tr>
   <td style="text-align:right;"> 8 </td>
   <td style="text-align:right;"> 1 </td>
   <td style="text-align:right;"> 6 </td>
   <td style="text-align:right;"> 4 </td>
   <td style="text-align:right;"> 6 </td>
   <td style="text-align:right;"> 4.25 </td>
  </tr>
  <tr>
   <td style="text-align:right;"> 9 </td>
   <td style="text-align:right;"> 6 </td>
   <td style="text-align:right;"> 1 </td>
   <td style="text-align:right;"> 6 </td>
   <td style="text-align:right;"> 4 </td>
   <td style="text-align:right;"> 4.25 </td>
  </tr>
  <tr>
   <td style="text-align:right;"> 10 </td>
   <td style="text-align:right;"> 1 </td>
   <td style="text-align:right;"> 1 </td>
   <td style="text-align:right;"> 4 </td>
   <td style="text-align:right;"> 1 </td>
   <td style="text-align:right;"> 1.75 </td>
  </tr>
</tbody>
</table>

#### Bootstrapping a sampling distribution

Of course, in actuality, we never have access to the population distribution. We try to infer characteristics of that distribution (e.g. its mean) from our sample. So using the population distribution to create a sampling distribution is sort of cheating -- helpful cheating though since it gives us a sense for the relationship between population, sample, and sampling distribution. 

It urns out that we can approximate the sampling distribution only using our actual sample. The idea is to take the sample that we drew, and generate new samples from it by drawing with replacement. Essentially, we are treating our original sample like the population from which we are generating random samples to derive the sampling distribution. 


```r
# make example reproducible 
set.seed(1)

# how many bootstrapped samples shall we draw? 
n_samples = 1000

# generate a new sample from the original one by sampling with replacement
func.bootstrap = function(df){
  df %>% 
    sample_frac(size = 1, replace = T) %>% 
    summarize(mean = mean(number)) %>% 
    pull(mean)
}

# data frame with bootstrapped results 
df.bootstrap = tibble(
  bootstrap = 1:n_samples, 
  average = replicate(n = n_samples, func.bootstrap(df.sample))
)
```

Let's plot the bootstrapped sampling distribution: 


```r
# plot the bootstrapped sampling distribution
ggplot(data = df.bootstrap, aes(x = average)) +
  geom_histogram(aes(y = stat(density)),
                 color = "black",
                 fill = "lightblue",
                 binwidth = 0.05) + 
  stat_density(geom = "line",
               size = 1.5,
               bw = 0.1) +
  labs(x = "mean") +
  scale_y_continuous(expand = expand_scale(mult = c(0, 0.01)))
```

<img src="08-simulation2_files/figure-html/simulation2-14-1.png" width="672" />

And let's calculate the mean and standard deviation: 


```r
# print out sampling distribution mean and standard deviation 
df.sampling_distribution_means %>% 
  summarize(bootstrapped_distribution_mean = mean(mean),
            bootstrapped_distribution_sd = sd(mean)) %>% 
  kable(digits = 2) %>% 
  kable_styling(bootstrap_options = "striped",
                full_width = F)
```

<table class="table table-striped" style="width: auto !important; margin-left: auto; margin-right: auto;">
 <thead>
  <tr>
   <th style="text-align:right;"> bootstrapped_distribution_mean </th>
   <th style="text-align:right;"> bootstrapped_distribution_sd </th>
  </tr>
 </thead>
<tbody>
  <tr>
   <td style="text-align:right;"> 3.5 </td>
   <td style="text-align:right;"> 0.33 </td>
  </tr>
</tbody>
</table>

Neat, as we can see, the mean and standard deviation of the bootstrapped sampling distribution are very close to the sampling distribution that we generated from the population distribution. 

### Exploring the CLT 

#### Distribution of height 

In order for the CLT to apply, the following have to hold (approximately): 

- sufficiently large number of variables that affect the outcome 
- the variables are idependent and identically distributed 
- the variables contribute additively, and none of the variables affects the outcome much more strongly than the rest 

Let's take a look at a situation where the CLT breaks down. We use survey data collected by the US National Center for Health Statistics (NCHS). The data is from 2009-2012. You can get more information about the NHANES data set by running `help(NHANES)`. 

Let's load the data set into our environment first.


```r
df.nhanes = NHANES %>% 
  clean_names() %>% 
  distinct(id, .keep_all = T) #drop duplicates
```

Let's now plot a density of the distribution of womens' height 


```r
df.plot = df.nhanes %>% 
  drop_na(height) %>% # remove missing values
  filter(
    age >= 18, # only look at adults 
    gender == "female"
    ) 
  
ggplot(data = df.plot, 
       mapping = aes(x = height)) +
  geom_density(size = 1,
               fill = "red",
               alpha = 0.5,
               kernel = "gaussian",
               bw = 2) +
  stat_function(fun = "dnorm",
                color = "red",
                args = list(mean = mean(df.plot$height),
                            sd = sd(df.plot$height)),
                size = 2) +
  labs(title = "Women's height") +
  coord_cartesian(expand = F, clip = "off")
```

<img src="08-simulation2_files/figure-html/simulation2-17-1.png" width="672" />

Women's height in the `NHANES` data set is approximately normally distributed. 


```r
df.plot = df.nhanes %>% 
  drop_na(height) %>% # remove missing values
  filter(
    age >= 18, # only look at adults 
    gender == "male"
    ) 
  
ggplot(data = df.plot, 
       mapping = aes(x = height)) +
  geom_density(size = 1,
               fill = "blue",
               alpha = 0.5,
               kernel = "gaussian",
               bw = 2) +
  stat_function(fun = "dnorm",
                color = "blue",
                args = list(mean = mean(df.plot$height),
                            sd = sd(df.plot$height)),
                size = 2) +
  labs(title = "Men's height") +
  coord_cartesian(expand = F, clip = "off")
```

<img src="08-simulation2_files/figure-html/simulation2-18-1.png" width="672" />

The same is true for men's height.


```r
df.plot = df.nhanes %>% 
  drop_na(height) %>% # remove missing values
  filter(
    age >= 18 # only look at adults
    ) 

ggplot(data = df.plot, 
       mapping = aes(x = height))+
  geom_density(size = 1,
               fill = "gray50",
               alpha = 0.5,
               kernel = "gaussian",
               bw = 2)+
  stat_function(fun = "dnorm",
                color = "black",
                args = list(mean = mean(df.plot$height),
                            sd = sd(df.plot$height)),
                size = 2)+
  labs(title = "Adults' height") +
  coord_cartesian(expand = F, clip = "off")
```

<img src="08-simulation2_files/figure-html/simulation2-19-1.png" width="672" />

However, adults' height is not quite normally distributed. Note that the distribution is too flat in the middle.  


```r
df.plot = df.nhanes %>% 
  drop_na(height) %>% # remove missing values
  filter(
    age >= 18
    )

ggplot(data = df.plot, aes(x = height, group = gender, fill = gender))+
  geom_density(size = 1, 
               alpha = 0.5,
               kernel = "gaussian",
               bw = 2)+
  stat_function(fun = "dnorm", color = "blue", 
                args = df.plot %>% 
                  filter(gender == "male") %>% 
                  summarise(mean = mean(height),
                            sd = sd(height)) %>% 
                  as.list(),
                size = 2)+
  stat_function(fun = "dnorm", color = "red", 
                args = df.plot %>% 
                  filter(gender == "female") %>% 
                  summarise(mean = mean(height),
                            sd = sd(height)) %>% 
                  as.list(),
                size = 2)+
  labs(title = "Adults' height (separated by gender)")+
  theme(legend.position = c(0.9, 0.8))
```

<img src="08-simulation2_files/figure-html/simulation2-20-1.png" width="672" />

The fact that adults' height overall is not normally distributed is because there is a single factor (gender) that accounts for much of the variation. 

#### Testing the limits 

How do sample size and the number of samples affect what the sampling distribution looks like? Here are some simulations. Feel free to play around with: 

- the population distributions to sample from
- the sample size for each sample 
- the number of samples


```r
ggplot(data = tibble(x = c(0, 20)), aes(x = x)) +
  stat_function(fun = "dnorm",
                args = list(mean = 10,
                            sd = 5),
                size = 1,
                color = "red") +
  stat_function(fun = "dunif",
                args = list(min = 0,
                            max = 20),
                size = 1,
                color = "green") +
  stat_function(fun = "dexp",
                args = list(rate = 0.1),
                size = 1,
                color = "blue") +
  annotate(geom = "text",
           label = "normal",
           x = 0,
           y = .03,
           hjust = 0,
           color = "red",
           size = 6) +
  annotate(geom = "text",
           label = "uniform",
           x = 0,
           y = .055,
           hjust = 0,
           color = "green",
           size = 6) +
  annotate(geom = "text",
           label = "exponential",
           x = 0,
           y = .105,
           hjust = 0,
           color = "blue",
           size = 6)
```

<img src="08-simulation2_files/figure-html/simulation2-21-1.png" width="672" />



```r
# Parameters for the simulation
n_samples = c(10, 100, 1000, 10000)
sample_size = c(5, 10, 25, 100)
distributions = c("normal", "uniform", "exponential")

# take samples (of size n) from specified distribution and calculate the mean 
fun.sample_mean = function(n, distribution){
  if (distribution == "normal"){
    tmp = rnorm(n, mean = 10, sd = 5)
  }else if (distribution == "uniform"){
    tmp = runif(n, min = 0, max = 20) 
  }else if (distribution == "exponential"){
    tmp = rexp(n, rate = 0.1)
  }
  return(mean(tmp)) 
}

df.central_limit = tibble()

for (i in 1:length(n_samples)){
  for (j in 1:length(sample_size)){
    for (k in 1:length(distributions)){
      # calculate sample mean 
      sample_mean = replicate(n_samples[i], 
                              fun.sample_mean(sample_size[j],
                                              distributions[k]))
      df.tmp = tibble(n_samples = n_samples[i], 
                       sample_size = sample_size[j],
                       distribution = distributions[k],
                       mean_value = list(sample_mean))
      df.central_limit = rbind(df.central_limit, df.tmp)
    }
  }
}

# transform from list column
df.plot = df.central_limit %>% 
  unnest() %>% 
  mutate(sample_size = str_c("sample size = ", sample_size),
         sample_size = factor(sample_size,
                              levels = str_c("sample size = ", c(5, 10, 25, 100))),
         n_samples = str_c("n samples = ", n_samples),
         distribution = factor(distribution,
                               levels = c("normal", "uniform", "exponential"))
         )
  
# densities of sample means 
ggplot(df.plot, aes(x = mean_value, color = distribution))+
  stat_density(geom = "line", position = "identity")+
  facet_grid(n_samples ~ sample_size, scales = "free")+
  scale_x_continuous(breaks = c(0, 10, 20))+
  coord_cartesian(xlim = c(0, 20))+
  labs(x = "sample mean")+
  theme(
    axis.text.x = element_text(size = 10),
    axis.text.y = element_text(size = 10),
    strip.text.y = element_text(size = 6),
    strip.text.x = element_text(size = 8),
    legend.position = "bottom",
    panel.background = element_rect(color = "black")
    )
```

<img src="08-simulation2_files/figure-html/simulation2-22-1.png" width="672" />

No matter where we start, as long as we draw samples that are independent and identically distributed, and these samples combine in an additive way, we end up with a normal distribution (note that this takes considerably longer when we start with an exponential distribution -- shown in blue -- compared to the other population distributions).

## Understanding p-values 

> The p-value is the probability of finding the observed, or more extreme, results when the null hypothesis ($H_0$) is true.

$$
\text{p-value = p(observed or more extreme sample statistic} | H_{0}=\text{true})
$$
What we are really interested in is the probability of a hypothesis given the data. However, frequentist statistics doesn't give us this probability -- we'll get to Bayesian statistics later in the course. 

Instead, we define a null hypothesis, construct a sampling distribution that tells us what we would expect the test statistic of interest to look like if the null hypothesis were true. We reject the null hypothesis in case our observed data would be unlikely if the null hypothesis were true. 

An intutive way for illustrating (this rather unintuitive procedure) is the permutation test. 

### Permutation test 

Let's start by generating some random data from two different normal distributions (simulating a possible experiment). 


```r
# make example reproducible 
set.seed(1)

# generate data from two conditions 
df.permutation = tibble(
  control = rnorm(25, mean = 5.5, sd = 2),
  experimental = rnorm(25, mean = 4.5, sd = 1.5)
) %>% 
  gather("condition", "performance")
```

Here is a summary of how each group performed: 


```r
df.permutation %>% 
  group_by(condition) %>%
  summarize(mean = mean(performance),
            sd = sd(performance)) %>%
  gather("statistic", "value", - condition) %>%
  spread(condition, value) %>%
  kable(digits = 2) %>% 
  kable_styling(bootstrap_options = "striped",
                full_width = F)
```

<table class="table table-striped" style="width: auto !important; margin-left: auto; margin-right: auto;">
 <thead>
  <tr>
   <th style="text-align:left;"> statistic </th>
   <th style="text-align:right;"> control </th>
   <th style="text-align:right;"> experimental </th>
  </tr>
 </thead>
<tbody>
  <tr>
   <td style="text-align:left;"> mean </td>
   <td style="text-align:right;"> 5.84 </td>
   <td style="text-align:right;"> 4.55 </td>
  </tr>
  <tr>
   <td style="text-align:left;"> sd </td>
   <td style="text-align:right;"> 1.90 </td>
   <td style="text-align:right;"> 1.06 </td>
  </tr>
</tbody>
</table>

Let's plot the results: 


```r
ggplot(data = df.permutation, 
       mapping = aes(x = condition, y = performance)) +
  geom_point(position = position_jitter(height = 0, width = 0.1),
             alpha = 0.5) + 
  stat_summary(fun.data = mean_cl_boot, 
               geom = "linerange", 
               size = 1) +
  stat_summary(fun.y = "mean", 
               geom = "point", 
               shape = 21, 
               color = "black", 
               fill = "white", 
               size = 4) +
  scale_y_continuous(breaks = 0:10,
                     labels = 0:10,
                     limits = c(0, 10))
```

<img src="08-simulation2_files/figure-html/simulation2-25-1.png" width="672" />

We are interested in the difference in the mean performance between the two groups: 


```r
# calculate the difference between conditions
difference_actual = df.permutation %>% 
  group_by(condition) %>% 
  summarize(mean = mean(performance)) %>% 
  pull(mean) %>% 
  diff()
```

The difference in the mean rating between the control and experimental condition is -1.2889834. Is this difference between conditions statistically significant? What we are asking is: what are the chances that a result like this (or more extreme) could have come about due to chance? 

Let's answer the question using simulation. Here is the main idea: imagine that we were very sloppy in how we recorded the data, and now we don't remember anymore which participants were in the controld condition and which ones were in experimental condition (we still remember though, that we tested 25 participants in each condition). 


```r
set.seed(0)
df.permutation = df.permutation %>% 
  mutate(permutation = sample(condition)) #randomly assign labels

df.permutation %>% 
  group_by(permutation) %>% 
  summarize(mean = mean(performance),
            sd = sd(performance)) %>% 
  ungroup() %>% 
  summarize(diff = diff(mean))
```

```
# A tibble: 1 x 1
     diff
    <dbl>
1 -0.0223
```

Here, the difference between the two conditions is 0.0223078.

After randomly shuffling the condition labels, this is how the results would look like: 


```r
ggplot(data = df.permutation, aes(x = permutation, y = performance))+
  geom_point(aes(color = condition), position = position_jitter(height = 0, width = 0.1)) +
  stat_summary(fun.data = mean_cl_boot, geom = 'linerange', size = 1) +
  stat_summary(fun.y = "mean", geom = 'point', shape = 21, color = "black", fill = "white", size = 4) + 
  scale_y_continuous(breaks = 0:10,
                     labels = 0:10,
                     limits = c(0, 10))
```

<img src="08-simulation2_files/figure-html/simulation2-28-1.png" width="672" />

The idea is now that, similar to bootstrapping above, we can get a sampling distribution of the difference in the means between the two conditions (assuming that the null hypothesis were true), by randomly shuffling the labels and calculating the difference in means (and doing this many times). What we get is a distribution of the differences we would expect, if there was no effect of condition. 


```r
set.seed(1)

n_permutations = 500

# permutation function
func_permutations = function(df){
  df %>%
    mutate(condition = sample(condition)) %>% #we randomly shuffle the condition labels
    group_by(condition) %>%
    summarize(mean = mean(performance)) %>%
    pull(mean) %>%
    diff()
}

# data frame with permutation results 
df.permutations = tibble(
  permutation = 1:n_permutations, 
  mean_difference = replicate(n = n_permutations, func_permutations(df.permutation))
)

#plot the distribution of the differences 
ggplot(data = df.permutations, aes(x = mean_difference)) +
  geom_histogram(aes(y = stat(density)),
                 color = "black",
                 fill = "lightblue",
                 binwidth = 0.05) + 
  stat_density(geom = "line",
               size = 1.5,
               bw = 0.2) +
  geom_vline(xintercept = difference_actual, color = "red", size = 2) +
  labs(x = "difference between means") +
  scale_x_continuous(breaks = seq(-1.5, 1.5, 0.5),
                     labels = seq(-1.5, 1.5, 0.5),
                     limits = c(-2, 2)) +
  coord_cartesian(expand = F, clip = "off")
```

```
Warning: Removed 2 rows containing missing values (geom_bar).
```

<img src="08-simulation2_files/figure-html/simulation2-29-1.png" width="672" />

And we can then simply calculate the p-value by using some basic data wrangling (i.e. finding the proportion of differences that were as or more extreme than the one we observed).


```r
#calculate p-value of our observed result
df.permutations %>% 
  summarize(p_value = sum(mean_difference <= difference_actual)/n())
```

```
# A tibble: 1 x 1
  p_value
    <dbl>
1   0.006
```


## Confidence intervals 

The definition of the confidence interval is the following: 

> “If we were to repeat the experiment over and over, then 95% of the time the confidence intervals contain the true mean.” 

If we assume normally distributed data (and a large enough sample size), then we can calculate the confidence interval on the estimate of the mean in the following way: $\overline X \pm Z \frac{s}{\sqrt{n}}$, where $Z$ equals the value of the standard normal distribution for the desired level of confidence. 

For smaller sample sizes, we can use the $t$-distribution instead with $n-1$ degrees of freedom. For larger $n$ the $t$-distribution closely approximates the normal distribution. 

So let's run a a simulation to check whether the definition of the confidence interval seems right. We will use our heavy metal distribution from above, take samples from the distribution, calculate the mean and confidende interval, and check how often the true mean of the population ($M = 3.5$) is contained within the confidence interval. 


```r
# make example reproducible 
set.seed(1)

# parameters 
sample_size = 25 # size of each sample
sample_n = 20 # number of samples 
confidence_level = 0.95 # desired level of confidence 

# define a function that draws samples and calculates means and CIs
fun.confidence = function(sample_size, distribution){
  df = tibble(
    values = sample(distribution$numbers,
                    size = sample_size,
                    replace = T,
                    prob = distribution$probability)) %>% 
    summarize(mean = mean(values),
              sd = sd(values),
              n = n(),
              # confidence interval assuming a normal distribution 
              # error = qnorm(1-(1-confidence_level)*2) * sd / sqrt(n),
              # assuming a t-distribution (more conservative, appropriate for smaller
              # sample sizes)
              error = qt(1-(1-confidence_level)/2, df = n-1) * sd / sqrt(n),
              conf_low = mean - error,
              conf_high = mean + error)
  return(df)
}

# build data frame of confidence intervals 
df.confidence = tibble()
for(i in 1:sample_n){
  df.tmp = fun.confidence(sample_size, df.population)
  df.confidence = df.confidence %>% 
    bind_rows(df.tmp)
}

# code which CIs contain the true value, and which ones don't 
population_mean = 3.5
df.confidence = df.confidence %>% 
  mutate(sample = 1:n(),
         conf_index = ifelse(conf_low > population_mean | conf_high < population_mean,
                             'outside',
                             'inside'))

# plot the result
ggplot(data = df.confidence, aes(x = sample, y = mean, color = conf_index))+
  geom_hline(yintercept = 3.5, color = "red")+
  geom_point()+
  geom_linerange(aes(ymin = conf_low, ymax = conf_high))+
  coord_flip()+
  scale_color_manual(values = c("black", "red"), labels = c("inside", "outside"))+
  theme(axis.text.y = element_text(size = 12),
        legend.position = "none")
```

<img src="08-simulation2_files/figure-html/simulation2-31-1.png" width="672" />

So, out of the 20 samples that we drew the 95% confidence interval of 1 sample did not contain the true mean. That makes sense! 

Feel free to play around with the code above. For example, change the sample size, the number of samples, the confidence level.  

## Additional resources 

### Datacamp 

- [Foundations of Inference](https://www.datacamp.com/courses/foundations-of-inference)

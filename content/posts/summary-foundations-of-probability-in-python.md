---
title: "Summary: Foundations of Probability in Python"
date: 2020-08-10T00:49:17-04:00
author: Ze Xuan Ong
feature_image: /summary-foundations-of-probability-python/feature.png
draft: true
katex: true
markup: "mmark"
---

### Bernoulli Trial

```
# Import the bernoulli object from scipy.stats
from scipy.stats import bernoulli

# Seed is still controlled by Numpy
np.random.seed(42)

# Simulate one coin flip with 35% chance of getting heads
coin_flip = bernoulli.rvs(p=0.35, size=1)

# Simulate 20 trials of 10 coin flips with 35% chance of getting heads
draw = binom.rvs(n=10, p=0.35, size=20)

```

### PMF

```
binom.cdf(k=5, n=10, p=0.5)

```
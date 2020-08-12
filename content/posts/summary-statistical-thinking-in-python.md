---
title: "Summary: Statistical Thinking in Python"
date: 2020-08-09T00:49:17-04:00
author: Ze Xuan Ong
feature_image: /summary-statistical-thinking-in-python/feature.png
draft: true
katex: true
markup: "mmark"
---

### Bee Swarm Plots


### Computing Empirical Cummulative Distribution Function (ECDF)

```
def ecdf(data):

    n = len(data)
    x = np.sort(data)
    y = np.arange(1, n+1) / n   # crucial bit about ECDF

    return x, y

x1, y1 = ecdf(data1)
x2, y2 = ecdf(data2)

# Generate multiple data sets on one plot
plt.plot(x1, y1, marker=".", linestyle="none")
plt.plot(x2, y2, marker=".", linestyle="none")

# Label the axes
plt.xlabel("Some x")
plt.ylabel("Some y")

# Display the plot
plt.show()


```

### Box Plots

Whiskers extend 1.5 IQRs. Generally, greater than 2 IQR considered outlier.

```
# Specify array of percentiles: percentiles
percentiles = np.array([2.5, 25, 50, 75, 97.5])

# Compute percentiles: ptiles_vers
ptiles_vers =np.percent


```


```
# Plot the ECDF
_ = plt.plot(x_vers, y_vers, '.')
_ = plt.xlabel('petal length (cm)')
_ = plt.ylabel('ECDF')

# Overlay percentiles as red diamonds.
_ = plt.plot(ptiles_vers, percentiles/100, marker='D', color='red', linestyle="none")

# Show the plot
plt.show()
```

{{< figure src="/summary-statistical-thinking-in-python/percentile.svg" caption="Plotting th eECDF and labelling the percentile points." >}}

```
# Create box plot with Seaborn's default settings
sns.boxplot("species", "petal length (cm)", data=df)

# Label the axes
plt.xlabel("species")
plt.ylabel("petal length (cm)")

# Show the plot
plt.show()

```

{{< figure src="/summary-statistical-thinking-in-python/box_and_whisker.svg" caption="Box and whisker plot." >}}

### Mean, Variance, StdDev

```
np.var(data)
np.std(data)
np.mean(data)


```

### Scatter Plot

Covariance $$ = \frac{1}{n} \sum^n_{i-1} (x_i - \bar{x}) (y_i - \bar{y}) $$

### Pearson Correlation

```
def pearson_r(x, y):
    """Compute Pearson correlation coefficient between two arrays."""
    # Compute correlation matrix: corr_mat

    corr_mat = np.corrcoef(x, y)

    # Return entry [0,1]
    return corr_mat[0,1]

# Compute Pearson correlation coefficient for I. versicolor: r
r = pearson_r(versicolor_petal_length, versicolor_petal_width)

# Print the result
print(r)

```


### Distributions

```
np.random.poisson(lam=1.0, size=None)
np.random.exponential(scale=1.0, size=None)
np.random.normal(mean, stdev, size=None)
np.random.binomial(n, p, size=None)

```
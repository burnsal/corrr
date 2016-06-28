corrr
================

corrr is a package for exploring **corr**elation matrices in **R**. It makes it possible to easily perform routine tasks when exploring correlation matrices such as comparing only some variables against others, or arranging the matrix in terms of the strength of the correlations, and so on. `corrr` also provides visualisation methods for extracting useful information such as variable clustering and latent dimensionality.

`corrr` is intended to be used for exploration and visualisation, NOT for statistical modeling (obtaining p values, factor analysis, etc.).

You can install:

-   the latest development version from github with

``` r
if (packageVersion("devtools") < 1.6) {
  install.packages("devtools")
}
devtools::install_github("drsimonj/corrr")
```

Getting Started
---------------

`corrr` centers around the `correlate()` function, which returns a correlation matrix generated by `stats::cor()`, but in a format of the following:

-   A tibble::data\_frame
-   An additional class, "cor\_df"
-   A "rowname" column
-   Standardised variances (the matrix diagonal) set to missing values (`NA`) so they can be ignored in calculations.

Another small adjustment is that `correlate()` uses pairwise deletion by default.

``` r
library(MASS)
library(corrr)
set.seed(1)

# Simulate three columns correlating about .7 with each other
mu <- rep(0, 3)
Sigma <- matrix(.7, nrow = 3, ncol = 3) + diag(3)*.3
seven <- mvrnorm(n = 1000, mu = mu, Sigma = Sigma)

# Simulate three columns correlating about .4 with each other
mu <- rep(0, 3)
Sigma <- matrix(.4, nrow = 3, ncol = 3) + diag(3)*.6
four <- mvrnorm(n = 1000, mu = mu, Sigma = Sigma)

# Bind together
d <- cbind(seven, four)
colnames(d) <- paste0("v", 1:ncol(d))

# Insert some missing values
d[sample(1:nrow(d), 100, replace = TRUE), 1] <- NA
d[sample(1:nrow(d), 200, replace = TRUE), 5] <- NA

# Correlate
x <- correlate(d)
class(x)
```

    ## [1] "cor_df"     "tbl_df"     "tbl"        "data.frame"

``` r
x
```

    ## Source: local data frame [6 x 7]
    ## 
    ##   rowname            v1          v2           v3            v4          v5
    ##     (chr)         (dbl)       (dbl)        (dbl)         (dbl)       (dbl)
    ## 1      v1            NA  0.70986371  0.709330652  0.0001947192 0.021359764
    ## 2      v2  0.7098637068          NA  0.697411266 -0.0132575510 0.009280530
    ## 3      v3  0.7093306516  0.69741127           NA -0.0252752456 0.001088652
    ## 4      v4  0.0001947192 -0.01325755 -0.025275246            NA 0.421380212
    ## 5      v5  0.0213597639  0.00928053  0.001088652  0.4213802123          NA
    ## 6      v6 -0.0435135083 -0.03383145 -0.020057495  0.4424697437 0.425441795
    ## Variables not shown: v6 (dbl)

Correlation matrices as Correlation Data Frames (cor\_df)
---------------------------------------------------------

By using the data\_frame structure, we can more easily leverage functions from packages like `dplyr`, `tidyr`, `ggplot2`, and so on. Below are some useful examples:

``` r
library(dplyr)

# Select a subset of variables by column
x %>% select(rowname, v1, v2)
```

    ## Source: local data frame [6 x 3]
    ## 
    ##   rowname            v1          v2
    ##     (chr)         (dbl)       (dbl)
    ## 1      v1            NA  0.70986371
    ## 2      v2  0.7098637068          NA
    ## 3      v3  0.7093306516  0.69741127
    ## 4      v4  0.0001947192 -0.01325755
    ## 5      v5  0.0213597639  0.00928053
    ## 6      v6 -0.0435135083 -0.03383145

``` r
# Select a subset of variables by rows (using filter)
x %>% filter(rowname %in% c("v1", "v2"))
```

    ## Source: local data frame [2 x 7]
    ## 
    ##   rowname        v1        v2        v3            v4         v5
    ##     (chr)     (dbl)     (dbl)     (dbl)         (dbl)      (dbl)
    ## 1      v1        NA 0.7098637 0.7093307  0.0001947192 0.02135976
    ## 2      v2 0.7098637        NA 0.6974113 -0.0132575510 0.00928053
    ## Variables not shown: v6 (dbl)

``` r
# Select columns and rows
x %>%
  filter(rowname %in% paste0("v", 4:6)) %>%
  select(rowname, v1:v3)
```

    ## Source: local data frame [3 x 4]
    ## 
    ##   rowname            v1          v2           v3
    ##     (chr)         (dbl)       (dbl)        (dbl)
    ## 1      v4  0.0001947192 -0.01325755 -0.025275246
    ## 2      v5  0.0213597639  0.00928053  0.001088652
    ## 3      v6 -0.0435135083 -0.03383145 -0.020057495

``` r
# Filter rows by correlation size
x %>% filter(v1 > .6)
```

    ## Source: local data frame [2 x 7]
    ## 
    ##   rowname        v1        v2        v3          v4          v5
    ##     (chr)     (dbl)     (dbl)     (dbl)       (dbl)       (dbl)
    ## 1      v2 0.7098637        NA 0.6974113 -0.01325755 0.009280530
    ## 2      v3 0.7093307 0.6974113        NA -0.02527525 0.001088652
    ## Variables not shown: v6 (dbl)

``` r
# Calculate the mean correlation for each variable
x %>%
  select(-rowname) %>%
  summarise_each(funs(mean(., na.rm = TRUE))) %>%
  round(2)
```

    ## Source: local data frame [1 x 6]
    ## 
    ##      v1    v2    v3    v4    v5    v6
    ##   (dbl) (dbl) (dbl) (dbl) (dbl) (dbl)
    ## 1  0.28  0.27  0.27  0.17  0.18  0.15

Manipulating cor\_df
--------------------

`corrr` provides convenience functions for routine manipulations of the matrix. In general, these functions begin with the letter `r`.

### focus()

`focus()` helps to focus in on a section of a `cor_df`. You use it similarly to selecting columns with `dplyr::select()`, but rows are also affected. As arguments, it takes your `cor_df`, expressions you would use in `select()`, and an optional boolean argument `rows`, indicating whether to keep the selected, or all other variables (default), in the rows. Here are some examples of using `focus()`:

``` r
# select v1 and v2 to stay in the columns but not rows
x %>% focus(v1, v2)
```

    ## Source: local data frame [4 x 3]
    ## 
    ##   rowname            v1          v2
    ##     (chr)         (dbl)       (dbl)
    ## 1      v3  0.7093306516  0.69741127
    ## 2      v4  0.0001947192 -0.01325755
    ## 3      v5  0.0213597639  0.00928053
    ## 4      v6 -0.0435135083 -0.03383145

``` r
# Keep in columns and rows (drop all others)
x %>% focus(v1, v2, rows = TRUE)
```

    ## Source: local data frame [2 x 3]
    ## 
    ##   rowname        v1        v2
    ##     (chr)     (dbl)     (dbl)
    ## 1      v1        NA 0.7098637
    ## 2      v2 0.7098637        NA

``` r
# Or put these variables into the rows by dropping from columns
x %>% focus(-v1, -v2)
```

    ## Source: local data frame [2 x 5]
    ## 
    ##   rowname        v3            v4         v5          v6
    ##     (chr)     (dbl)         (dbl)      (dbl)       (dbl)
    ## 1      v1 0.7093307  0.0001947192 0.02135976 -0.04351351
    ## 2      v2 0.6974113 -0.0132575510 0.00928053 -0.03383145

``` r
# And can use any dplyr::select() expressions
x %>% focus(num_range("v", 1:3))
```

    ## Source: local data frame [3 x 4]
    ## 
    ##   rowname            v1          v2           v3
    ##     (chr)         (dbl)       (dbl)        (dbl)
    ## 1      v4  0.0001947192 -0.01325755 -0.025275246
    ## 2      v5  0.0213597639  0.00928053  0.001088652
    ## 3      v6 -0.0435135083 -0.03383145 -0.020057495

### stretch()

`stretch()` converts your `cor_df` into a long-format data frame. It behaves similarly to `tidyr::gather()`. As arguments, it takes your `correlate()` correlation matrix, expressions you would use in `dplyr::select()` (as above), and some optional boolean arguments. The result is a three-column data frame with columns `x` and `y` storing variable names, and `r`, storing the correlation. Here are some examples of using `stretch()`:

``` r
# Convert all to long format
x %>% stretch(everything())
```

    ## Source: local data frame [36 x 3]
    ## 
    ##        x     y             r
    ##    (chr) (chr)         (dbl)
    ## 1     v1    v1            NA
    ## 2     v1    v2  0.7098637068
    ## 3     v1    v3  0.7093306516
    ## 4     v1    v4  0.0001947192
    ## 5     v1    v5  0.0213597639
    ## 6     v1    v6 -0.0435135083
    ## 7     v2    v1  0.7098637068
    ## 8     v2    v2            NA
    ## 9     v2    v3  0.6974112661
    ## 10    v2    v4 -0.0132575510
    ## ..   ...   ...           ...

``` r
# Select certain variables to convert to long format
x %>% stretch(v1, v3, v4)
```

    ## Source: local data frame [9 x 3]
    ## 
    ##       x     y             r
    ##   (chr) (chr)         (dbl)
    ## 1    v1    v1            NA
    ## 2    v1    v3  0.7093306516
    ## 3    v1    v4  0.0001947192
    ## 4    v3    v1  0.7093306516
    ## 5    v3    v3            NA
    ## 6    v3    v4 -0.0252752456
    ## 7    v4    v1  0.0001947192
    ## 8    v4    v3 -0.0252752456
    ## 9    v4    v4            NA

``` r
# Drop missing values (the diagonal by default) using na_omit = TRUE
x %>% stretch(num_range("v", 4:6), na_omit = TRUE)
```

    ## Source: local data frame [6 x 3]
    ## 
    ##       x     y         r
    ##   (chr) (chr)     (dbl)
    ## 1    v4    v5 0.4213802
    ## 2    v4    v6 0.4424697
    ## 3    v5    v4 0.4213802
    ## 4    v5    v6 0.4254418
    ## 5    v6    v4 0.4424697
    ## 6    v6    v5 0.4254418

``` r
# Or combine with shave() to set all duplicates to missing first
# and then omit (to retain each correlation just once)
x %>% shave() %>% stretch(everything(), na_omit = TRUE)
```

    ## Source: local data frame [15 x 3]
    ## 
    ##        x     y             r
    ##    (chr) (chr)         (dbl)
    ## 1     v1    v2  0.7098637068
    ## 2     v1    v3  0.7093306516
    ## 3     v1    v4  0.0001947192
    ## 4     v1    v5  0.0213597639
    ## 5     v1    v6 -0.0435135083
    ## 6     v2    v3  0.6974112661
    ## 7     v2    v4 -0.0132575510
    ## 8     v2    v5  0.0092805296
    ## 9     v2    v6 -0.0338314515
    ## 10    v3    v4 -0.0252752456
    ## 11    v3    v5  0.0010886516
    ## 12    v3    v6 -0.0200574953
    ## 13    v4    v5  0.4213802123
    ## 14    v4    v6  0.4424697437
    ## 15    v5    v6  0.4254417954

Visualising
-----------

`corrr` provides a number of methods for visualising a correlation matrix. Below are some examples.

`rplot()` creates a visual of the entire correlate() matrix. It uses the package `ggplot2` to produce a point plot with colouring, size and alpha to support visualisation.

``` r
mtcars %>% correlate() %>% rplot()
```

![](README_files/figure-markdown_github/rplot-1.png)

This is particularly useful if `rearrange()` is used first, as it rearranges the correlations to group highly correlated variables together. Some examples combining this and the above:

``` r
# Default settings
mtcars %>% correlate() %>% rearrange() %>% rplot()
```

![](README_files/figure-markdown_github/rplot_arranged-1.png)

``` r
# Using hierarchical clustering (method = "HC") and treating negative and
# positive correlations as distant (absolute = FALSE)
mtcars %>% correlate() %>% rearrange(method = "HC", absolute = FALSE) %>% rplot()
```

![](README_files/figure-markdown_github/rplot_arranged-2.png)

``` r
# As with stretch(), use shave() to screen out one of the triangles
mtcars %>% correlate() %>% rearrange() %>% shave() %>% rplot()
```

![](README_files/figure-markdown_github/rplot_arranged-3.png)

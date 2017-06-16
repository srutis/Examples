Example
-------

In [`R`](https://www.r-project.org) [`tidyeval`/`rlang`](https://CRAN.R-project.org/package=rlang) generates domains specific languages that, in my opinion, have somewhat involved laws. Here are some details paraphrased from [`vignette('programming', package = 'dplyr')`](https://cran.r-project.org/web/packages/dplyr/vignettes/programming.html)):

-   "`:=`" is needed to make left-hand-side re-mapping possible (adding yet another "more than one assignment type operator running around" notation issue).
-   "`!!`" substitution requires parenthesis to safely bind (so the notation is actually "`(!! )`, not "`!!`").
-   Left-hand-sides of expressions are names or strings, while right-hand-sides are `quosures`/expressions.

Let's apply tidyeval`/`rlang`notation to the task of building re-usable generic in [`R\`\](<https://www.r-project.org>).

``` r
suppressPackageStartupMessages(library("dplyr"))
packageVersion("dplyr")
```

    ## [1] '0.7.0'

``` r
d <- data.frame(a=1)
```

[`vignette('programming', package = 'dplyr')`](https://cran.r-project.org/web/packages/dplyr/vignettes/programming.html) the following example:

``` r
my_mutate <- function(df, expr) {
  expr <- enquo(expr)
  mean_name <- paste0("mean_", quo_name(expr))
  sum_name <- paste0("sum_", quo_name(expr))

  mutate(df, 
    !!mean_name := mean(!!expr), 
    !!sum_name := sum(!!expr)
  )
}

my_mutate(d, a)
```

    ##   a mean_a sum_a
    ## 1 1      1     1

So it seems the way to write general expressions (controlling both left-hand and right-hand sides) is the following:

``` r
tidy_add_one <- function(df, res_var, input_var) {
  input_var <- enquo(input_var)
  res_var <- quo_name(enquo(res_var))
  mutate(df,
         !!res_var := (!!input_var) + 1)
}

tidy_add_one(d, res, a)
```

    ##   a res
    ## 1 1   2

We can also write this using `base::substitute()` or `base::quote()` as usable "short forms" for what we are trying to do.

``` r
tidy_add_one_nse <- function(df, res_var, input_var) {
  input_var <- enquo(input_var)
  res_var <- quote(res_var)
  mutate(df,
         !!res_var := (!!input_var) + 1)
}

tidy_add_one_nse(d, res, a)
```

    ##   a res_var
    ## 1 1       2

And we can make a "standard evaluation" version using `base::as.name()`:

``` r
tidy_add_one_se <- function(df, res_var_name, input_var_name) {
  input_var <- as.name(input_var_name)
  res_var <- res_var_name
  mutate(df,
         !!res_var := (!!input_var) + 1)
}

tidy_add_one_se(d, 'res', 'a')
```

    ##   a res
    ## 1 1   2

wrapr::let
----------

It is easy to specify the function we want with [`wrapr`](https://CRAN.R-project.org/package=wrapr) as follows (both using standard evaluation, and using non-standard evaluation):

``` r
library("wrapr")

wrapr_add_one_se <- function(df, res_var_name, input_var_name) {
  wrapr::let(
    c(RESVAR= res_var_name,
      INPUTVAR= input_var_name),
    df %>%
      mutate(RESVAR = INPUTVAR + 1)
  )
}

wrapr_add_one_se(d, 'res', 'a')
```

    ##   a res
    ## 1 1   2

``` r
wrapr_add_one_nse <- function(df, res_var, input_var) {
  wrapr::let(
    c(RESVAR= substitute(res_var),
      INPUTVAR= substitute(input_var)),
    df %>%
      mutate(RESVAR = INPUTVAR + 1)
  )
}

wrapr_add_one_nse(d, res, a)
```

    ##   a res
    ## 1 1   2

Or, if you are uncomfortable with macros being implemented through string-substitution one can use `wrapr::let()` in "language mode" (where it works directly on abstract syntax trees).

``` r
wrapr_add_one_se <- function(df, res_var_name, input_var_name) {
  wrapr::let(
    c(RESVAR= res_var_name,
      INPUTVAR= input_var_name),
    df %>%
      mutate(RESVAR = INPUTVAR + 1),
    subsMethod= 'langsubs'
  )
}

wrapr_add_one_se(d, 'res', 'a')
```

    ##   a res
    ## 1 1   2

``` r
wrapr_add_one_nse <- function(df, res_var, input_var) {
  wrapr::let(
    c(RESVAR= substitute(res_var),
      INPUTVAR= substitute(input_var)),
    df %>%
      mutate(RESVAR = INPUTVAR + 1),
    subsMethod= 'langsubs'
  )
}

wrapr_add_one_nse(d, res, a)
```

    ##   a res
    ## 1 1   2

Conclusion
----------

`tidyeval`/`rlang` provides general tools to compose or daisy-chain non-standard-evaluation functions (i.e., write new non-standard-evaluation functions in terms of others. This abrogates the issue that it can be hard to compose non-standard function interfaces (i.e., one can not [parameterize them or program over them](https://www.youtube.com/watch?v=iKLGxzzm9Hk) without a tool such as `tidyeval`/`rlang`). `wrapr::let()` concentrates on standard evaluation, providing a tool that allows one to re-wrap non-standard-evaluation interfaces as standard evaluation interfaces.

I think the `tidyeval`/`rlang` philosophy is a "tools to application view" and `wrapr::let()` is a "use-case to tool view." These are differing views, so each will artificially look "silly" if judged in terms of the other.

A lot of the `tidyeval`/`rlang` design is centered on treating variable names as lexical closures that capture an environment they should be evaluated in. This does make them more like general `R` functions (which also have this behavior).

However, creating so many long-term bindings is a actually counter to some common data analyst practice.

The `my_mutate(df, expr)` example itself from `vignette('programming', package = 'dplyr')` even shows the pattern I am referring to: the analyst transiently pairs abstract variable names to a chosen concrete data set. One argument is the data and the other is the expression to be applied to that data (and only that data, with clean code not capturing values from environments).

Many calls are written this way (for example `predict()`) and it has the huge advantage that it documents your intent to change out what data is being applied (such as running a procedure twice, once on training data and once on future application data).

This is a principle we also strongly apply in our [join controller](http://www.win-vector.com/blog/2017/06/use-a-join-controller-to-document-your-work/) which has no issue sharing variables out as an external spreadsheet, because it thinks of variable names as fundamentally being strings (not as `quosures` temporally working "under cover" in string representations).
<!-- badges: start -->

[![R-CMD-check](https://github.com/mpadge/typetracer/workflows/R-CMD-check/badge.svg)](https://github.com/mpadge/typetracer/actions)
[![codecov](https://codecov.io/gh/mpadge/typetracer/branch/main/graph/badge.svg)](https://codecov.io/gh/mpadge/typetracer)
<!-- badges: end -->

# typetracer

`typetracer` is an R package to trace function parameter types. The main
usage of `typetracer` is to identify parameters used as input to R
functions. Many computer languages have formal type systems, meaning the
types of parameters must be formally declared and encoded. R is
different, and offers no way to specify the expected types of input
parameters. `typetracer` identifies the types of parameters passed to R
functions. The package can trace individual functions or entire
packages, as demonstrated below.

## Installation

The package can be installed with the following command:

    remotes::install_github ("mpadge/typetracer")

Then loaded for use by calling `library`:

    library (typetracer)

## Example \#1 - A Single Function

`typetracer` works by “injecting” tracing code into the body of a
function using the `inject_tracer()` function. The following function
includes four parameters, including `...` to allow passing of additional
and entirely arbitrary parameter types and values.

    f <- function (x, y, z, ...) {
        x * x + y * y
    }
    inject_tracer (f)

After injecting the `typetracer` code, calls to the function, `f`, will
“trace” each parameter of the function, by capturing both unevaluated
and evaluated representation at the point at which the function is first
called. These values can be accessed with the `load_traces` function,
which returns a `data.frame` object (in [`tibble`
format](https://tibble.tidyverse.org) with one row for each parameter
from each function call.

    val <- f (x = 1:2, y = 3:4 + 0., a = "blah", b = list (a = 1, b = "b"))
    x <- load_traces ()

Traces themselves are saved in the temporary directory of the current R
session, and the `load_traces()` function simple loads all traces
created in that session. The function `clear_traces()` removes all
traces, so that `load_traces()` will only load new traces produced after
that time.

## Example \#2 - Tracing a Package

This section presents a more complex example tracing all function calls
from [the `rematch` package](https://github.com/MangoTheCat/rematch),
chosen because it has less code than almost any other package on CRAN.
The following single line traces function calls in all examples for the
nominated package:

    res <- trace_package ("rematch")
    res

    ## # A tibble: 8 × 9
    ##   fn_name  fn_call_hash par_name class     storage_mode length formal     uneval
    ##   <chr>    <chr>        <chr>    <I<list>> <chr>         <int> <named li> <I<li>
    ## 1 re_match f6hXTC9p     pattern  <chr [1]> character         1 <missing>  <chr> 
    ## 2 re_match f6hXTC9p     text     <chr [1]> character         7 <missing>  <chr> 
    ## 3 re_match f6hXTC9p     perl     <chr [1]> logical           1 <lgl [1]>  <chr> 
    ## 4 re_match f6hXTC9p     ...      <chr [1]> NULL              0 <missing>  <chr> 
    ## 5 re_match xQNA3X5z     pattern  <chr [1]> character         1 <missing>  <chr> 
    ## 6 re_match xQNA3X5z     text     <chr [1]> character         7 <missing>  <chr> 
    ## 7 re_match xQNA3X5z     perl     <chr [1]> logical           1 <lgl [1]>  <chr> 
    ## 8 re_match xQNA3X5z     ...      <chr [1]> NULL              0 <missing>  <chr> 
    ## # … with 1 more variable: eval <I<list>>

The result contains one line for every parameter passed to every
function call in the examples. The `trace_package()` function also
includes an additional parameter, `types`, which defaults to
`c ("examples", "tests")`, so that traces are also by default generated
for all tests included with local source packages.

The final two columns of the result hold the unevaluated and evaluated
representations of each parameter. The first two values of each
demonstrate the difference:

    res$uneval [1:2]

    ## $pattern
    ## [1] "isodaten"
    ## 
    ## $text
    ## [1] "dates"

    res$eval [1:2]

    ## $pattern
    ## [1] "(?<year>[0-9]{4})-(?<month>[0-1][0-9])-(?<day>[0-3][0-9])"
    ## 
    ## $text
    ## [1] "2016-04-20"       "1977-08-08"       "not a date"       "2016"            
    ## [5] "76-03-02"         "2012-06-30"       "2015-01-21 19:58"

The example first assigns a variable `isodaten` to the first of the
evaluated values, and then calls the function with `pattern = isodaten`.
The second constructs the vector called `dates` with the second of the
evaluated values, then calls the function with `test = dates`.

## Examples \#3 - Non-standard Evaluation

This example briefly illustrates some examples of tracing parameters
evaluated in non-standard ways. This first examples demonstrates that
parameter values are captured at the initial point of function entry.

    eval_x_late_NSE <- function (x, y) {
        y <- 10 * y
        eval (substitute (x))
    }
    inject_tracer (eval_x_late_NSE)
    eval_x_late_NSE (y + 1, 2:3)

    ## [1] 21 31

    res <- load_traces ()
    res$par_name

    ## [1] "x" "y"

    res$uneval

    ## $x
    ## [1] "y + 1"
    ## 
    ## $y
    ## [1] "2:3"

    res$eval

    ## $x
    ## [1] 3 4
    ## 
    ## $y
    ## [1] 2 3

The parameter `x` is evaluated at the point of function entry as `y + 1`
which, with a value of `y = 2:3`, gives the expected evaluated result of
`x = 3:4`, while the function ultimately returns the expected values of
`(10 * 2:3) + 1`, or `21 31`, because the first line of `y <- 10 * y` is
evaluated prior to substituting the value passed for `x` of `y + 1`.

The second example specifies a default value of `x = y + 1`, with the
actual call passing no value, and thus having `"NULL"` in the
unevaluated version, while evaluated versions remain identical.

    clear_traces () # clear all preceding traces
    eval_x_late_standard <- function (x = y + 1, y, z = y ~ x) {
        y <- 10 * y
        x
    }
    inject_tracer (eval_x_late_standard)
    eval_x_late_standard (, 2:3)

    ## [1] 3 4

    res <- load_traces ()
    res$par_name

    ## [1] "x" "y" "z"

    res$uneval

    ## $x
    ## [1] "NULL"
    ## 
    ## $y
    ## [1] "2:3"
    ## 
    ## $z
    ## [1] "NULL"

    res$eval

    ## $x
    ## [1] 3 4
    ## 
    ## $y
    ## [1] 2 3
    ## 
    ## $z
    ## y ~ x
    ## <environment: 0x5604bd289c98>

The traces produced by `typetracer` also include a column, `formal`,
which contains the default values specified in the definition of
`eval_x_late_standard()`:

    res$formal

    ## $x
    ## y + 1
    ## 
    ## $y
    ## 
    ## 
    ## $z
    ## y ~ x

Those three columns of `formal`, `uneval`, and `eval` thus contain all
definitions for all parameters passed to the function environment, in
the three possible states of:

1.  Formal or default values (by definition, in an unevaluated state);
2.  The unevaluated state of any specified parameters; and
3.  The equivalent versions evaluated within the function environmental.

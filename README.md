# Trouble with the Beta-Binomial lpmf in StanHeaders 2.21.0-7 

If you are fitting Binomial models with overdispersion, and have recently (as of 17.6.2022) installed rstan via R's package manager, this might affect you. Yes - you! 

So what's going on?

Let's begin with some prequisites.

## Tilde (~) and target += lpmf_somepmf/pdf are not the same!

Using the tilde-operator in Stan drops additive constants from the log probability calculations. In contrast to this, using the target += some_pmf_or_cdf(x | theta) notation retains the constants. 

Unluckily, how this is done is a bit broken in the Beta-Binomial probability mass function. 

Let us look into this in a bit more detail.

## Well, what's wrong with the beta_binomial_lpmf?

The problematic part is found on lines 77-86 in StanHeaders/include/stan/math/prim/scal/prob/beta_binomial_lpmf.hpp:

```
  if (include_summand<propto>::value) {
    for (size_t i = 0; i < size; i++) {
      // normalizing constant
      logp += binomial_coefficient_log(N_vec[i], n_vec[i]);
      // log numerator - denominator
      logp += lbeta(n_vec[i] + value_of(alpha_vec[i]),
                    N_vec[i] - n_vec[i] + value_of(beta_vec[i]))
              - lbeta(value_of(alpha_vec[i]), value_of(beta_vec[i]));
    }
  }
```

The if-statement in the beginning checks for the propto-agument that controls whether to use the normalized or the unnormalized version of the, in this case, probability mass function: if the propto argument is false, this block of code is not executed and the intended effect is to just not have the normalizing constant (binomial coefficient) being added to the logp-variable and the computations being ever so slightly more efficient. However due to slip of the finger by the programmer not only the normalizing constant but the whole pmf of the Beta-Binomial distribution is included in that optional block! So setting propto to false does not drop only the normalizing constant *but the whole pmf*, oops... 

This seems the have been fixed at leats in the development version[1], but using the usual install.packages rigmarole in R still installs the version with the faulty lpmf, so I'm writing this as 1) a reminder for myself 2) public safety announcment and 3) how to quickly fix this behaviour. 

Luckily this is easy to fix: just use the target += etc. version and use the normalized pmf! But, okay, the code can be fixed too so that the unnormalized version is available too, it's easy. Just edit the block above so that only the binomial_coefficient_log thing, the normalizing constant, is left inside the optional block, and the other pmf calculations are outside it. Don't forget the for-loop!
part

```  
  // log numerator - denominator
  for (size_t i = 0; i < size; i++) {
  	logp += lbeta(n_vec[i] + value_of(alpha_vec[i]),
                    N_vec[i] - n_vec[i] + value_of(beta_vec[i]))
              - lbeta(value_of(alpha_vec[i]), value_of(beta_vec[i]));
  }
```

So if you are still using a version of StanHeaders that has the faulty version of beta_binomial_lpmf, and you want to use that pmf, I recommend either using the normalized version or applying that little fix, which *seems* to have fixed the problem, although I haven't tested it thoroughly or in the context of complex models. 


## Sources

[1] https://github.com/stan-dev/math/blob/develop/stan/math/prim/prob/beta_binomial_lpmf.hpp

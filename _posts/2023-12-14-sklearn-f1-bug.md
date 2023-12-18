---
title: "Scikit-Learn's F-1 calculator is broken"
author: "Connor Boyle"
---

**TL;DR:** if you are using scikit-learn 1.3.X and
use [`f1_score()`](https://scikit-learn.org/stable/modules/generated/sklearn.metrics.f1_score.html)
or [`classification_report()`](https://scikit-learn.org/stable/modules/generated/sklearn.metrics.classification_report.html)
with the argument `zero_division=1.0` or `zero_division=np.nan`, then there's a chance that the output of that function
is wrong (possibly by any amount up to 100%, depending on the number of classes in your
dataset). E.g. for `zero_division=1.0`:

<pre>
>>> sklearn.__version__
'1.3.0'
>>> sklearn.metrics.f1_score(y_true=list(range(104)), y_pred=list(range(100)) + [101, 102, 103, 104], average='macro', zero_division=1.0)
<b>0.9809523809523809</b>  <i># incorrect</i>
</pre>

compare to (the exact same expression in an earlier version of Scikit-Learn):

<pre>
>>> sklearn.__version__
'1.2.2'
>>> sklearn.metrics.f1_score(y_true=list(range(104)), y_pred=list(range(100)) + [101, 102, 103, 104], average='macro', zero_division=1.0)
<b>0.9523809523809523</b>  <i># correct</i>
</pre>

Similar cases for `zero_division=np.nan` (which was introduced in 1.3.0, so I can't directly compare to the output in
1.2.2):

<pre>
>>> sklearn.metrics.f1_score([0, 1], [1, 0], average='macro', zero_division=np.nan)
<b>nan</b>  <i># should be 0.0</i>
>>> sklearn.metrics.f1_score([0, 1, 2], [1, 0, 2], average='macro', zero_division=np.nan)
<b>1.0</b>  <i># should be ~0.67</i>
</pre>

Both myself and the Scikit-Learn maintainers consider the behavior in 1.3.X to be incorrect. While a
[pull request](https://github.com/scikit-learn/scikit-learn/pull/27577) to fix this behavior was just merged, the fix
has not yet shipped on any released version of Scikit-Learn. Therefore, the easiest solution to this specific problem is
to revert to Scikit-Learn 1.2.2, or use `zero_division=0.0` if possible, while being careful to understand how this
parameter change will affect precision, recall, & F-1 (see below for an explainer on the purpose and function of
the `zero_division` parameter).

The problem is that F-1 for an individual class is getting calculated as `1.0` or `np.nan` when precision & recall are
both `0.0` (which is *not* the desired behavior for the `zero_division` parameter).

## How did this happen?

Let's take a look at the formulae for the metrics in this classification report:

$$ \textrm{precision} = \frac{\textrm{true positive}}{\textrm{true positive} + \textrm{false positive}} $$

$$ \textrm{recall} = \frac{\textrm{true positive}}{\textrm{true positive} + \textrm{false negative}} $$

$$ \textrm{F-1} = \frac{2 \cdot \textrm{precision} \cdot \textrm{recall}}{\textrm{precision} + \textrm{recall}} $$

There are three different places here where a division by zero can occur:

- in precision, if `true positive + false positive = 0` (the classifier made no
  positive predictions for the class)
- in recall, if `true positive + false negative = 0` (there are no truly
  positive examples of the class in the dataset)
- in F-1, if `precision = recall = 0` (the classifier has made a nonzero number
  of exclusively incorrect predictions) 

Two of these are interesting cases where reasonable people could disagree on
what the correct behavior should be:

- When the classifier has made *zero* positive predictions for the class, should that count as a precision of 1.0? If
  "perfect precision" is interpreted as "no false positives", then this is totally reasonable behavior.
- When the gold dataset has *zero* true positive examples of the class, should that count as a recall of 1.0? This is a
  much more unusual scenario than the "zero positive predictions" example--a good evaluation dataset should almost never
  be entirely missing a class. However, this can realistically occur when evaluating on subsets of a large multiclass
  dataset. Again, if the definition of "perfect recall" is taken as "no false negatives", then assigning a recall of 1.0
  in this case is totally reasonable behavior.

For F-1, however, the "division by zero" case is not interesting or controversial in any way. If a classifier has
achieved a recall of 0.0 (all negative predictions are false) *and* a precision of 0.0 (all positive predictions are
false), I don't think any reasonable person would disagree what the F-1 score should be: 0.0. Indeed, this is exactly
how Scikit-Learn calculated F-1 right up to (and including) version 1.2.2, regardless of the value of
the `zero_division` parameter.

However, in Scikit-Learn 1.3.0, the `zero_division` parameter was turned into a kind
of [monkey's paw](https://en.wiktionary.org/wiki/monkey%27s_paw) that defines the behavior of *any* division-by-zero
that happens to occur during the calculation of an F-1 score, leading to the bizarre scenario where a 100% wrong
classifier can get an F-1 score of 100%:[^1]

```
>>> sklearn.__version__
'1.3.0'
>>> print(sklearn.metrics.classification_report(y_true=[0, 1, 2, 3, 4], y_pred=[1, 2, 3, 4, 0], zero_division=1.0))
              precision    recall  f1-score   support

           0       0.00      0.00      1.00       1.0
           1       0.00      0.00      1.00       1.0
           2       0.00      0.00      1.00       1.0
           3       0.00      0.00      1.00       1.0
           4       0.00      0.00      1.00       1.0

    accuracy                           1.00       5.0
   macro avg       0.00      0.00      1.00       5.0
weighted avg       0.00      0.00      1.00       5.0

```

Why? Because precision and recall are both 0, which means the denominator of the F-1 formula is 0,
and `zero_division=1.0` now (as of Scikit-Learn 1.3.0) applies to the F-1 calculation itself, so that means F-1 is
calculated (incorrectly) as 1.0!


## Why does this matter?

I don't know if there are rigorous statistics on this, but I'd wager that macro average F-1 is the most commonly used
metric for multiclass classification by a wide margin. Scikit-Learn's `f1_score()` function is in turn very likely the
most commonly used implementation of F-1. Try asking Google or ChatGPT how to calculate F-1; the first results will very
likely tell you to use this exact function in Scikit-Learn.

The kinds of tasks F-1 could be used for range from low-risk, like sentiment analysis on customer reviews, to some
conceivably really safety-critical things. Imagine a researcher at an autonomous car company thinks their computer
vision system is performing really well at recognizing all categories of objects & entities on the road. But actually,
their classifier is completely missing every single example of a few classes!

Ideally, any machine learning practitioner probably *should* notice this bug well before a classifier is put into
production or reporting results in a submitted journal paper. On the other hand, you really would not expect the
definition of F-1 to change from one version of Scikit-Learn to the next! While just about any programmer should be able
to implement an F-1 calculator in very little time, most of us prefer to just import Scikit-Learn *specifically* to
avoid gotcha edge cases like this one.

## What should I do now?

If your project:

- is using, has used, or may have used any Scikit-Learn version starting with 1.3.0 (released 2023-06-30)
- contains any call to `classification_report()`, `f1_score()`, or `fbeta_score()`, and
- that call contains the parameter `zero_division=1.0` or `zero_division=0.0`

it may have been affected by this bug. To determine if any particular F-1 score calculation was impacted by this bug,
first change that F-1 score calculation to a `classification_report()` if possible. If any class in that classification
report contains a precision of `0.0`, a recall of `0.0`, and an f1-score of `1.0` or `nan`, then the F-1 score for this
classifier has been calculated incorrectly.

Any call using `zero_division=1.0` can be fixed by reverting to Scikit-Learn version 1.2.2. Unfortunately, the
parameter `zero_division=np.nan` did not exist in Scikit-Learn 1.2.2, and I don't believe there is any easy way to
replicate it.

[^1]: A completely wrong classifier can also get an F-1 score of 0.0 in Scikit-Learn 1.3.X, for example: 
    ```
    >>> print(sklearn.metrics.classification_report(y_true=[0, 0, 0], y_pred=[1, 1, 1], zero_division=1.0))
                  precision    recall  f1-score   support

               0       1.00      0.00      0.00       3.0
               1       0.00      1.00      0.00       0.0

        accuracy                           1.00       3.0
       macro avg       0.50      0.50      0.00       3.0
    weighted avg       1.00      0.00      0.00       3.0
    ```

    (correctly) receives an F-1 of 0.0, because in each class, *either* precision *or* recall (but never both) is zero,
    which means that the denominator of the F-1 score for each class is nonzero.
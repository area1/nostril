Nostril: Nonsense String Evaluator
==================================

Nostril (_Nonsense String Evaluator_) is a Python 3 package is designed to detect whether a given medium-length string of characters is likely to be random gibberish or something meaningful.  It makes a binary distinction: either a given input is (probably) nonsense, or it is (probably) not.  The main use case is to decide whether strings returned by source code mining methods are likely to be (e.g.) program identifiers, or random characters or other non-identifier strings.

Table of contents
-----------------

* [Usage](#usage)
* [Known limitations](#known-limitations)
* [Operating principles](#operating-principles)
* [Tuning, training and testing](#tuning-training-and-testing)
* [Finding parameter values](#finding-parameter-values)
* [Other approaches tried](#other-approaches-tried)

Usage
-----

The basic usage is very simple.  _Nostril_ is a Python package that (among other things) provides a function named `nonsense()`.  This function takes a text string as an argument and returns a Boolean value as a result.  Here is an example:

    from nostril import nonsense
    for s in ['bunchofwords', 'xywinlist', 'faiwtlwexu', 'asfgtqwafazfy']:
        if nonsense(s):
           print("{} is nonsense".format(s))
        else:
           print("{} is real".format(s))

The value returned by `nonsense` will be `True` if the input string is probably meaningless and `False` if it is probably not.  (Note: the first time you import the module `nostril`, it will take extra time because Nostril loads a large data file during the import step.  If you only execute the short two-line example above, it will seem that Nostril takes far too long to evaluate a string.  This is misleading: one it's loaded, multiple calls to `nonsense()` are very fast.)

Nostril includes parameters that can be adjusted.  Internally, Nostril also uses a table of precomputed [n-gram](https://en.wikipedia.org/wiki/N-gram) weights that were derived by training the system on data sets of labeled test cases.  A saved copy of the trained values are stored in this directory, so that it is not necessary to retrain the system to adjust the tunable parameters.  However, it is _also_ possible to retrain the system and recompute the n-gram weights.  This process is somewhat more involved, and discussed below.

Known limitations
-----------------

Nostril is not fool-proof; **it _will_ generate some false positive and false negatives**.  This is an unavoidable consequence of the problem domain: without direct knowledge, even a human cannot recognize a real text string in all cases.  Nostril's default trained system puts emphasis on reducing false positives (i.e., reducing how often it mistakenly labels something as nonsense) rather than false negatives, so it will sometimes report that something is not nonsense when it really is.

A vexing result is that this system does more poorly on supposedly "random" strings typed by a human.  I hypothesize this is because those strings may be less random than they seem: if someone is asked to type junk at random on a QWERTY keyboard, they are likely to use a lot of characters from the home row (a-s-d-f-g-h-j-k-l), and those actually turn out to be rather common in English words.  In other words, what we think of a strings "typed at random" on a keyboard are actually not that random, and probably have statistical properties similar to those of real words.  These cases are hard for Nostril, but thankfully, in real-world situations, they are rare.  This view is supported by the fact that Nostril's performance is much better on statistically random text strings generated by software.

Nostril has been trained using American English words, and is unlikely to work for other languages unchanged.  However, the underlying framework may work if it were retrained on different sample inputs.  Nostril uses uses [n-grams](https://en.wikipedia.org/wiki/N-gram) coupled with a custom [TF-IDF](https://en.wikipedia.org/wiki/Tf–idf) weighting scheme.  See the subdirectory `training` for the code used to train the system.

Finally, the algorithm does not perform well on very short text, and by default Nostril imposes a lower length limit of 6 characters &ndash; strings must be longer than 6 characters or else it will raise an exception.

Operating principles
--------------------

Nostril's approach is conceptually simple.  It uses a combination of (1) a prefilter that detects simple positive and negative cases using heuristic rules and (2) a custom [TF-IDF](https://en.wikipedia.org/wiki/Tf–idf) scoring scheme that uses letter 4-grams as features.  When testing a string, Nostril's algorithm first uses the hardwired prefilter to detect either high-probability nonsense or high-probability meaningful strings.  If a string passes the prefilter, Nostril then applies a TF-IDF scoring method that uses a custom mathematical formula.  If the formula value exceeds a fixed threshold, the string is rated as being nonsense.  To make an analogy to how TF-IDF is used in document classification, nonsense strings are those that use unusual n-grams and thus score highly, while real/meaningful strings are those that use more common n-grams and thus score lower.  The Nostril package includes a precomputed table of [n-gram](https://en.wikipedia.org/wiki/N-gram) weights derived by training the system on a large set of strings constructed from concatenated American English words, real text corpora, and other inputs.  Parameter values were optimized using the evolutionary algorithm [NSGA-II](https://link.springer.com/chapter/10.1007%2F3-540-45356-3_83).

There are some minor innovations in Nostril in the way it calculates TF-IDF scores.  First, empty n-grams in the n-gram frequency table (meaning, n-grams that were never seen during training) are given high values to reflect the fact that they are unusual.  Second, a power function is applied to repeated n-grams in a string, to raise the score of strings that contain embedded repeats of the same pattern: i.e., things like "SomethingHellohellohellohellohellohello"&mdash;it contains real words (not random characters) yet probably does not represent a meaningful identifier.  (Without a correction for repetition, strings like this would otherwise not score highly because they match common n-grams.)  Finally, there is a length-dependent factor applied to strings longer than about 28 characters, such that a string's score is raised slowly the longer the string is past 28 characters.  This helps detect very long strings that happen to use common n-grams *without* repeats: while long identifiers are not unusual in programming contexts, the longer they are the more likely they are nonsense rather than something a programmer wrote by hand.

Training uses a set that is constructed from (1) a few thousand real identifier strings taken from actual software, (2) a set of about 30,000 words taken from various contemporary text corpora, (3) a set of common stop words, and (4) a few million strings created by randomly concatenating items from 2-3 (but not the real identifiers, which are left as-is).  The current stored results were produced after experimenting with 2-grams, 3-grams, 4-grams and 5-grams, and different thresholds.  The best performance was achieved using 4-grams, and that is the value stored in the `ngram_data.pklz` pickle file provided with Nostril.  The parameter values used in the nonsense evaluation function were found initially by extensive hand-tuning, and finally by using a multiobjective optimization method implemented in the Python package [Platypus-Opt](http://platypus.readthedocs.io/en/latest/).

Tuning, training and testing
----------------------------

It is possible to tune some of the parameters used by the classifier.  The parameters are part of the mathematical function used internally by Nostril to compute a score for a given input string.  To get a pointer to a new classifier function (a closure) with different values of the tunable parameters, call `generate_nonsense_detector` like so:

    from nostril import generate_nonsense_detector
    nonsense = generate_nonsense_detector(...)

where `...` are parameters that are explained in the help string.  The new function `nonsense()` obtained by calling the generator this way can be used exactly as the default version of `nonsense()`:

    result = nonsense('yoursinglestringhere')

The final performance of the nonsense detector is dependent on many things: the characteristics of the training set, the length of the n-grams used, the parameters in `string_score()`, and thresholds set in the function `generate_nonsense_detector()`.

The comments in the file `training.py` provide information about how the system is trained.  Basically, the process begins by generating a lot of strings that are representative of program identifiers, then computing n-gram frequency scores (IDF &ndash; inverse document frequency scores) and storing them in a Python dictionary.  Then comes a period of adjusting the parameters in the function `string_score()` and the thresholds in `generate_nonsense_detector()`.  This can be done manually by scoring a lot of both real and nonsense strings with the detector function created by `generate_nonsense_detector()`, then guessing at likely values for the thresholds and parameters, then re-scoring the example strings again, and iterating this process until the detector function created by `generate_nonsense_detector()` produces good results on real and random strings.  A better method for finding optimal parameter values is to use a multiobjectve optimization algorithm.  Nostril's parameter values were initially derived manually and then fine-tuned using the NSGA-II (Non-dominated Sorting Genetic Algorithm) routine in [Platypus-Opt](https://github.com/Project-Platypus/Platypus).

One of the characteristics of the training set influencing classification performance is how the synthetic identifier strings are generated.  The training set creation function, `training_set()`, takes a parameter for the maximum number of words to concatenate.  I experimented with 2-5 words, and found that a low number of 2 produced the best results (at least when combined with 4-grams).  The current hypothesis for why a low (rather than high) number is better is that concatenating real words at random produces character sequences that are random at the juncture points.  Since all the strings are scored for n-grams, this increases n-gram IDF scores for some n-grams that are likely to be random strings in real identifiers.  Now, this is not _completely_ undesirable: after all, programmers often combine acronyms, shorthand, and unusual words when creating identifiers, and parts of those identifiers often really do look like random character sequences.  So, there is a balancing act here, in which we try to have some realistic randomness (but not too much) in the training set.  The value of 2 for the parameter `max_concat_words` in `training_set()` seems to produce the best results.

For the record, here is what I ultimately did to produce the final values in `ngram_frequencies.pklz`.  With the training set generation function, `training_set()`, I experimented with setting `max_words` to 2-5, and ultimately had best results with a value of 2.

    words = english_word_list()
    ids = identifiers_from_file('random-identifiers-from-github.txt')
    ts = training_set(words, ids, 3000000, 2)

The creation of the n-gram frequency table is as follows (here using 4-grams):

    freq = ngram_frequencies(ts, 4)

You can save the values of things like this:

    dataset_to_pickle('ngram_frequencies.pklz', freq)
    dataset_to_pickle('training/training_set.pklz', ts)

Finding parameter values
------------------------

The actual nonsense detector function is a closure created by `generate_nonsense_detector()`:

    nonsense = generate_nonsense_detector(...)

The optional parameters given as arguments to `generate_nonsense_detector(...)` are used in creating the scoring function.  If trying to find parameter values manually, you have to iterate between picking parameter values for `generate_nonsense_detector()` while looking at the results produced by the nonsense detector, creating a new version of the nonsense detector with different parameter values, etc., ad nauseam, until the output of `nonsense()` starts to improve.  As a starting point to finding a threshold, you can tabulate samples of n-gram scores using

    tabulate_scores(ts, freq)

You can test like this:

    rs = random_set(ts)

    test_strings(rs[:100000], nonsense)
    test_strings('/usr/share/dict/web2', nonsense)
    test_labeled('tests/labeled-cases/real-not-real.csv', nonsense)

An optimization method can automate the process of finding parameter values, but it is not easy to find optimal values in the present context.  The problem with judging whether a given text string is nonsense or real is that it is essentially impossible to be certain in all cases.  Even a human can't be certain: some identifiers used by people in their code are composed of ad hoc acronyms and contractions that are only meaningful to the author, which means that another reader may easily mistake real identifiers for gibberish.  Thus, we are faced with balancing false positives against false negatives, and the final choice of parameter values must be made subjectively according to whether we want to tolerate more false positives or more false negatives.

This is a problem of multiobjective optimization: we have several objectives that conflict, and the optimization method needs to minimize all of them simultaneously.  Nostril's final parameter values were found using the code in the file `optimize.py`.  The optimization code uses [Platypus](https://github.com/Project-Platypus/Platypus) to run an evolutionary algorithm, which generates a set of results from which a final choice must be made by a human.  Using the training set created as described above, I ended up with the following parameter values for the nonsense detector:

    length threshold = 25
    minimum score = 8.47
    length penalty exponent = 0.9233
    repetition penalty exponent = 0.9674

Running `optimize.py` takes a long time: 7 hours on a late-2015 Apple iMac with 4 cores (4 GHz Intel Core i7) and 32 GB of RAM.

Other approaches tried
----------------------

Other methods can be used to evaluate whether given strings are real or nonsense.  For Nostril, I attempted three other approaches based on the Naive Bayes method, because Naive Bayes has had tremendous success in email spam detection and the discrimination made by Nostril has some similarity to the problem of email spam detection.  The variants were:

* A straightforward multinomial Naive Bayes algorithm using n-grams and term frequencies (TF) as described in the book _Introduction to Information Retrieval_ by Manning et al. (2009, Cambridge University Press).

* A Bernoulli Naive Bayes method, as described in the same book by Manning et al. method above.

* A generalized multinomial Naive Bayes algorithm using n-grams and term frequencies, based on the paper "Combining Modifications to Multinomial Naive Bayes for Text Classification" by A. Puurula (AIRS conference 2012).

The code for these alternative approaches is available in the diretory [../nostril/variants](../nostril/variants), for reference purposes.  The Bernoulli Naive Bayes method produced noticeably worse results than the other variants, and I did not pursue it far.  With the other two variants, I expended considerable time trying to make them work.  This included optimizing the parameter values using multiobjective optimization methods.  However, neither of the MNB approaches above were able to achieve the level of performance provided by the custom TF-IDF evaluation method implemented in `generate_nonsense_detector()`.

Due to the fact that Nostril's current TF-IDF approach produces very good performance, and the MNB approach failed to improve on it, I did not pursue other possible approaches.  One can imagine trying additional variations on MNB, as well as machine-learning methods such as kNN and SVM, and other methods such as text compression.  This is left to future work.

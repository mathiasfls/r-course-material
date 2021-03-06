Choosing the number of topics (and other parameters) in a topic model
================
Wouter van Atteveldt & Kasper Welbers
November 2019

-   [Introduction](#introduction)
-   [Calculating perplexity](#calculating-perplexity)
-   [Measuring topic coherence based on human interpretation](#measuring-topic-coherence-based-on-human-interpretation)
-   [Conclusion](#conclusion)

Introduction
============

Topic models such as LDA allow you to specify the number of topics in the model. On the one hand, this is a nice thing, because it allows you to adjust the granularity of what topics measure: between a few broad topics and many more specific topics. On the other hand, it begets the question what *the best number* of topics is.

The short and perhaps disapointing answer is that *the best number* of topics does not exist. After all, there is no singular idea of what *a topic* even is is. What a good topic is also depends on what you want to do. If you want to use topic modeling to interpret what a corpus is about, you want to have a limited number of topics that provide a good representation of overall themes. Alternatively, if you want to use topic modeling to get topic assignments per document without actually interpreting the individual topics (e.g., for document clustering, supervised machine l earning), you might be more interested in a model that fits the data as good as possible.

Still, even if the best number of topics does not exist, some values for k (i.e. the number of topics) are better than others. Use too few topics, and there will be variance in the data that is not accounted for, but use too many topics and you will overfit. So how can we at least determine what *a good number* of topics is?

In this document we discuss two general approaches. The first approach is to look at how well our model fits the data. As a probabilistic model, we can calculate the (log) likelihood of observing data (a corpus) given the model parameters (the distributions of a trained LDA model). For models with different settings for k, and different hyperparameters, we can then see which model best fits the data. The nice thing about this approach is that it's easy and free to compute. However, it still has the problem that no human interpretation is involved. The second approach does take this into account but is much more time consuming: we can develop tasks for people to do that can give us an idea of how coherent topics are in human interpretation.

Calculating perplexity
======================

The most common measure for how well a probabilistic topic model fits the data is **perplexity** (which is based on the log likelihood). The lower (!) the perplexity, the better the fit.

Let's first make a DTM to use in our example.

``` r
library(topicmodels)
library(quanteda)
texts = corpus_reshape(data_corpus_inaugural, to = "paragraphs")
dfm = dfm(texts, remove_punct=T, remove=stopwords("english"))
dfm = dfm_trim(dfm, min_docfreq = 10)
dtm = convert(dfm, to = "topicmodels") 
```

Now, to calculate perplexity, we'll first have to split up our data into data for training and testing the model. This way we prevent overfitting the model. Here we'll use 75% for training, and held-out the remaining 25% for test data.

``` r
train = sample(rownames(dtm), nrow(dtm) * .75)
dtm_train = dtm[rownames(dtm) %in% train, ]
dtm_test = dtm[!rownames(dtm) %in% train, ]
```

We can now get an indication of how 'good' a model is, by training it on the training data, and then testing how well the model fits the test data. Conveniently, the topicmodels packages has the `perplexity` function which makes this very easy to do.

First we train the model on dtm\_train.

``` r
m = LDA(dtm_train, method = "Gibbs", k = 5,  control = list(alpha = 0.01))
```

And then we calculate perplexity for dtm\_test

``` r
perplexity(m, dtm_test)
```

    ## [1] 692.3172

Now, a single perplexity score is not really usefull. What we want to do is to calculate the perplexity score for models with different parameters, to see how this affects the perplexity. Here we'll use a for loop to train a model with different topics, to see how this affects the perplexity score. Note that this might take a little while to compute.

``` r
## create a dataframe to store the perplexity scores for different values of k
p = data.frame(k = c(2,4,8,16,32,64,128), perplexity = NA)

## loop over the values of k in data.frame p 
for (i in 1:nrow(p)) {
  print(p$k[i])
  ## calculate perplexity for the given value of k
  m = LDA(dtm_train, method = "Gibbs", k = p$k[i],  control = list(alpha = 0.01))
  ## store result in our data.frame
  p$perplexity[i] = perplexity(m, dtm_test)
}
```

    ## [1] 2
    ## [1] 4
    ## [1] 8
    ## [1] 16
    ## [1] 32
    ## [1] 64
    ## [1] 128

Now we can plot the perplexity scores for different values of k.

``` r
library(ggplot2)
ggplot(p, aes(x=k, y=perplexity)) + geom_line()
```

![](img/perplexity_plot-1.png)

What we see here is that first the perplexity decreases as the number of topics increases. This makes sense, because the more topics we have, the more information we have. It is only between 64 and 128 topics that we see the perplexity rise again. If we would use smaller steps in k we could find the lowest point. If we repeat this several times for different models, and ideally also for different samples of train and test data, we could find a value for k of which we could argue that it is the *best* in terms of model fit.

Measuring topic coherence based on human interpretation
=======================================================

We already know that the number of topics k that optimizes model fit is not necessarily the best number of topics. After all, this depends on what the researcher wants to measure. But we might ask ourselves if it at least coincides with human interpretation of how coherent the topics are. In other words, whether using perplexity to determine the value of k gives us topic models that 'make sense'.

Alas, this is not really the case. In the paper "Reading tea leaves: How humans interpret topic models", Chang et al. ([2009](http://papers.nips.cc/paper/3700-reading-tea-leaves-how-humans-interpret-topic-models.pdf)) show that human evaluation of the coherence of topics based on the top words per topic, is not related to predictive perplexity.

They measured this by designing a simple task for humans. Given a topic model, the top 5 words per topic are extracted. Then, a sixth random word was added to act as the *intruder*. Human coders (they used crowd coding) were then asked to identify the intruder. If the topics are coherent (e.g., "cat", "dog", "fish", "hamster"), it should be obvious which word the intruder is ("airplane"). Thus, the extent to which the intruder is correctly identified can serve as a measure of coherence.

We can make a little game out of this. We first train a topic model with the full DTM.

``` r
m = LDA(dtm, method = "Gibbs", k = 10,  control = list(alpha = 0.01))
```

Now we get the top terms per topic. This can be done with the `terms` function from the `topicmodels` package. However, as these are simply the most likely terms per topic, the top terms often contain overall common terms, which makes the game a bit too much of a guessing task (which, in a sense, is fair). Here we therefore use a simple (though not very elegant) trick for penalizing terms that are likely across more topics.

``` r
tw = posterior(m)$terms    
tw = tw/colSums(tw)      
tw = apply(tw, 1, function(x) colnames(tw)[head(order(-x), 5)])  ## top 5 terms
```

Selecting terms this way makes the game a bit easier, so one might argue that its not entirely fair. However, you'll see that even now the game can be quite difficult! The following lines of code start the game.

``` r
for (i in 1:ncol(tw)) {
  real = tw[,i]
  intruder = sample(setdiff(colnames(dtm), real), 1)
  options = sample(c(real, intruder), 6)
  cat(paste(paste(1:6, options, sep=':  '), collapse='\n'))
  answer = readline(prompt="Which is the intruder? [1-6]  ")
  if (options[as.numeric(answer)] == intruder) 
    message("CORRECT!!\n") 
  else 
    message(sprintf('WRONG!! it was "%s"\n', intruder))
}
```

Note that this is not the same as validating whether a topic models measures what *you want* to measure. By using a simple task where humans evaluate coherence without receiving strict instructions on what a topic is, the 'unsupervised' part is kept intact.

Now, it is hardly feasible to use this approach yourself for every topic model that you want to use. More importantly, the paper tells us something about how we should be carefull to interpret what a topic means based on just the top words.

Conclusion
==========

There is no golden bullet. The choice for how many topics (k) is *best* comes down to what you want to use topic models for. Predictive validity, as measured with perplexity, is a good approach if you just want to use the document X topic matrix as input for an analysis (clustering, machine learning, etc.). If you want to use topic modeling as a tool for bottom-up (inductive) analysis of a corpus, it is still usefull to look at perplexity scores, but rather than going for the k that optimizes fit, you might want to look for a knee in the plot, similar to how you would choose the number of factors in a factor analysis. But more importantly, you'd need to make sure that how you (or your coders) interpret the topics is not just reading tea leaves.

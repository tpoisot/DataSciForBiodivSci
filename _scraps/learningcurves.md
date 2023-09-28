
## Classification based on probabilities

When first introducing classification, in **TODO CHAPTER**, we used a model that returned a deterministic answer, which is to say, the name of a class (in our case, this class was either "present" or "absent"). But a lot of classifiers return quantitative values, that correspond to (proxies for) the probability of the different classes. Nevertheless, because we are interesting in solving a classification problem, we need to end up with a confusion table, and so we need to turn a number into a class. In the context of binary classification (we model a yes/no variable), this can be done using a threshold for the probability.

The idea behind the use of thresholds is simple: if the classifier output $\hat y$ is larger than (or equal to) the threshold value $\tau$, we consider that this prediction corresponds to the positive class (the event we want to detect, for example the presence of a species). In the other case, this prediction corresponds to the negative class. Note that we do not, strictly, speaking, require that the value $\hat y$ returned by the classifier be a probability. We can simply decide to pick $\tau$ somewhere in the support of the distribution of $\hat y$.

The threshold to decide on a positive event is an hyper-parameter of the model. In the NBC we built in **TODO**, our decision rule was that $p(+) > p(-)$, which when all is said and done (but we will convince ourselves of this in **TODO**), means that we used $\tau = 0.5$. But there is no reason to assume that the threshold needs to be one half. Maybe the model is overly sensitive to negatives. Maybe there is a slight issue with our training data that bias the model predictions. And for this reason, we have to look for the optimal value of $\tau$.

There are two important values for the threshold, at which we know the behavior of our model. The first is $\tau = \text{min}(\hat y)$, for which the model *always* returns a negative answer; the second is, unsurprisingly, $\tau = \text{max}(\hat y)$, where the model *always* returns a positive answer. Thinking of this behavior in terms of the measures on the confusion matrix, as we have introduced them in **TODO**, the smallest possible threshold gives only negatives, and the largest possible one gives only positives: they respectively maximize the false negatives and false positives rates.

### The ROC curve

This is a behavior we can exploit, as increasing the threshold away from the minimum will lower the false negatives rate and increase the true positive rate, while decreasing the threshold away from the maximum will lower the false positives rate and increase the true negative rate. If we cross our fingers and knock on wood, there will be a point where the false events rates have decreased as much as possible, and the true events rates have increased as much as possible, and this corresponds to the optimal value of $\tau$ for our problem.

We have just described the Receiver Operating Characteristic (ROC; @fawcett2006) curve! The ROC curve visualizes the false positive rate on the $x$ axis, and the true positive rate on the $y$ axis. The area under the curve (the ROC-AUC) is a measure of the overall performance of the classifier [@hanley1982]; a model with ROC-AUC of 0.5 performs at random, and values moving away from 0.5 indicate better (close to 1) or worse (close to 0) performance.The ROC curve is a description of the model performance across all of the possible threshold values we investigated!

### The PR curve

One very common issue with ROC curves, is that they are overly optimistic about the performance of the model, especially when the problem we work on suffers from class imbalance, which happens when observations of the positive class are much rarer than observations of the negative class. In ecology, this is a common feature of data on species interactions [@poisot2023]. For this reason, it is always advised to ...

### Cross-entropy loss and other classification loss functions

## How to optimize the threshold?

In order to understand the optimization of the threshold, we first need to understand how a model with thresholding works. When we run such a model on multiple input features, it will return a list of probabilities, for example $[0.2, 0.8, 0.1, 0.5, 1.0]$. We then compare all of these values to an initial threshold, for example $\tau = 0.05$, giving us a vector of Boolean values, in this case $[+, +, +, +, +]$. We can then compare this classified output to a series of validation labels, *e.g.* $[-, +, -, -, +]$, and report the performance of our model. In this case, the very low thresholds means that we accept any probability as a positive case, and so our model is very strongly biased. We then increase the threshold, and start again.

As we have discussed in **TODO previous**, moving the threshold is essentially a way to move in the space of true/false rates. As the measures of classification performance capture information that is relevant in this space, there should be a value of the threshold that maximizes one of these measures. Alas, no one agrees on which measure this should be [@Perkins2006; @Unal2017]. The usual recommendation is to use the True Skill Statistic, also known as Youden's $J$ [@youden1950]. The biomedical literature, which is quite naturally interested in getting the interpretation of tests right, has established that maximizing this value brings us very close to the optimal threshold for a binary classifier [@perkins2005]. In a simulation study, using the True Skill Statistic gave good predictive performance for models of species interactions [@poisot2023a].

Some authors have used the MCC as a measure of optimality [@zhou2013], as it is maximized *only* when a classifier gets a good score for the basic rates of the confusion matrix. Based on this information, @chicco2023 recommend that MCC should be used to pick the optimal threshold *regardless of the question*, and I agree with their assessment. A high MCC is always associated to a high ROC-AUC, TSS, etc., but the opposite is not necessarily true. This is because the MCC can only reach high values when the model is good at *everything*, and therefore it is not possible to trick it. In fact, previous comparisons show that MCC even outperform measures of classification loss [@Jurman2012].

For once, and after over 15 years of methodological discussion, it appears that we have a conclusive answer! In order to pick the optimal threshold, we find the value that maximizes the MCC. Note that in previous chapters, we already used the MCC as a our criteria for the best model, and now you know why.

## The problem: building a probabilistic NBC model

## Application: improved reindeer distribution model

{{< embed ../notebooks/sm-moving-threshold.qmd#tbl-moving-confusion >}}

## The problem: reindeer distribution

Throughout these chapters, we will be working on a single problem, which is to predict the distribution of the Reindeer, *Rangifer tarandus tarandus*. Species Distribution Modeling (SDM; @elith2009), or Ecological Niche Modeling (ENM), is an excellent instance of ecologists doing applied machine learning already, as @beery2021 rightfully pointed out. In fact, the question of fitness-for-purpose, which we discussed in previous chapters (for example in @sec-crossvalidation-fitness), has been covered in the SDM literature [@guillera-arroita2015]. In these chapters, we will fully embrace this idea, and look at the problem of predicting where species can be as a data science problem.

Because this chapter is the first of a series, we will start by building a bare-bones model on ecological first principles. This is an important step. The rough outline of a model is often indicative of how difficult the process of training a really good model will be. But building a good model is an iterative process, and so we will start with a very simple model and training strategy, and refine it over time. In this chapter, the purpose is less to have a very good training process; it is to familiarize ourselves with the task of classification.

We will therefore start with a blanket assumption: the distribution of species is something we can predict based on temperature and precipitation. We know this to be important for plants [@clapham1935] and animals [@whittaker1962], to the point where the relationship between mean temperature and annual precipitation is how we find delimitations between biomes. If you need to train a lot of models on a lot of species, temperature and precipitation are not the worst place to start [@berteaux2014].

Consider our dataset for a minute. In order to predict the presence of a species, we need information about where the species has been observed; this we can get from the [Global Biodiversity Information Facility]. We need information about where the species has *not* been observed; this is usually not directly available, but there are ways to generate background points that are a good approximation of this [@hanberry2012; @barbet-massin2012]. All of these data points come in the form $(\text{lat.}, \text{lon.}, y)$, which give a position in space, as well as $y = \{+,-\}$ (the species is present or absent!) at this position.

  [Global Biodiversity Information Facility]: https://www.gbif.org/

To build a model with temperature and precipitation as inputs, we need to extract the temperature and precipitation at all of these coordinates. We will use the WorldClim2 dataset [@fick2017] for this purpose. In a great many situations, CHELSA2 [@karger2017] would be a better source of bioclimatic variables, but it has a much higher spatial resolution; what we want to do here is iterate rapidly and focus on the high-level discussion of how we make predictions, so a coarser data product will serve us just as well.

The predictive task we want to complete is to get a predicted presence or absence $\hat y = \{+,-\}$, from a vector $\mathbf{x}^\top = [\text{temp.} \quad \text{precip.}]$. This specific task is called classification, and we will now introduce some elements of theory.

## What is classification?

Classification is the prediction of a qualitative response. In @sec-clustering, for example, we predicted the class of a pixel, which is a qualitative variable with levels $\{1, 2, \dots, k\}$. This represented an instance of *unsupervised* learning, as we had no *a priori* notion of the correct class of the pixel. When building SDMs, by contrast, we often know where species are, and we can simulate "background points", that represent assumptions about where the species are not.

::: column-margin
When working on $\{+,-\}$ outcomes, we are specifically performing *binary* classification. Classification can be applied to more than two levels.
:::

In short, our response variable has levels $\{+, -\}$: the species is there, or it is not -- we will challenge this assumption later in the series of chapters, but for now, this will do. The case where the species is present is called the *positive class*, and the case where it is absent is the *negative class*. We tend to have really strong assumptions about classification already. For example, monitoring techniques using environmental DNA [*e.g.* @perl2022] are a classification problem: the species can be present or not, $y = \{+,-\}$, and the test can be positive of negative $\hat y = \{+,-\}$. We would be happy in this situation whenever $\hat y = y$, as it means that the test we use has diagnostic value. This is the essence of classification, and everything that follows is more precise ways to capture how close a test comes from this ideal scenario.

### Separability

A very important feature of the relationship between the features and the classes is that, broadly speaking, classification is much easier when the classes are separable. Separability (often linear separability) is achieved when, if looking at some projection of the data on two dimensions, you can draw a line that separates the classes (a point in a single dimension, a plane in three dimension, and so on and so forth). For reasons that will become clear in @sec-variable-selection-curse, simply adding more predictors is not necessarily the right thing to do.

In @fig-classification-separability, we can see the temperature (in degrees) for locations with recorded presences of reindeers, and for locations with assumed absences. These two classes are not quite linearly separable alongside this single dimension (maybe there is a different projection of the data that would change this; we will explore one in @sec-variable-selection), but there are still some values at which our guess for a class changes. For example, at a location with a temperature colder than 1°C, presences are far more likely. For a location with a temperature warmer than 5°C, absences become overwhelmingly more likely. The locations with a temperature between 0°C and 5°C can go either way.

{{< embed ../notebooks/sm-classification.qmd#fig-classification-separability >}}

### The confusion table

Evaluating the performance of a classifier (a classifier is a model that performs classification) is usually done by looking at its confusion table, which is a contingency table of the form

$$
\begin{pmatrix}
\text{TP} & \text{FP}\\
\text{FN} & \text{TN} 
\end{pmatrix} \,.
$$ {#eq-classification-confusion}

This can be stated as "counting the number of times each pair of prediction, observation occurs", like so:

$$
\begin{pmatrix}
|\hat +, +| & |\hat +, -|\\
|\hat -, +| & |\hat -, -| 
\end{pmatrix} \,.
$$ {#eq-classification-explain}

The four components of the confusion table are the true positives (TP; correct prediction of $+$), the true negatives (TN; correct prediction of $-$), the false positives (FP; incorrect prediction of $+$), and the false negatives (FN; incorrect prediction of $-$). Quite intuitively, we would like our classifier to return mostly elements in TP and TN: a good classifier has most elements on the diagonal, and off-diagonal elements as close to zero as possible.

As there are many different possible measures on this matrix, we will introduce them as we go. In this section, it it more important to understand how the matrix responds to two important features of the data and the model: balance and bias.

Balance refers to the proportion of the positive class. Whenever this balance is not equal to 1/2 (there are as many positives as negative cases), we are performing *imbalanced* classification, which comes with additional challenges; few ecological problems are balanced. There is a specific hypothetical classifier, called the *no-skill classifier*, which guesses classes at random as a function of their proportion. It turns out to have an interesting confusion matrix! If we note $b$ the proportion of positive classes, the no-skill classifier will guess $+$ with probability $b$, and $-$ with probability $1-b$. Because these are also the proportion in the data, we can write the adjacency matrix as

$$
\begin{pmatrix}
b^2 & b(1-b)\\
(1-b)b & (1-b)^2 
\end{pmatrix} \,.
$$ {#eq-classification-noskill}

The proportion of elements that are on the diagonal of this matrix is $b^2 + (1-b)^2$. When $b$ gets lower, this value actually increases: the more difficult a classification problem is, the more accurate random guesses *look like*. This has a simple explanation: if most of the cases are negative, and you predict a negative case often, you will by chance get a very high true negative score. For this reason, measures of model performance will combine the positions of the confusion table to avoid some of these artifacts.

Bias refers to the fact that a model can recommend more (or fewer) positive or negative classes than it should. An extreme example is the *zero-rate classifier*, which will always guess the most common class, and which is commonly used as a baseline for imbalanced classification. A good classifier has high skill (which we can measure by whether it beats the no-skill classifier for our specific problem) and low bias. In this chapter, we will explore different measures on the confusion table the inform us about these aspects of model performance, using the Naive Bayes Classifier.

## The Naive Bayes Classifier

The Naive Bayes Classifier (NBC) is my all-time favorite classifier. It is build on a very simple intuition, works with almost no data, and more importantly, often provides an annoyingly good baseline for other, more complex classifiers to meet. That NBC works at all is counter-intuitive [@hand2001]. It assumes that all variables are independent, it works when reducing the data to a simpler distribution, and although the numerical estimate of the class probability is remarkably unstable, it generally gives good predictions. NBC is the data science equivalent of saying "eh, I reckon it's probably *this* class" and somehow getting it right 95% of the case [there are, in fact, several papers questioning *why* NBC works so well; see *e.g.* @kupervasser2014].

### How the NBC works

In @fig-classification-separability, what is the most likely class if the temperature is 2°C? We can look at the density traces on top, and say that because the one for presences is higher, we would be justified in guessing that the species is present. Of course, this is equivalent to saying that $P(2^\circ C | +) > P(2^\circ C | -)$. It would appear that we are looking at the problem in the wrong way, because we are really interested in $P(+ | 2^\circ C)$, the probability that the species is present knowing that the temperature is 2°C.

Using Baye's theorem, we can re-write our goal as

$$
P(+|x) = \frac{P(+)}{P(x)}P(x|+) \,,
$$ {#eq-nbc-onevar}

where $x$ is one value of one feature, $P(x)$ is the probability of this observation (the evidence, in Bayesian parlance), and $P(+)$ is the probability of the positive class (in other words, the prior). So, this is where the "Bayes" part comes from.

But why is NBC naïve?

In @eq-nbc-onevar, we have used a single feature $x$, but the problem we want to solve uses a vector of features, $\mathbf{x}$. These features, statisticians will say, will have covariance, and a joint distribution, and many things that will challenge the simplicity of what we have written so far. These details, NBC says, are meaningless.

NBC is naïve because it makes the assumptions that the features are all independent. This is very important, as it means that $P(+|\mathbf{x}) \propto P(+)\prod_i P(\mathbf{x}_i|+)$ (by the chain rule). Note that this is not a strict equality,as we need to divide by the evidence. But the evidence is constant across all classes, and so we do not need to measure it to get an estimate of the score for a class.

To generalize our notation, the score for a class $\mathbf{c}_j$ is $P(\mathbf{c}_j)\prod_i P(\mathbf{x}_i|\mathbf{c}_j)$. In order to decide on a class, we apply the following rule:

$$
\hat y = \text{argmax}_j \, P(\mathbf{c}_j)\prod_i P(\mathbf{x}_i|\mathbf{c}_j) \,.
$$ {#eq-nbc-decision}

In other words, whichever class gives the higher score, is what the NBC will recommend for this instance $\mathbf{x}$. In @sec-tuning, we will improve upon this model by thinking about the evidence $P(\mathbf{x})$, but as you will see, this simple formulation will already prove frightfully effective.

### How the NBC learns

There are two unknown quantities at this point. The first is the value of $P(+)$ and $P(-)$. These are priors, and are presumably important to pick correctly. In the spirit of iterating rapidly on a model, we can use two starting points: either we assume that the classes have the same probability, or we assume that the representation of the classes (the balance of the problem) *is* their prior.

The most delicate problem is to figure out $P(x|c)$, the probability of the observation of the variable when the class is known. There are variants here that will depend on the type of data that is in $x$; as we work with continuous variables, we will rely on Gaussain NBC. In Gaussian NBC, we will consider that $x$ comes from a normal distribution $\mathcal{N}(\mu_{x,c},\sigma_{x,c})$, and therefore we simply need to evaluate the probability density function of this distribution at the point $x$. Other types of data are handled in the same way, with the difference that they use a different set of distributions.

Therefore, the learning stage of NBC is extremely quick: we take the mean and standard deviation of the values, split by predictor and by class, and these are the parameters of our classifier.

## Application: a baseline model of reindeer presence

### Training and validation strategy

### Performance evaluation of the model

{{< embed ../notebooks/sm-classification.qmd#fig-classification-crossvalidation >}}

### The decision boundary

Now that the model is trained, we can take a break in our discussion of its performance, and think about *why* it makes a specific classification in the first place. Because we are using a model with only two input features, we can generate a grid of variables, and the ask, for every point on this grid, the classification made by our trained model. This will reveal the regions in the space of parameters where the model will conclude that the species is present.

The output of this simulation is given in @fig-classification-decision. Of course, in a model with more features, we would need to adapt our visualisations, but because we only use two features here, this image actually gives us a complete understanding of the model decision process. Think of it this way: even if we lose the code of the model, we could use this figure to classify any input made of a temperature and a precipitation, and read what the model decision would have been.

{{< embed ../notebooks/sm-classification.qmd#fig-classification-decision >}}

The line that separates the two classes is usually refered to as the "decision boundary" of the classifier: crossing this line by moving in the space of features will lead the model to predict another class at the output. In this instance, as a consequence of the choice of models and of the distribution of presence and absences in the environmental space, the decision boundary is not linear.

It is interesting to compare @fig-classification-decision with, for example, the distribution of the raw data presented in @fig-classification-separability. Although we initially observed that temperature was giving us the best chance to separate the two classes, the shape of the decision boundary suggests that our classifier is considering that reindeers enjoy cold and dry climates.

### What is an acceptable model?
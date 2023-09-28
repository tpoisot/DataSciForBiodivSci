
## The curse of dimensionality {#sec-variable-selection-curse}

It would be tempting to say that adding dimensions should improve our chances to find a feature alongside which the classes become linearly separable. If only!

The "curse of dimensionality" is the common term of everything breaking down when the dimensions of a problem increase. In our perspective, where we rely on the resemblance between features to make a prediction, increasing the dimensions of a problem means adding features, and it has important consequences on the distance between observations. Picture two points positioned at random on the unit interval: the average distance between them is 1/3. If we add one dimension, keeping two points but turning this line into a cube, the average distance would be about 1/2. For a cube, about 2/3. For $n$ dimensions, we can figure out that the average distance grows like $\sqrt{n/6 + c}$, which is to say that when we add more dimensions, we make the average distance between two points go to infinity. To ass

Ecological studies are not immune to this [e.g. @smith2017]!

we will use MCC [@chicco2020] rather than $F_1$ or accuracy to decide on the best model

is the Pearson product-moment correlation on a contingency table [@powers2020]
## Using lasso regressions and multilayer neural networks to predict NBA player statistics

<img src="images/jokic.png?raw=true/">

This portfolio is largely intended to showcase the power of data science and analytics to drive human rights investigations. That said, those who know me will know that my one of my biggest passions (obsessions?) is NBA basketball, and I've recently been enjoying using machine learning methods to play around with NBA statistics. This is a tidy example of how methods like lasso regression and neural networks can serve as powerful predictive models when fed large, robust datasets, so I wanted to post my results.

In this project, I took 30 years of historical NBA player data and built a series of machine learning models to predict how all non-rookie players would perform in the 2021-2022 NBA season, which was ongoing at the time of the writing of this post.

### The raw data

The data for this project was scraped from [Basketball Reference](https://www.basketball-reference.com/), a leading online repository of NBA statistics. Compiling 30 years of data into one dataframe was a large task that involved quite a bit of code that I will not include here, but the end result is a data set that looks like the below, which I have already scaled (only a few of the 60+ columns are visible below):

<img src="images/nba_data_screenshot.png?raw=true/">

Each row of this dataframe corresponds to an an individual "player season." For example, there is one row for Lebron James's 2011-2012 season, a separate row for his 2010-2011 season, and so on, for every player that has played in the NBA since 1990. The columns in the data set represent a variety of different demographic and statistical categories. The demographic categories include thing like age and player position. The statistical categories include both what are called basic "counting stats" (things like points, rebounds, assists, etc.), as well as "advanced stats", which are statistical categories invented by both amateur and professional NBA analysts which amalgamate counting stats into purportedly more descriptive new statistical parameters. Advanced stats include things like true shooting percentage, three-point attempt rate, and assist percentage, for example.

There are two important things to note about this data. The first relates to the response variables I am attempting to predict. For each player in the 2021-2022 season, these models aim to predict:

- Total points
- Total 3-pointers made
- Total field goals attempted
- Total field goals made
- Total free throws attempted
- Total free throws made
- Total rebounds
- Total assists
- Total steals
- Total blocks

Accordingly, these statistics are the final ten categories of the dataframe. However, it should be noted that in each row, the response columns correspond to **one season later** than the rest of the columns in that row. For example, for the row corresponding to Kevin Durant's 2015-2016 season, the last ten columns of that row (total points, total 3-pointers made, total field goals attempted, etc.) correspond to Kevin Durant's 2016-2017 season. The data must be set up this way in order to predict future performance.

The second note about this data is that I have included a column called 't'. This column is a continuous numerical variable that takes a larger value based on the recency of the value in the "season column". So a row corresponding to the most recent year represented in the data has 30 in the 't' column, a row corresponding to the second-most recent year represented in the data has 29 in the 't' column, and so on. Doing this is an easy (if relatively imprecise) way to weight data so that more recent data is given more predictive weight by machine learning algorithms. There are more sophisticated ways to do this (exponential smoothing or recurrent neural networks, for example), but this is the method I chose to implement within the context of the lasso.

### Lasso regression

The lasso is a handy tool to use in a case like this where we have many predictor variables that may be significant, and no clear way to choose which to include in our model. Broadly, the lasso does this for us by imposing a penalty term on the regression that shrinks to zero the coefficients of variables that the model deems to be less important. Below, I outline my approach to predicting total points for each player; I repeated this process ten times, building one model for each statistical category I wanted to predict.

The first step was to split the data set into training and testing data. I chose to use the data from the 2015/2016 - 2018/2019 seasons as the training data and the data from the 2019/2020 season as the testing data. This approach was chosen such that the 2019/2020 comprises 30% of the total data, and then I selected enough seasons for the training data such that they cumulatively comprised 70% of the total. Note that I have read in the scaled data from a .csv as a dataframe called "nba_scaled_names":

```javascript
library(tidyverse)
library(glmnet)

nba_train_w_names = subset(nba_scaled_names, t > 24 & t < 29)
nba_train <-  select(nba_train_w_names, -c(name, season))

nba_test_w_names = subset(nba_scaled_names, t == 29)
nba_test <-  select(nba_test_w_names, -c(name, season))
```
Next, I built x_train, y_train, x_test, and y_test objects for use with the lasso model. The glmnet() function, which I use below, requires matrices, not data frames. The model.matrix() function is helpful for this, as it creates a matrix corresponding to the predictors and also automatically transforms any qualitative variables into dummy variables. The latter property is important because glmnet() can also only take numerical, quantitative inputs:
```javascript
x_train <- model.matrix(tot_pts ~ ., nba_train)[, -1]
y_train <- nba_train$tot_pts
x_test <- model.matrix(tot_pts ~. , nba_test)[, -1]
y_test <- nba_test$tot_pts
```
The next step was to use a grid-search cross-validation process to find the optimal value for lambda, which is the parameter that the lasso will use to penalize the regression and shrink coefficients. By default the glmnet() function, which I use below, performs lasso regression for an automatically selected range of lambda values. However, here I chose to implement the function over a grid of values essentially covering the full range of scenarios from the null model containing only the intercept, to the least squares fit:
```javascript
grid <- 10^seq(10, -2, length = 100)

cv_out <- cv.glmnet(x_train, y_train, alpha = 1, lambda = grid)
best_lam <- cv_out$lambda.min
```
Next, I built the lasso model using glmnet():
```javascript
lasso_reg_nba <- glmnet(x_train, y_train, alpha = 0, family = "gaussian", lambda = best_lam)
```
From there, I used the model I fit on the training data to make predictions on the test data, and calculated the test mean squared error and test R-squared:
```javascript
lasso_pred_nba <- predict(lasso_reg_nba, s = best_lam, newx = x_test)
test_MSE <- mean((lasso_pred_nba - y_test)^2)
SSE <- sum((lasso_pred_nba - y_test)^2)
SST <- sum((y_test - mean(y_test))^2)
R_square <- 1 - SSE / SST
```
This results in a test R-squared of 0.856.
The final step is to use the lasso model to make predictions based on data from the most recent NBA season, which is the 2019-2020 season. These are the rows for which the value in the t column in the data set == 30. I scaled this data and stored it in a dataframe called t30_no_names. First, I created a matrix object for use with the lasso regression, and then I made predictions on that data:
```javascript
x_t30 <- model.matrix(tot_pts ~ ., t30_scaled_no_names)[, -1]
predict(lasso_reg_nba, s = best_lam, newx = x_t30)
```
When sorted in descending order, my lasso model predicted the following top 12 scorers in the NBA in the 2021-2022 season:

<img src="images/lasso_pts_leaders_tot.png?raw=true/">

These results actually make a good deal of sense, and are largely consonant with 2020-2021's leading scorers and the leading scorers through half of the 2021-2022 season. That said, there are some glaring omissions and questionable inclusions, which I discuss later in this post.

### Multilayer neural networks with hypertuning

The other machine learning method I tested on this data was a multilayer neural network. The appeal there is clear, as neural networks offer advanced predictive capabilities. The drawback is that a neural network is a black box model--while it may give us accurate predictions, we cannot learn much about how individual predictor variables contributed to the outcome. I discuss this below in my comparison of the lasso and neural network models.

Building a neural network model in R is somewhat complicated. R does have the neuralnet() package, but for more advanced construction and hypertuning of neural networks, the keras platform for python is preferable. Luckily, through the Anaconda platform we can port keras, python, and tensorflow into R. I found instructions for how to do this on Stack Overflow [here](https://stackoverflow.com/questions/49684605/how-to-install-package-keras-in-r).

The data preparation for the neural network was the same as for the lasso. The first step in building the neural network model and hypertuning its parameters is to lay out the structure of the model itself. First, I built an object called "FLAGS" that I will use later when I call on the model to iterate over different values for its hyperparameters:
```javascript
[//]:#devtools::install_github("rstudio/keras") 
library(keras)
library(reticulate)
use_python("C:/Users/benpi/anaconda3/envs/r-tensorflow/python.exe")
[//]:#install.packages("ISLR2")
library(ISLR2)
[//]:#install.packages("tidyverse")
library(tidyverse)
[//]:#install.packages("ggplot2")
library(ggplot2)
[//]:#install.packages("caTools")
library(caTools)
[//]:#install.packages("tfruns")
library(tfruns)

FLAGS <- flags(
  flag_integer('neurons1', 32),
  flag_numeric('dropout1', 0.2),
  flag_integer('neurons2', 16),
  flag_numeric('dropout2', 0.2),
  flag_numeric('lr', 0.001)
)
```
Next, I built a function called build_model that calls the keras_model_sequential() function and  builds the structure of the neural network. This model has two hidden layers, each with a dropout layer, and an output layer. These layers contain the various hyperparameters that I will tune.
```javascript
build_model <- function() {
  nn_mod <- keras_model_sequential() 
  nn_mod %>% 
    
    layer_dense(units = FLAGS$neurons1,
                kernel_initializer = "uniform",
                activation = "relu",
                input_shape = ncol(x)) %>%
    
    layer_dropout(rate = FLAGS$dropout1) %>%
    
    layer_dense(units = FLAGS$neurons2, activation = "relu") %>%
    
    layer_dropout(rate = FLAGS$dropout2) %>%
    
    layer_dense(units = 1,
                kernel_initializer = "uniform",
                activation = "linear")
  
  nn_mod %>% compile(
    loss = "mse",
    optimizer = optimizer_adam(lr = FLAGS$lr),
    metrics = list('mean_squared_error')
  )
  nn_mod
}
```
Then I call the new build_model() function and specify an early stopping mechanism:
```javascript
nn_mod <- build_model()
early_stop <- callback_early_stopping(monitor = "val_loss", patience = 20)
```
Next I call the fit() function to fit the model on the training data:
```javascript
history <- nn_mod %>% fit(
  x_train,
  y_train,
  epochs = 100,
  batch_size = 32,
  validation_split = 0.2,
  verbose = 1,
  callbacks = list(early_stop)
)
```
Finally, I plot the model, evaluate its performance, and save it as a .h5 file:
```javascript
plot(history)
score <- nn_mod %>% evaluate(
  x_test, y_test,
  verbose = 0
)
save_model_hdf5(nn_mod, 'nn_mod.h5')
cat('Test loss:', score$loss, '\n')
cat('Test accuracy:', score$mean_absolute_error, '\n')
```
In order to tune the hyperparameters of this model, I save all of the above code as its own .R file, titled "nn_ht_mult_reg_nba.R", which I then called in the code below. The next step was to build a list object called par that contains the series of values that I want the model to consider for its hyperparemeters:
```javascript
par <- list(
  dropout1 = c(0.2,0.3,0.4),
  dropout2 = c(0.2,0.3,0.4),
  neurons1 = c(32,64,128,256),
  neurons2 = c(16,32,64,128),
  lr = c(0.001,0.01,0.1)
)
```
Then I call the tuning_run function from the tf_runs library and feed it my nn_ht_mult_reg_nba.R file, which generates and compiles the model and then fits it on the training data. I also provided the par object to the flags argument. I note here that since I specified 3 values for dropout1, 3 for dropout2, 4 for neurons1, 4 for neurons2, and 3 for lr, there are 432 possible combinations of the tuning parameters (3x3x4x4x3). Iterating over that many combinations would take my laptop hours, so i have passed 0.1 to the "sample" argument, indicating that I want the model to sample only 10% of the possible hyperparameter combinations for a total of 44 runs.
```javascript
runs <- tuning_run("C:/Users/benpi/OneDrive/Documents/Coding/R projects/nn_ht_mult_reg_nba.R",
                   runs_dir = '_nba_reg_mult_tuning', sample = 0.1, flags = par)
```
Once all 44 runs are finished, I find the run with the lowest validation mean squared error and save that as best_run
```javascript
best_run <- ls_runs(order = metric_val_mean_squared_error, decreasing= T, runs_dir = '_nba_reg_mult_tuning')[1,]
```
Then, I call the training_run() function, to which I feed the nn_ht_mult_reg_nba.R file, containing the structure of the neural network model and the code to fit it on the training data. I also specify that training_run() should use the best hyperparameters identified by tuning_run:
```javascript
run <- training_run('nn_ht_mult_reg_nba.R',flags = list(
  dropout1 = best_run$flag_dropout1,
  dropout2 = best_run$flag_dropout2,
  neurons1 = best_run$flag_neurons1,
  neurons1 = best_run$flag_neurons2,
  lr = best_run$flag_lr))
```
Once training_run() completes I load the nn_mod.h5 file, which has been saved by training_run using the ideal hyperparameters identified by tuning_run(), and save it as best_model:
```javascript
best_model <- load_model_hdf5('nn_mod.h5')
```
Now I can take the model that was fit on the training data and use it to make predictions on the test data, as well as assess test MSE and test R_square:
```javascript
nn_ht_pts_pred <- predict(best_model, x_test)
test_MSE <- mean((y_test - nn_ht_pts_pred)^2)
SSE <- sum((nn_ht_pts_pred - y_test)^2)
SST <- sum((y_test - mean(y_test))^2)
R_square <- 1 - (SSE / SST)
```
Finally, I take the data corresponding to the 2020-2021 season (where the value in the 't' column == 30) and use my neural network model to make predictions based on that data for the 2021-2022 season. The 2020-2021 data is held in a dataframe called "x_t30":
```javascript
x_t30 <- model.matrix(tot_pts ~., t30_scaled_no_names)[, -1]
nn_ht_pts_pred_2021_2022 <- predict(best_model, x_t30)
```
When sorted in descending order, my neural network model predicted the following top 12 scorers in the NBA in the 2021-2022 season. Again, these broadly make sense and are consistent with the previous year's top performers, as well as the top scorers halfway through the 2021-2022 season, to some degree:

<img src="images/nn_pts_leaders_tot.png?raw=true/">

### Comparing and assessing the results

In this case, the results of the neural network and lasso regression were very similar, both in the players they identified as the likely top scorers and in their predictive accuracy. As you can see below, the top 12 for both models were the same individuals in different orders:

<img src="images/comp_results.png?raw=true/">

With this in mind, the lasso is a preferable model in this case because it offers broadly equivalent predictive accuracy to the neural net, but it is much more interpretable. In fact, we can examine the coefficients resulting from the lasso regression and see which variables the model believed to be more important, and which it zeroed out. Many are not surprising--total points scored, field goals made, and field goals attempted were highly predictive for a player's scoring in the subsequent season. But there are also some interesting insights here. For example, the number of turnovers a player committed was positively correlated with the next season's scoring, likely because players who commit lots of turnovers tend to be players who have the ball in their hands for much of the game, and thus take a lot of shots:

<img src="images/coef_table.png?raw=true/">

Overall, I think the results here are probably good not great. There are a few issues. One, which couldn't really be avoided, is the inclusion of highly ranked players who in reality have missed much or all of the 2021-2022 season so far due to injury, such as Zion Williamson and Damian Lillard. The models of course do not know not to include those players, and thus rank them highly.

There are a couple of larger issues here, though, both of which have to do with recency bias. One is the exclusion of certain players, namely Kevin Durant. Durant may be the best basketball player in the world, but he played no games in the 2019-2020 season and only 35 games in the 2020-2021 season, both due to injury. Based on that, the models have ranked him quite low for 2021-2022. That indicates to me that both the lasso and the neural network are exhibiting considerable recency bias. Similarly, both models rank Julius Randle very highly. Randle had a fantastic 2020-2021 season, but he had never performed nearly that well in the past. Based on his strong 2020-2021 season, the models have ranked him higher than he should be (perhaps evidenced by the fact that Randle is really struggling so far in the 2021-2022 season). This recency bias largely derives from the inclusion of the 't' variable, which was intentional--it is true that on balance, data from one season ago will have more predictive power than any other season for the subsequent season--Lebron James's statistics from last year are a much better indication of what he will do next year than his stats from 2015 are. Clearly, though, there are drawbacks to this approach, as seen in the Durant and Randle examples. as above, a recurrent neural network or an exponential smoothing approach may perform better in this regard; I may try these in the future.

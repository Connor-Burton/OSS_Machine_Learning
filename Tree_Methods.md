Stroke Prediction
================
Connor Burton
6/2/2022

Machine learning can sound like an intimidating approach to data
analysis. However, the step by step process described below is designed
to demystify this very powerful tool in the modern data scientist’s
arsenal. This project will utilize the R programming and a few
specialized packages to predict instances of stroke using publically
available Kaggle data. Data is available
[here](https://www.kaggle.com/datasets/fedesoriano/stroke-prediction-dataset?select=healthcare-dataset-stroke-data.csv).
This data includes a range of potential prediction variables. In total,
the data contains:

``` r
library(tidyverse)
```

    ## ── Attaching packages ─────────────────────────────────────── tidyverse 1.3.1 ──

    ## ✔ ggplot2 3.3.6     ✔ purrr   0.3.4
    ## ✔ tibble  3.1.7     ✔ dplyr   1.0.8
    ## ✔ tidyr   1.2.0     ✔ stringr 1.4.0
    ## ✔ readr   2.1.2     ✔ forcats 0.5.1

    ## ── Conflicts ────────────────────────────────────────── tidyverse_conflicts() ──
    ## ✖ dplyr::filter() masks stats::filter()
    ## ✖ dplyr::lag()    masks stats::lag()

``` r
tibble(col_desc)
```

    ## # A tibble: 12 × 2
    ##    columns            description                                               
    ##    <chr>              <chr>                                                     
    ##  1 id:                Unique identifier for each patient                        
    ##  2 gender:            The gender of the observation                             
    ##  3 age:               The age at time of measurement                            
    ##  4 hypertension:      Binary indicator for presence of hypertension             
    ##  5 heart_disease:     Binary indicator for presence of heart disease            
    ##  6 ever_married:      Binary indicator for if the observation had married by ti…
    ##  7 work_type:         Category of work, such as 'Govt_job' or 'private'         
    ##  8 Residence_type:    If the observation lived in a rural or urban area         
    ##  9 avg_glucose_level: The average level of glucose in the the blood of the pati…
    ## 10 bmi:               Body mass index                                           
    ## 11 smoking_status:    Categories of smoking behavior, such as 'smokes' or 'neve…
    ## 12 stroke:            Binary indicator of stroke happenstance

## 1. Data Input and Visualization

The first step to a machine learning project is to load up the necessary
R packages, pull in the data from whatever source you are utilizing
(locally in this case), and visualize the data to determine the
distribution of the outcome variable (primarily).

``` r
#Loading packages

library(tidymodels)
library(rpart)
library(rpart.plot)
library(parsnip)
library(ranger)
library(xgboost)
library(vip)

#Read in the data:
df =read.csv("healthcare-dataset-stroke-data.csv")

#clean the column names for consistent reference (I prefer the "snake" column name method):
names(df)[8] = "residence_type"


#Visualizing outcome distribution
ggplot(df) +
  geom_bar(aes(x=stroke)) +
  labs(title = "Counts of Stroke Instances (1 = stroke instance)",
       y = "Count",
       x = "Stroke Observation") +
  theme_classic()
```

![](Tree_Methods_files/figure-gfm/unnamed-chunk-3-1.png)<!-- -->

This graph depicts a key concept in machine learning– namely the null
classifier. This term describes the “score to beat” in that we are
seeking an algorithm that can predict better than the modal case of the
outcome. In this case, the lack of a stroke represents \~4900
observations or 95.13% of the data. This is the barometer for model
success, as we will utilize the accuracy of our models to describe their
performance.

## 2. Model Tuning:

Machine learning requires that we split our data into both a training
data set and a testing set. This means that our models will be trained
on a subset of the data, and then their performance will be determined
by how accurately they can predict *out of sample*. In order to maintain
consistent results across R sessions, we first set our seed, then split
the data randomly using a randomization function. This process prevents
*overfitting* our data on the full sample, which runs a high risk of
poor predictive power outside of sample. Below, outcome distributions of
the training and testing datasets are provided to ensure that
randomization does not result in incomparable samples.

### 2.1 Data splitting:

``` r
##cleaing
#subsequent models require that the outcome be coded as a factor, rather than a numeric outcome
df$stroke = as.factor(df$stroke)
#code bmi as numeric
df$bmi = as.numeric(df$bmi)
#remove single observation with "other" value for gender to avoid spurious correlation
df = df |> filter(gender %in% c("Male", "Female"))

#Setting Randomization Seed
set.seed(123456789)

#Subsetting 80% of the data into a training set
df_train =sort(sample(nrow(df), nrow(df)*.8))
train=df[df_train,]

#defining testing data as all observations that are not found in the training set
test=df[-df_train,]

#Visualizing outcome distribution in training data
ggplot(train) +
  geom_bar(aes(x=stroke)) +
  labs(title = "Counts of Stroke Instances (1 = stroke instance)",
       y = "Count",
       x = "Stroke Observation") +
  theme_classic()
```

![](Tree_Methods_files/figure-gfm/unnamed-chunk-4-1.png)<!-- -->

``` r
#Visuzalizing outcome distribution in testing data
ggplot(test) +
  geom_bar(aes(x=stroke)) +
  labs(title = "Counts of Stroke Instances (1 = stroke instance)",
       y = "Count",
       x = "Stroke Observation") +
  theme_classic()
```

![](Tree_Methods_files/figure-gfm/unnamed-chunk-4-2.png)<!-- -->

We will be utilizing the TidyModels galaxy of machine learning
processes. While this package is not necessarily the fastest way to
train ML models, it provides a consistent syntax for dealing with
several different algorithms. To begin, we will provide R with a method
for splitting the training data into 5 different validation “sets”. This
method is known as 5-fold cross validation and it prevents the models
from overfitting on the training set by validating four folds on a
“held-out” fifth set. While this is computationally expensive, it can be
thought of as a series of mini test sets that prevent overfitting in the
model tuning process.

``` r
#Splits for cross validation
df_cv = train |> vfold_cv(v = 5)
```

TidyModels generally follows a *Recipe* -> *Model Definition* ->
*Workflow* -> *Tuning* process. These processes vary slightly by
algorithms but their distinctions will be clear as we proceed. Each step
must be assigned to an R object and I suggest a consistent naming scheme
to simplify your R global environmnent.

#### Recipe:

Broadly, the ‘recipe’ process can be thought of as telling R how you
intend the model to relate the variables, as well as which variables you
which to exclude –which should always include any unique identifier
variables– and if you want to normalize numeric variables to avoid
issues with different denominations of the same phenomenon (such as
meters and kilometers) from being treated differently by the algorthim,
which is not a present concern in our dataset.

#### Model Definition:

The model definition process allows R to call on any necessary secondary
packages (“set_engine”) as well as to provide which hyperparameters you
want to allow to vary. Note: the more hyperparameters you tune and the
broader set of values you provide them, the longer your tuning process
will take. Hyperparameters available for tuning will vary based on the
model of choice.

#### Workflow:

The workflow step merely provides TidyModels with which model and recipe
you wish the algorthim to use in model training.

#### Tuning:

This step is where everything is combined and the model is trained on
the split data. This step allows hyperparameters to vary according to
which hyperparameters you allowed to vary in the model definition step.
The model is also trained each on the five folds defined above. This
step is by far the most computationally-intensive so I’d plan a tea
break before running it.

### 2.2 Tuning a Decision Tree

Decision trees are quick and simple way to train a nonlinear model on a
prediction problem, and function by splitting the data by asking
different binary questions in “branches” that describe their binary
outcome (stroke).

``` r
#Defining decision tree recipe
rec_tree = recipe(stroke ~ ., data=train) |> #use all variables to predict stroke
  step_rm(id)                                #remove unique identifiers


#Defining decision tree model
model_tree = decision_tree(   #this function comes from the parsnip package
  mode = "classification",    #specify that we are dealing with a classification problem, not numeric or otherwise
  cost_complexity = tune(),   #allow the tuning process to vary the cost function associated with the tree
  tree_depth = tune(),        #allow the tuning process to vary the depth of the tree
  min_n = 10) |>              #do not pare down the data past 10 observations regardless of tree depth
  set_engine("rpart")         #utilize the rpart package to drive this process

#Defining decision tree workflow
workflow_tree =
  workflow() |>
  add_model(model_tree) |>
  add_recipe(rec_tree)

#CV Decision tree with tuning
cv_tree =
  workflow_tree |>
  tune_grid(                  #provide the tunning process with the five folds, as well as which
    df_cv,                    #feed the method of cross validation, or a list of splits
    grid = expand_grid(
      cost_complexity = seq(0.01,0.04, by = 0.001),
      tree_depth = c(10,20,30)),
    metrics = metric_set(accuracy))

#Extract the best-performing models and their hyperparameters using accuracy as the performance metric
best_tree = cv_tree |> show_best(metric="accuracy")
best_tree
```

    ## # A tibble: 5 × 8
    ##   cost_complexity tree_depth .metric  .estimator  mean     n std_err .config    
    ##             <dbl>      <dbl> <chr>    <chr>      <dbl> <int>   <dbl> <chr>      
    ## 1           0.011         10 accuracy binary     0.950     5 0.00444 Preprocess…
    ## 2           0.011         20 accuracy binary     0.950     5 0.00444 Preprocess…
    ## 3           0.011         30 accuracy binary     0.950     5 0.00444 Preprocess…
    ## 4           0.012         10 accuracy binary     0.950     5 0.00444 Preprocess…
    ## 5           0.012         20 accuracy binary     0.950     5 0.00444 Preprocess…

We can see here that each of the models slightly under-performs the null
classifier of 95.13% accuracy. Let’s see if stronger models can do
better.

## 2.3 Tuning a Random Forest:

Random forests are a method for combining multiple decision trees into a
more flexible model with an added degree of randomness to reduce the
reliance on a single decision tree.

``` r
#Defining random forest recipe
rec_randomforest = recipe(stroke ~ ., data=train) |>
  step_rm(id) |>
  step_dummy(gender, ever_married, work_type, residence_type, smoking_status) |>   #create dummies for categorical variables
  step_impute_mean(bmi) #impute missing data with the mean

#Defining random forest model
model_randomforest = rand_forest(
  mode = "classification",
  mtry = tune(),
  trees= tune(),
  min_n = tune()) |>
  set_engine(engine = "ranger",  splitrule = "gini")


#Defining random forest workflow
workflow_randomforest =
  workflow() |>
  add_model(model_randomforest) |>
  add_recipe(rec_randomforest)


#CV random forest with tuning
cv_randomforest =
  workflow_randomforest |>
  tune_grid(
    df_cv,
    grid = expand_grid(
      mtry = c(2, 4),
      trees = c(50, 100, 150),
      min_n = c(5, 10, 15)),
    metrics = metric_set(accuracy))


#Extract the best-performing models and their hyperparameters using accuracy as the performance metric
best_randomforest = cv_randomforest |> show_best(metric="accuracy")
best_randomforest
```

    ## # A tibble: 5 × 9
    ##    mtry trees min_n .metric  .estimator  mean     n std_err .config             
    ##   <dbl> <dbl> <dbl> <chr>    <chr>      <dbl> <int>   <dbl> <chr>               
    ## 1     2    50     5 accuracy binary     0.950     5 0.00444 Preprocessor1_Model…
    ## 2     2    50    10 accuracy binary     0.950     5 0.00444 Preprocessor1_Model…
    ## 3     2    50    15 accuracy binary     0.950     5 0.00444 Preprocessor1_Model…
    ## 4     2   100     5 accuracy binary     0.950     5 0.00444 Preprocessor1_Model…
    ## 5     2   100    10 accuracy binary     0.950     5 0.00444 Preprocessor1_Model…

We can again see that each of the best models under-performs the null
classifier of 95.13% accuracy. Let’s move on to boosted trees.

### 2.4 Tuning a Boosted Tree

Boosted trees are an improvement on random forests in that they utilize
the residuals from the predictions of each tree to subsequently train
the proceeding tree, with a discount hyperparameter to reduce the size
of the residuals of that they are trained on to avoid overfitting and
repeated trees.

``` r
#Defining boosted tree recipe
rec_boosted = recipe(stroke ~ ., data=train) |>
  step_rm(id) |>
  step_dummy(gender, ever_married, work_type, residence_type, smoking_status) |>
  step_impute_mean(bmi)

#Defining boosted tree model
model_boosted =boost_tree(
  mode = "classification",
  mtry = tune(),
  trees= 300,
  min_n = tune(),
  tree_depth = tune(),
  learn_rate = tune()) |>
  set_engine(engine = "xgboost") #utilize the xgboost package to run the algorithm

#Defining boosted tree workflow
workflow_boosted =
  workflow() |>
  add_model(model_boosted) |>
  add_recipe(rec_boosted) # we can use the same recipe as specified in the RF model

#CV boosted tree with tuning
cv_boosted =
  workflow_boosted |>
  tune_grid(
    df_cv,
    grid = expand_grid(
      mtry = c(3, 6),
      min_n = c(15, 25),
      tree_depth = c(30, 50, 100),
      learn_rate = c(0.05, 0.025, 0.01)),
    metrics = metric_set(accuracy))

#Extract the best-performing models and their hyperparameters using accuracy as the performance metric
best_boosted = cv_boosted |> show_best(metric="accuracy")
best_boosted
```

    ## # A tibble: 5 × 10
    ##    mtry min_n tree_depth learn_rate .metric  .estimator  mean     n std_err
    ##   <dbl> <dbl>      <dbl>      <dbl> <chr>    <chr>      <dbl> <int>   <dbl>
    ## 1     3    25         30       0.05 accuracy binary     0.950     5 0.00425
    ## 2     6    25        100       0.05 accuracy binary     0.950     5 0.00427
    ## 3     6    15         30       0.05 accuracy binary     0.950     5 0.00405
    ## 4     6    25         30       0.05 accuracy binary     0.950     5 0.00440
    ## 5     6    25         50       0.05 accuracy binary     0.950     5 0.00440
    ## # … with 1 more variable: .config <chr>

## 4. Prediction: Fitting on Testing Data

The following subsections extract the accuracy of each ML model in
predicting the testing data, with accuracy utilized as a performance
metric.

### 4.1 Decision Tree:

``` r
#Running best workflow on new data
test_wf_tree =
  workflow_tree |>
  finalize_workflow(select_best(cv_tree, metric='accuracy')) |>
  fit(data=test)

#Predicting new data with trained model
test_tree_acc=predict(test_wf_tree, test)
test_tree_out =cbind(test, test_tree_acc)
test_tree_out$stroke = test_tree_out$stroke |> as.factor()
test_tree_out$.pred =test_tree_out$.pred |> as.factor()

#Decision Tree Test Accuracy:
Decision_Tree = test_tree_out |> accuracy(stroke, .pred)
```

### 4.2 Random Forest:

``` r
#Running best workflow on new data
test_wf_randomforest =
  workflow_randomforest |>
  finalize_workflow(select_best(cv_randomforest, metric='accuracy')) |>
  fit(data=test)

#Predicting new data with trained model
test_randomforest_acc = predict(test_wf_randomforest, test)
test_randomforest_out = cbind(test, test_randomforest_acc)
test_randomforest_out$stroke =test_randomforest_out$stroke |> as.factor()
test_randomforest_out$.pred =test_randomforest_out$.pred  |> as.factor()

#Random Forest Test Accuracy:
Random_Forest = test_randomforest_out |> accuracy(stroke, .pred)
```

### 4.3 Boosted Tree:

``` r
#Running best workflow on new data
test_wf_boosted =
  workflow_boosted |>
  finalize_workflow(select_best(cv_boosted, metric='accuracy')) |>
  fit(data=test)
```

    ## [01:11:31] WARNING: amalgamation/../src/learner.cc:1115: Starting in XGBoost 1.3.0, the default evaluation metric used with the objective 'binary:logistic' was changed from 'error' to 'logloss'. Explicitly set eval_metric if you'd like to restore the old behavior.

``` r
#Predicting new data with trained model
test_boosted_acc=predict(test_wf_boosted, test)
test_boosted_out =cbind(test, test_boosted_acc)
test_boosted_out$stroke = test_boosted_out$stroke |> as.factor()
test_boosted_out$.pred =test_boosted_out$.pred |> as.factor()

#Boosted Tree Test Accuracy:
Boosted_Tree=test_boosted_out |> accuracy(stroke, .pred)
```

## 5. Summary Table:

``` r
#Prepping graphic of model performance
acc_graph =rbind(
  Decision_Tree, Random_Forest, Boosted_Tree) |>
  as.data.frame()
acc_graph_names =c('1.Decision Tree', '2. Random_Forest', '3. Boosted Tree')
acc_graph = cbind(acc_graph, acc_graph_names)
names(acc_graph)[4] ='Model_Type'

acc_graph
```

    ##    .metric .estimator .estimate       Model_Type
    ## 1 accuracy     binary 0.9598826  1.Decision Tree
    ## 2 accuracy     binary 0.9569472 2. Random_Forest
    ## 3 accuracy     binary 0.9559687  3. Boosted Tree

It is evident that the decision tree model had the best testing
performance, at 95.99% accuracy. This is marginally better than the null
classifier of 95.13% and may indicate that subsequent models were
overfitted on the training data. I hope this walk-through has been
informative on the methods and syntax of TidyModels tree ML methods.
Below is a way to visualize what variables had the greatest predictive
power, resulting from the decision tree model and the vip R package.

``` r
#Which variables have the best predictive power?
tree_var_imp = test_wf_tree |> extract_fit_parsnip()
tree_var_imp |> vip()
```

![](Tree_Methods_files/figure-gfm/unnamed-chunk-13-1.png)<!-- -->

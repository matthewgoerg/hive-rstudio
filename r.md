# RStudio Environment

Go back to the command line on the master node and run this to install R on your cluster:

```clustercommandline
sudo apt-get update
sudo apt-get install -y \
    r-base r-base-dev \
    libcurl4-openssl-dev libssl-dev libxml2-dev

sudo apt-get install gdebi-core
wget https://download2.rstudio.org/server/debian9/x86_64/rstudio-server-1.2.1335-amd64.deb
sudo gdebi rstudio-server-1.2.1335-amd64.deb

sudo adduser [USER_NAME]
```

Open your browser and type http://localhost:8787 in the address bar. Enter your user name and password. 
Once you are in, open a new R script.

The first order of business is to install all of the packages we need. Installing packages in R 
can take much longer on GCP than the desktop versions we're accustom to.

```r
install.packages("sparklyr")
install.packages("dplyr")
install.packages("ggplot2")
install.packages("tidyr")
install.packages("lsei")
install.packages("stringr")
install.packages("rlang")
install.packages("MLmetrics")

library(sparklyr)
library(dplyr)
library(ggplot2)
library(tidyr)
library(lsei)
library(stringr)
library(rlang)
library(MLmetrics)

sparklyr::spark_install()
```

Next, we will configure Spark and connect to it. Spark will run on our cluster in the background and 
sparklyr will translate the R code into Scala (or Java? I'm not sure) code to make our lives easier.
While we are here it would be helpful to point out potential problems with this approach. 
- There are far fewer users in the sparklyr community than the R community as a whole. This means that there 
is less support in terms of forums. 
- The code is being translated into a paradigm and code that we are not as familiar with as R. When things go 
wrong, it is difficult to read through the error messages and debug. It would be a good idea to read up on Spark 
and Hive to understand how they work.
As an example, when I was creating this project, I was reminded that Hive tables are not automatically ACID compliant. 
When I tried to mutate some columns in dplyr, I ran into errors. I was stuck until I realized that instead of updating 
the single column I was interested in, Hive was creating a new table with the updated column and re-assigning it back 
to the name of the old table.

```r-base
#config
Sys.setenv(SPARK_HOME="/usr/lib/spark")
config <- spark_config()

#connect
sc <- spark_connect(master = "local")
```

Let's connect to the click_sample table so we can modify it.

```r
click_sample <- tbl(sc, "click_sample")
```

We need to convert the categorical variables into an expanded set of binary columns. This loop 
goes through each of the 26 categorical variable columns and does three things:
- ft_string_indexer converts each of the masked labels within the column to a numeric index because 
the machine learning models won't take strings as inputs
- the first categorical column contains over a million different labels and this is going to 
be too much for our models to handle. I learned that if we use only the 20 most frequent labels 
we can get similar ML performance at a much lower computational cost. The mutate step turns all 
of the labels that are not in the 20 most frequent to 0.
- ft_one_hot_encoder converts the indicies to a column of vectors with binary values for each 
level in the column.

```r
for (i in 1:26) {
idx_col <- paste0('cat_idx_', str_pad(i, 2, pad = "0"))

click_sample <- click_sample %>%
  ft_string_indexer(input_col = paste0('cat_feature_', str_pad(i, 2, pad = "0")),
                    output_col = paste0('cat_idx_', str_pad(i, 2, pad = "0"))) %>%
  mutate(!!idx_col := ifelse(!!sym(idx_col) >= 21, 21, !!sym(idx_col))) %>%
  ft_one_hot_encoder(input_col = paste0('cat_idx_', str_pad(i, 2, pad = "0")),
                     output_col = paste0('cat_vec_', str_pad(i, 2, pad = "0")))
}
```

We want to drop all of the original and intermediate categorical columns.

```r
click_sample <- click_sample %>%
  select(-starts_with('cat_feat')) %>%
  select(-starts_with('cat_idx'))
```

The neural net model needs to know how many individual features are in the model. This is complicated 
because the interger columns are length 1 and the lengths for each of the categorical columns ranges 
between 2 and 21. Here's a loop that counts the lengths of each categorical column and adds 13 for the 
number of integer columns.

```r
preview <- as.data.frame(head(click_sample, 1))

num_features <- 13

for (i in 15:40) {
  num_features <- features + length(preview[1,i][[1]])
}
```

Now we're ready to start the actual machine learning. My approach for this project is an ensemble 
of four models and consists of three steps: Training on 60% of the sample, constructing weights for 
each model with 10% of the data, and validating on 40% of the data to see how well we predict on new 
data.

```r
click_data <- click_sample %>%
  sdf_random_split(train = 0.6, validation = 0.4, weight = 0.1, seed = 123)

train_tbl <- click_data$train
test_tbl <- click_data$validation
weight_tbl <- click_data$weight
```

Here's the formula we will use in each model. 

```r
ml_formula <- formula(label ~ .)
```

The four models I'm using are Decision Tree, Gradient Boosted Trees, Random Forest, and Neural Net. 
Below I'm training each one. I am not doing much with the hyperparameters but they will need to be 
tuned eventually. Let's try running with mostly defaults and see how they perform.

```r
ml_dt <- ml_decision_tree_classifier(x=train_tbl, formula=ml_formula)
ml_gbt <- ml_gbt_classifier(train_tbl, ml_formula, max_iter = 250, step_size = 0.01)
ml_rf <- ml_random_forest_classifier(train_tbl, ml_formula, num_trees = 250)
ml_nn <- ml_multilayer_perceptron_classifier(train_tbl, ml_formula, layers = c(num_features, 200, 100, 2))
```

I need to convert this code into an lapply function. This step collects predicted values for 
each observation in the weighting data set and merges them all into a single table.

```r
log_pred <- ml_predict(ml_log, weight_tbl) %>%
  select(probability_1) %>%
  mutate(log_prob = probability_1) %>%
  select(-probability_1)

gbt_pred <- ml_predict(ml_gbt, weight_tbl) %>%
  select(probability_1) %>%
  mutate(gbt_prob = probability_1) %>%
  select(-probability_1)

rf_pred <- ml_predict(ml_rf, weight_tbl) %>%
  select(probability_1)  %>%
  mutate(rf_prob = probability_1) %>%
  select(-probability_1)

nn_pred <- ml_predict(ml_nn, weight_tbl) %>%
  select(label, probability_1) %>%
  mutate(nn_prob = probability_1) %>%
  select(-probability_1)

combined <- as.data.frame(cbind(nn_pred, gbt_pred, rf_pred, log_pred))
```

This step regresses a matrix of predicted values for each of the 4 models 
against the observed label to determine how the models compare. I use the 
pnnls() instead of lm() because I want the sum of the coefficients to be 1.

```r
a <- as.matrix(combined[2:4])
b <- combined$label

my_coefs <- pnnls(a, b, sum = 1)$x
my_coefs
```

This step repeats the process for the validation (test) data set and 
collects them into a table.

```r
log_pred <- ml_predict(ml_log, test_tbl) %>%
  select(probability_1) %>%
  mutate(log_prob = probability_1) %>%
  select(-probability_1)

rf_pred <- ml_predict(ml_rf, test_tbl) %>%
  select(probability_1)  %>%
  mutate(rf_prob = probability_1) %>%
  select(-probability_1)

nn_pred <- ml_predict(ml_nn, test_tbl) %>%
  select(label, probability_1) %>%
  mutate(nn_prob = probability_1) %>%
  select(-probability_1)

combined_test <- as.data.frame(cbind(log_pred, nn_pred, gbt_pred, rf_pred))
```

Here I'm adding a column to the table of predicted values for the test set.
weighted_prob is the weighted average for each of the predicted models. 

If my coefficients from the previous step are log=0.1, nn=0.2, gbt=0.4, rf=0.3 
and my predicted values for an observation are log=0.02, nn=0.03, gbt=0.03, rf=0.01 
then the weighted probability for this observation is 
0.1*0.02 + 0.2*0.03 + 0.4*0.03 + 0.3*0.01 = 0.023. The weighted probability should be 
more accurate than any of the individual models alone.

```r
combined_test <- combined %>%
  mutate(weighted_prob = nn_prob*my_coefs[1]+gbt_prob*my_coefs[2]+rf_prob*my_coefs[3])
```

The log-loss for the weighted probability is 0.137086, compared to 0.44 for the winning Kaggle entry. I'm not sure 
why my score is so much lower but it is probably due to the fact that the Kaggle dataset was a smaller subset of 
the data that I'm using. I will investigate to find out for sure.

```r
LogLoss(combined_test$weighted_prob, combined_test$label)
```

Regardless, my models do have some predictive power. Here's the ROC curves for the individual models:


For an intutive interpretation of the models' usefulness consider we can break down the predicted probabilities.
The overall sample click rate is 0.03203 or ~3.2%. My model gave a range of predicted values for each observation.
If we take the number of clicks for observations where the predicted probability of clicking is at or above the 
median, we get a click rate of 0.0489 or ~4.9% which is well above the sample rate. For the observations with 
predicted probabilities below the median, the click rate is 0.01514482 (~1.5%), less than half of the sample
click rate.

```r
summary(combined_test)



     label            nn_prob           gbt_prob          rf_prob        weighted_prob    
 Min.   :0.00000   Min.   :0.02271   Min.   :0.04082   Min.   :0.01486   Min.   :0.01764  
 1st Qu.:0.00000   1st Qu.:0.02850   1st Qu.:0.04680   1st Qu.:0.02600   1st Qu.:0.02812  
 Median :0.00000   Median :0.02942   Median :0.05109   Median :0.03043   Median :0.03251  
 Mean   :0.03203   Mean   :0.03420   Mean   :0.05688   Mean   :0.03243   Mean   :0.03484  
 3rd Qu.:0.00000   3rd Qu.:0.03211   3rd Qu.:0.05951   3rd Qu.:0.03707   3rd Qu.:0.03921  
 Max.   :1.00000   Max.   :0.12803   Max.   :0.65463   Max.   :0.10008   Max.   :0.13684  

combined_test %>%

  filter(weighted_prob >= 0.03251) %>%

  select(label, weighted_prob) %>%

  summarize(sum(label), n())

[1] 0.04893169

combined_test %>%

  filter(weighted_prob < 0.03251) %>%

  select(label, weighted_prob) %>%

  summarize(sum(label), n())

[1] 0.01514482
```

[Dashboard with Shiny](https://github.com/matthewgoerg/hive-rstudio/blob/master/shiny.md)
# SuperLearnerMacro v1.0
A SAS macro for stacking, a supervised ensemble machine learning approach to prediction

### Requirements
1. Windows 8+ or Red Hat Linux 7.1+ (untested on other versions)
2. SAS v 9.4+ (untested earlier versions)
3. SAS/STAT + SAS/OR v 14.1+ (untested earlier versions)

### Recommended enhancements
1. SAS Enterprise Miner High Performance Procedures 14.1+ (enables many data adaptive learners)
2. SAS/IML v 13.1+ (enables use of R functions)


### Quick start
Quick simulation example: predicting continuous outcomes in a validation sample by combining predictions from a linear regression, LASSO regression, and a generalized additive model

Place the following two lines at the top of your program:

    FILENAME slgh URL "https://raw.githubusercontent.com/alexpkeil1/SuperLearnerMacro/master/super_learner_macro.sas";
    %INCLUDE slgh / SOURCE2;
    


Simulate data

    DATA train valid ;
      LENGTH id x l 3;
      CALL STREAMINIT(1192887);
      DO id = 1 TO 1100;
        u = RAND("uniform")*0.1 + 0.4;  
        l = RAND("bernoulli", 1/(1+exp(-1 + u)));
        c = RAND("normal", u, 1);
        c2 = RAND("normal", u, .3);
        x = RAND("bernoulli", 1/(1+exp(-1.5 + 2*l + c + c2)));
        y = RAND("NORMAL", u + x, 0.5);
        KEEP x l c c2 y;
        IF id <= 100 THEN OUTPUT train;
        ELSE OUTPUT valid;
      END;
    RUN;

Call super learner macro

    TITLE "Super learner predictions in validation data";
    %SuperLearner(Y=y,
                  X=x l c c2,
                  indata=train, 
                  preddata=valid, 
                  outdata=sl_output,
                  library= linreg lasso gampl,
                  folds=10, 
                  method=NNLS, 
                  dist=GAUSSIAN 
    );
    
Estimate mean squared error
    
    DATA mse(KEEP=__train squarederror:);
     SET sl_output;
     squarederror_sl = (y - p_SL_full)**2;
     squarederror_linreg = (y - p_linreg_full)**2;
     squarederror_lasso = (y - p_lasso_full)**2;
     squarederror_gampl = (y - p_gampl_full)**2;
    PROC MEANS DATA = mse FW=5 MEAN;
     TITLE 'Mean squared error of predictions in training/validation data';
     VAR squarederror:;
     CLASS __train;
    RUN;
    
    
Results: super learner has lowest mean squared prediction error (__train=0)

    Mean squared error of predictions in training/validation data
                  The MEANS Procedure
   
                  N
    __train     Obs    Variable                Mean
    -----------------------------------------------
          0    1000    squarederror_sl        0.245
                       squarederror_linreg    0.247
                       squarederror_lasso     0.280
                       squarederror_gampl     0.247
   
          1     100    squarederror_sl        0.211
                       squarederror_linreg    0.209
                       squarederror_lasso     0.257
                       squarederror_gampl     0.207
    -----------------------------------------------



### Usage details
The sas script containing the SuperLearner macro actually contains 4 main macros: %SuperLearner, _SuperLearner, %CVSuperLearner macro, and %_CVSuperLearner

#### 0. Common features of all macros
##### Available Learners (* denotes SAS Enterprise Miner procedure)

- **back**: backward selection by BIC
- **bagging**: * Bootstrap aggregation of regression/classification trees
- **bayesnet**: * Bayesian network [binary only]
- **boxcox**: Box-Cox transformation of target variables, positively bound continuous Y only
- **boost**: * Gradient boosting of regression/classification trees
- **cvcart**: classification/regression tree with cross validated selection of meta parameters
- **cart**: classification/regression tree, no cross validation
- **enet**: elastic net - warning for binary outcome: does not respect [0,1] probability space 
- **gam**: generalized additive model with 3 df splines on all continuous variables
- **gampl**: faster generalized additive model with 3 df splines on all continuous variables
- **glm**: linear or logistic regression: slower wrappers for logit and linreg
- **knn**: [bernoulli only] k-nearest neighbors classification
- **lar**: least angle regression - warning for binary outcome: does not respect [0,1] probability space
- **lasso**: LASSO
- **lassob**: LASSO with glmselect [use caution with binary variables] - may be appropriate for older sas versions
- **logit**: main term logistic regression
- **linreg**: main term linear regression
- **mars**: multivariate adaptive regression splines
- **mean**: marginal mean of the prediction variable
- **nbayes**: * naive Bayes
- **bspline**: basis spline regression
- **pbspline**: penalized basis spline regression [SINGLE CONTINUOUS PREDICTOR ONLY]
- **probit**: main term probit regression
- **ridge**: ridge regression - warning for binary outcome: does not respect [0,1] probability space
- **quantreg**: Quantile regression for the median - warning for binary outcome: does not respect [0,1] probability space
- **rf**: random forest
- **rfoob**: random forest, using out of bag predictions and modified selection criteria
- **nn**: * neural network
- **sherwood**: * random forest, using proc ARBOR, sampling with replacement
- **swise**: Stepwise model selection with HPGENSELECT - may be appropriate alternative to lasso for older sas versions

Also includes multiple learners that are identical to other learners but include all first order interaction terms:

**backint**, **logitint**, **linregint**, **lassoint**, **lassobint**, **swiseint**, **larint**, **enetint**, **gamint**, **gamplint**, **marsint**, **probitint**
  
Included R functions (requires SAS/IML and RLANG system option enabled) allows a limited set of functions that call learners in the R programming language. Provided that R is installed and the RLANG option properly enabled, the required packages will be automatically installed the first time the learner is called (if running the SuperLearner or CVSuperLearner macros)

- **r_bagging**:  Bootstrap aggregation of regression/classification trees (requires ipred, rpart packages)
- **r_bart**:  Bayesian additive regression/classification trees (requires dbarts package)
- **r_boost**:  Gradient boosting regression/classification (requires xgboost)
- **r_gam**:  generalized additive model (requires gam, foreach, splines packages)
- **r_mars**: MARS - multivariate adaptive regression splines (requires earth package)
- **r_polymars**: MARS - multivariate adaptive polynomial regression splines (requires polspline package)
- **r_rf**: random forest using R superlearner defaults (requires randomForest package)
- **r_svm**:  support vector machine regression/classification (requires e1071)

##### Creating new learners
All sas macros to enable a learner in the library are of the form:

    %learner_in(Y,indata,outdata, binary_predictors,ordinal_predictors,nominal_predictors,
    continuous_predictors,weight,suff,seed);

 Using this standard call, it is relatively straightforward to add new learners, provided that the conventions are followed.
Conventions:

1. each macro name must be structured like binary dep. vars:     %[library name]\_in  non-binary dep. vars: %[library name]\_cn where [library name] is the user defined name for a given learner. E.g. for a  random forest, [library name] is 'rf' (without quotes)

2. macro call/parameters must follow exact naming conventions as shown above in '%learner_in' example

3. the sample space of each predictor must be respected (e.g. &binary_predictors should be used where binary predictors are appropriate for the learner.) For example,  model statements in, PROC genmod will contain all &..._predictor macro variables, but  &nominal_predictors could be included in a CLASS statement, for example.

4. outdata must contain: all variables from indata dataset (the data used in the learner) PLUS a variable that follows the naming convention: p_[library name]&SUFF that is either a) predicted probability (binary dep var) or b) predicted value (non- binary dep var)

5. to include all interaction terms between predictors, the predictors must include  &SLIXterms, which may be a mix of discrete and continuous variables

Example: creating a learner that fits a generalized linear model with a log link and under a gamma error distribution

     %MACRO custom_cn(Y=,indata=, outdata=, binary_predictors=, ordinal_predictors=,nominal_predictors=,continuous_predictors=,weight=,suff=,seed=);  
      PROC GENMOD DATA = &indata;
      %IF ((&binary_predictors~=) OR (&ordinal_predictors~=) OR (&nominal_predictors~=)) %THEN CLASS &binary_predictors &ordinal_predictors &nominal_predictors ;;
       MODEL &Y = &binary_predictors &ordinal_predictors &nominal_predictors &continuous_predictors / LINK=LOG D=GAMMA;
       OUTPUT OUT = &OUTDATA PRED=p_custom&SUFF; /*change predicted value name with p_[libraryname]&suff*/
      RUN;
    %MEND custom_cn; /*optional: include macro name in mend statement with [libraryname]_cn*/

Custom leaners must have the same structure of the macro call, but the macro need not actually use the arguments. For example, rather than using **[coding]_predictors** arguments, one could hard code the model predictors, including interaction terms. Thus, one could create multiple learners with different covariate sets as a way to select from a number of possible parameterizations of the same model, but with different covariates (i.e. one could select the model with the lowest loss function, or take the model average based on the super learner fit).

#### 1. %SuperLearner macro

Stacking is based on what Wolpert refers to as a set of 'level-0' models and a 'level-1' model, indexed by parameters $\mathbf{\beta}_m$ and $\mathbf{\alpha}$ in some study sample $S$. Where

Level-0: $\hat{Y}_{m} = f_m(\mathbf{x};\mathbf{\beta}_m,S) \mbox{ for }m \in 1,\ldots,M $

Level-1: $\hat{Y}_{sl} = f_{sl}(\hat{\mathbf{Y}}_{\bar{m}};\mathbf{\alpha},S)$

The parameterization of the macro is based loosely on this notation. Macro parameters include the following:

- **Y**: [value = variable name] the target variable, or outcome

- **X**: [value =   blank, or a space separated list of variable names] predictors of **Y** on the right side of the level-0 models. Note that this is a convenience function for the individual **[coding]_predictors** macro variables. The macro will make a guess at whether each predictor in **X** is continuous, categorical, or binary. (OPTIONAL but at least one of the **X** or **[coding]_predictors** parameters must be specified). If **X** is specified and any one of the **[coding]_predictors** has a value, the macro will generate an error.

- **library**: [value =  a space separated list of learners] the names of the *m* level-0 models (e.g. glm lasso cart). A single learner can be used here if you only wish to know the cross-validated expected loss (e.g. mean-squared error).

- **indata**: [value = an existing dataset name] the dataset used for analysis that contains *Y* and all predictors (and weight variables, if needed)

- **preddata**: [OPTIONAL value = a dataset name] the validation dataset. A dataset which contains all predictors and possibly *Y* that is not used in model fitting but predictions for each learner and superlearner are made in these data

- **outdata**: [value = a dataset name; default: sl_out] an output dataset that will contain all predictions as well as all variables and observations in the **indata** and **preddata** datasets

- **dist**: [value = one of: GAUSSIAN,BERNOULLI; default GAUSSIAN] Super learner can be used to make predictions of a continuous (assumed gaussian in some learners) or a binary variable. Use GAUSSIAN for all continuous variables and BERNOULLI for all binary variables. Nominal/categorical variables currently not supported.

- **method**:[value = one of: NNLS,NNLOGLIK,CCLOGLIK,LOGLIK,NNLS,OLS,CCLAE,NNLAE,LAE; default NNLS] the method used to estimate the $\alpha$ coefficients of the level-1 model.                    Methods are possibly indexed by prefixes: NN, CC, [none], where 

   NN implies non-negative coefficients that are standardized after fitting to sum to 1. 

   CC implies a convexity constraint where the super learner fit is subject to a constraint 
  that forces the coefficients to fall in [0,1] and sum to 1.0. No prefix implies no 
  constraints (which results in some loss of asymptotic properties such as the oracle property).
   Note: OLS violates this naming convention, but LS will also be accepted and is equivalent to OLS

   LS methods use an L2 loss function (least squares)
   LOGLIK methods use a loss function corresponding to the binomial likelihood with a logit link function
   LAE methods [experimental] use an L1 loss function (least absolute error), which will not penalize outliers as much as L2 methods, and is also non-differentiable at the minimum
  which may cause computational difficulties

- **by**: [OPTIONAL value = variable name] a by variable in the usual SAS usage. Separate super learner fits will be specified for each level of the by variable (only one allowed, unlike typical ``by'' variables. 
- **intvars**:[OPTIONAL value = variable name] an intervention variable that is included in the list of predictors. This is a convenience function that will make separate predictions for the intvars variable at 1 or 0 (with all other predictors remaining at their observed levels)
- **binary_predictors**: [value =  blank, or a space separated list of variable names] advanced specification of predictors: a space separated list of binary predictors (OPTIONAL but at least one of the **X** or **[coding]_predictors** parameters must be specified)
- **ordinal_predictors**: [value =  blank, or a space separated list of variable names]advanced specification of predictors: a space separated list of ordinal predictors (OPTIONAL but at least one of the **X** or **[coding]_predictors** parameters must be specified)
- **nominal_predictors**: [value =  blank, or a space separated list of variable names]advanced specification of predictors: a space separated list of nominal predictors (OPTIONAL but at least one of the **X** or **[coding]_predictors** parameters must be specified)
- **continuous_predictors**: [value =  blank, or a space separated list of variable names] advanced specification of predictors: a space separated list of continuous predictors (OPTIONAL but at least one of the **X** or **[coding]_predictors** parameters must be specified)
- **weight**: [OPTIONAL value = a variable name] a variable containing weights representing the relative contribution of each observation to the fit (a.k.a. case weights). Not all learners will respect non-integer weights, so weights will either be ignored or truncated by some procedures.
- **trtstrat**: [value  = true, false; DEFAULT: false] convenience function. If this is set to true and **intvars** is specified, then all fits will be stratified by levels of **intvars** (0,1) accepted only.
- **folds**: [value  = integer ; default: 10] number of cross-validation folds to use

#### 2. %\_SuperLearner macro
This is a less user-friendly version of the %SuperLearner macro that may be somewhat faster due to reduced error checking, and offers finer level controls. If the %SuperLearner macro completes successfully, it will give some example code that can be run with %_SuperLearner. Of note, there is no checking of parameter syntax, so the case-sensitive parameter arguments may cause an error in %\_SuperLearner, but not %SuperLearner.

One main difference is that %\_SuperLearner will make no guesses about variable types for **X**, so use of the **[coding]_predictors** is required for correct specification.

#### 3. %CVSuperLearner macro
- **Y**: see %SuperLearner macro definition
- **X**: see %SuperLearner macro definition
- **by**: see %SuperLearner macro definition
- **binary_predictors**:  see %SuperLearner macro definition
- **ordinal_predictors**:  see %SuperLearner macro definition
- **nominal_predictors**: see %SuperLearner macro definition 
- **continuous_predictors**:  see %SuperLearner macro definition
- **weight**: see %SuperLearner macro definition
- **indata**: see %SuperLearner macro definition
- **outdata**: see %SuperLearner macro definition 
- **dist**: see %SuperLearner macro definition (default: GAUSSIAN)
- **library**:  see %SuperLearner macro definition 
- **slfolds**:[value = integer; default: 10] number of ''inner folds'' (number of folds within each super learner fit) should only be different from **cvslfolds** in odd cases
- **cvslfolds**:[value = integer; default: 10] number of ''outer folds'' (the number of folds for cross- validating super learner) should only be different from **slfolds** in odd cases
- **method**:  see %SuperLearner macro definition (default: NNLS)


#### 4. %\_CVSuperLearner macro
This is a less user-friendly version of the %CVSuperLearner macro that may be somewhat faster due to reduced error checking, and offers finer level controls. See the source code for further tuning options.
author: sdevitt@snowflake.com
id: HrAnalytics
summary: This lab will walk you through how to use Snowflake, Hex and Modelbit.
categories: data-science-&-ml,partner-integrations
environments: web
status: Unpublished
feedback link: https://github.com/Snowflake-Labs/sfguides/issues
tags: Hex, Modelbit, Notebooks, Partner Connect

# Building and deploying a Nurse Attrition model with Hex + Modelbit + Snowflake

<!-- ------------------------ -->
## Lab Overview 
Duration: 5

In this demo, we will play the role of a data scientist at a large hospital tasked with identifying employees at risk of attrition and factors contributing to attrition. To do this, we would like to build a predictive model using our employee database and information about staff attrition. Let's see how we can use Hex in collaboration with Snowflake/Snowpark to do some Exploratory Data Analysis and then build a random forest forecasting model. Once our model is tuned, we will use Snowflake Model registry and modelbit to easily deploy our model to Snowflake so that we can do inference scoring.

### Prerequisites
- Familiarity with basic Python and SQL 
- Familiarity with training ML models
- Familiarity with data science notebooks
- Familiarity with Machine Learning pipelines and deployment
- Modelbit trial account


### What You'll Learn
* How to load data into Snowflake
* How to import/export data between Hex and Snowflake
* How to train a Random forest model and deploy to Snowflake using Snowflake Model registry or Modelbit
* How to visualize the predicted results from the forecasting model
* How to convert a Hex project into an interactive web app

## Data Loading into Snowflake
We need to first download these files to the local workstation by clicking on the hyperlink below. 

&nbsp;

#### Synthetic Employee HR Data 
Data for this lab was generated using Gretel.ai based on a sample HR dataset schema.

We need to first download the sample data for this lab to the local workstation by clicking on the hyperlink below

[employees_merged.csv](</data/employees.csv>) 


## Loading the Data into Snowflake
There are amny ways to load data into Snowflake, we will highlight one of the easiest, using our new data loader to upload data, infer schema and generate a table, no need to predefine your DML.

Navigate to Snowflake and Log in. 

[](assets/LoadData.png)
[](assets/parseSchema.png)
[](assets/DataLoaded.png)

Once the data is loaded, we can set up our Hex Connection

## Setting up partner connect

Duration: 5

After logging into your Snowflake account, you will land on the `Learn` page. To connect with Hex, navigate to the `Admin` tab on the left and click on `Partner connect`. In the search bar at the top, type `Hex` and the Hex partner connect tile will appear. Clicking on the tile will bring up a new screen, and click the `connect button` in the lower right corner. A new screen will confirm that your account has been created, from which you can click `Activate`.

![](assets/pc.gif)

### Creating a workspace

After activating your account, you'll be directed to Hex and prompted to create a new workspace and give it a name. Once you've named your workspace, you'll be taken to the projects page where you can create new projects, import existing projects (Hex or Jupyter), and navigate to other sections of your workspace.

## Getting Started with Hex

Duration: 5

Now we can move back over to Hex and get started on our project. The first thing you'll need to do is get the Hex project that contains all of the code we'll work through to train our model.

Clicking this button will copy the template project into your new workspace.

<button>

[Duplicate Hex project](https://app.hex.tech/snowflake/hex/924d2ac6-bf26-4757-980f-95453d6820ef/draft/logic)

</button>

Now that you've got your project imported, you will find yourself in the [Logic view](https://learn.hex.tech/docs/develop-logic/logic-view-overview) of a Hex project. The Logic view is a notebook-like interface made up of cells such as code cells, markdown cells, input parameters and more! On the far left side, you'll see a control panel that will allow you to do things like upload files, import data connections, or search your project.

Before we dive into the code, we'll need to import our Snowflake data connection, which has been automatically created by the partner connect process.

Head over to the Data sources tab represented by a database icon with a lightning bolt. You should see two data connections - [Demo] Hex public data and [Snowflake]. Import both connections.

![](assets/DC.gif)

One nice feature of Hex is the [reactive execution model](https://learn.hex.tech/docs/develop-logic/compute-model/reactive-execution). This means that when you run a cell, all related cells are also executed so that your projects are always in a clean state. However, if you want to ensure you don’t get ahead of yourself as we run through the tutorial, you can opt to turn this feature off. In the top right corner of your screen, you’ll see a Run mode dropdown. If this is set to Auto, select the dropdown and change it to cell only.

![](assets/mode.gif)



## Reading data into HEX

Duration: 8

To predict employee attrition, we first need data to train our model. In the SQL cell labeled **Retrieving Data** assign `[Snowflake] ` as the data connection source and run the cell.

![](assets/pull_data.png)



## Data preparation

Duration: 2

Now that we have our data in Hex, we want to make sure the it’s clean enough for our machine learning algorithm. To ensure this, we’ll first check for any null values.

```python
data.isnull().sum()
```

Now that we have checked for null values, let's look at the distribution of each variable.

```python
# create a 15x5 figure
plt.figure(figsize=(15, 5))

# create a new plot for each variable in our dataframe
for i, (k, v) in enumerate(data.items(), 1):
    plt.subplot(3, 5, i)  # create subplots
    plt.hist(v, bins = 50)
    plt.title(f'{k}')

plt.tight_layout()
plt.show();
```

![](assets/chart.png)



## Understanding attrition rate

Duration: 2

If you take a look at our visuals, you may notice that the attrition chart looks a little odd. Specifically, it looks like there are a lot more users who haven’t churned than who have.

We take a closer look at this by visualizing in a chart cell.

![](assets/imbal.png)
As you can see, the majority of observations are in support of user who haven’t yet churned. Specifically, 94% of employees haven’t churned (attrited) while the other 6% has. In machine learning, a class imbalance such as this can cause issues when evaluating the model since it’s not getting equal attention from each class. In the next section we will go over a method to combat the imbalance problem.

## Establishing a Snowpark connection

Duration: 2

Now, we can connect to our Snowflake connection that we imported earlier. To do this head over to the data sources tab on the left control panel to find your Snowflake connection. If you hover your mouse over the connection and click on the dropdown next to the `query` button and select `get Snowpark session`. This will create a new cell for us with all the code needed to spin up a Snowpark session.

![](assets/snowpark.gif)

To ensure we don't run into any problems down the line, paste in the following line at the bottom of the cell.

```python
session.use_schema("HR_ANALYTICS_DB.PUBLIC")
```

## Feature engineering

Duration: 10

In order to predict the attrition outcomes for customers not in our dataset, we’ll need to train a model that can identify users who are at risk of churning from the history of users who have. However, it was mentioned in the last section that there is an imbalance in the class distribution that will cause problems for our model if not handled properly. One way to handle this is to create new data points such that the classes balance out. This is also known as upsampling.

For this, we’ll be using the `SMOTE` algorithm from the `imblearn` package.

First, we'll get our features— aka everything except the target column, ATTRITION. We do this in a SQL cell, making use of the SELECT / EXCLUDE syntax in Snowflake! Uncomment + run the SQL cell called "Features".

Next we need to do some feature engineering - we will drop null values, select the relevant columns. We can use Snowpark ML Preprocessing helper functions to do one hot encoding of our categorical variables, transform using standar scalar and get our final feature set. 

Now is a good time to go back to Snowflake to see how we've been pushing down all our operations to our Snowflake warehouse.

Now, we'll upsample. We could do this locally using SMOTE, but we want this entire workflow to run in Snowpark end-to-end, so we're going to create a stored procedure to do our upsampling.

Run the code cell labeled **Upsampling the data**.

```python
def upsample(
    session: Session,
    features_table_name: str,
) -> str:

    import pandas as pd
    import sklearn
    from imblearn.over_sampling import SMOTE

    features_table = session.table(features_table_name).to_pandas()
    X = features_table.drop(columns=["Churn"])
    y = features_table["Churn"]
    upsampler = SMOTE(random_state=111)
    r_X, r_y = upsampler.fit_resample(X, y)
    upsampled_data = pd.concat([r_X, r_y], axis=1)
    upsampled_data.reset_index(inplace=True)
    upsampled_data.rename(columns={'index': 'INDEX'}, inplace=True)



    upsampled_data_spdf = session.write_pandas(
        upsampled_data,
        table_name=f'{features_table_name}_SAMPLED',
        auto_create_table=True,
        table_type='', # creates a permanent table
        overwrite=True,
    )

    return "Success"
```

Now we have created a function that will be our stored procedure. It will perform upsampling, and write the upsampled data back to a new Snowflake table called '{features_table_name}\_SAMPLED'.

Now we create + call the stored procedure, in 2 more python cells:

```python
session.sql('CREATE OR REPLACE STAGE SNOWPARK_STAGE').collect()

session.sproc.register(
    upsample,
    name="upsample_data_with_SMOTE",
    stage_location='@SNOWPARK_STAGE',
    is_permanent=True,
    execute_as='caller',
    packages=['imbalanced-learn==0.10.1', 'pandas', 'snowflake-snowpark-python==1.6.1', 'scikit-learn==1.2.2'],
    replace=True,
)
```

```python
# # here we call the sproc to perform the upsampling
session.call('upsample_data_with_SMOTE', 'ATTRITION_FEATURES')
```

Now all that's left is to query the upsampled data from the table using another SQL cell, in this case the Features upsampled cell. If you prefer, you could change your stored procedure to return a Snowpark DataFrame directly rather than writing back a permanent table, but if you are going to be doing continued work on this dataset, writing a permanent table may make more sense.

Now that we have balanced our dataset, we can prepare our model for training. The model we have chosen for this project is a Random Forest classifier. A random forest creates an ensemble of smaller models that all make predictions on the same data. The prediction with the most votes is the prediction the model chooses.

Rather than use a typical random forest object, we'll make use of Snowflake ML. Snowflake ML offers capabilities for data science and machine learning tasks within Snowflake. It provides estimators and transformers compatible with scikit-learn and xgboost, allowing users to build and train ML models directly on Snowflake data. This eliminates the need to move data or use stored procedures. It uses wrappers around scikit-learn and xgboost classes for training and inference, ensuring optimized performance and security within Snowflake.

In the cell labeled `Snowflake ML model preprocessing` we'll import Snowpark ML to further process our dataset to prepare it for our model.

```python
import snowflake.ml.modeling.preprocessing as pp

numeric=['SALARY', 'SENIORITY', 'TENURE_MONTHS', 'MONTHS_AFTER_COLLEGE','BIRTH_YEAR', 'OVERTIME_HOURS', 'DISTANCE']
categorical= ['MAPPED_ROLE_CLEAN', 'SEX', 'ETHNICITY','DEGREE_CLEAN']

# StandardScalar
numeric_transformer = pp.StandardScaler(
    input_cols = numeric,
    output_cols = numeric,
)
numeric_transformer.fit(features)
features = numeric_transformer.transform(features)


#OneHotEncoding
categorical_transformer = pp.OneHotEncoder(
    input_cols = categorical,
    output_cols = categorical
)
categorical_transformer.fit(features)
features = categorical_transformer.transform(features)


# Data Clean Up
features = features.drop(categorical)
features.show()
```

### Accepting Anaconda terms

Before we can train the model, we'll need to accept the Anaconda terms and conditions.
To do this, navigate back to Snowflake and click on your username in the top left corner. You'll see a section that will allow you to switch to the `ORGADMIN` role. Once switched over, navigate to the `Admin` tab and select `Billing & Terms`. From here, you will see a section that will allow you to accept the anaconda terms and conditions. Once this is done, you can head back over to Hex and run the cell that trains our model.

![](assets/vhol-accept-terms.gif)

## Model training

Duration: 5

Now we can train our model. Run the cell labeled `Model training`.

```python
from snowflake.ml.modeling.ensemble import RandomForestClassifier
import snowflake.snowpark.functions as f


input_cols = features.drop('ATTRITION').columns
label = ['ATTRITION']
output = ['PREDICTED_ATTRITION']

model = RandomForestClassifier(
    input_cols = input_cols,
    label_cols = label,
    output_cols = output,
    random_state = 66,
    n_jobs= -1
)

# Note: random_split doesn't exactly match one to one with sklearn even with the same seed!
features_train, features_test = features.random_split(weights=[0.7, 0.3], seed=66)

model.fit(features_train)
result = model.predict(features_test)
```

![](assets/train.png)

In the next section, we will look at how well our model performed as well as which features played the most important role when predicting the attrition outcome.

## Model evaluation and feature importance

Duration: 5

In order to understand how well our model performs at identifying users at risk of churning, we’ll need to evaluate how well it does predicting attrition outcomes. Specifically, we’ll be looking at the recall score, which tells us _of all the customers that will attrite, how many can it identify._

Run the code cell labeled **Evaluate model** on _accuracy and recall._

```python
import snowflake.ml.modeling.metrics as m

precision = m.precision_score(
    df = result,
    y_true_col_names = 'ATTRITION',
    y_pred_col_names = 'PREDICTED_ATTRITION'
)
recall = m.recall_score(
    df = result,
    y_true_col_names = 'ATTRITION',
    y_pred_col_names = 'PREDICTED_ATTRITION'
)
```

This will calculate an accuracy and recall score for us which we'll display in a [single value cell](https://learn.hex.tech/docs/logic-cell-types/display-cells/single-value-cells#single-value-cell-configuration).

![](assets/scores.png)

### Feature importance

Next, we want to understand which features were deemed most important by the model when making predictions. Lucky for us, our model keeps track of the most important features, and we can access them using the `feature_importances_` attribute. Since we're using Snowflake-ml, we'll need to extract the original `sklearn` object from our model. Then we can perform feature importance as usual.

```python
rf = model.to_sklearn()
importances = pd.DataFrame(
    list(zip(features.columns, rf.feature_importances_)),
    columns=["feature", "importance"],
)
```

Let’s visualize the most important features.
![](assets/important.png)

## Predicting attrition for a new user

Duration: 10

Now is the moment we've all been waiting for: predicting the attrition outcome for a new user. In this section, you should see an array of input parameters already in the project. Each of these inputs allow you to adjust a different feature that goes into predicting employee attrition, which will simulate a new user. But we’ll still need to pass this data to our model, so how can we do that?

![](assets/input.png)

Each input parameter has its own variable as its output, and these variables can be referenced in a Python cell. The model expects the inputs it receives to be in a specific order otherwise it will get confused about what the features mean. Keeping this in mind, execute the Python cell labeled **_Create the user vector_**.

```python
from sklearn.preprocessing import StandardScaler

user_vector = np.array([
     distance,
     1 if bachelors_degree else 0,
     tenure_months,
     Overtime_hours,
     months_after_college,
     SEX,
     Role,
     birth_year
 ]).reshape(1,-1)

user_dataframe = pd.DataFrame(user_vector, columns=StandardScaler.input_cols)
user_vector = StandardScaler.transform(user_dataframe)
user_vector.columns = [column_name.replace('"', "") for column_name in user_vector.columns]
user_vector = session.create_dataframe(user_vector)
```

This creates a list where each element represents a feature that our model can understand. However, before our model can accept these features, we need to transform our array. To do this, we will convert our list into a numpy array and reshape it so that there is only one row and one column for all features.

```python
user_vector = np.array(inputs).reshape(1, -1)
```

As a last step, we’ll need to scale our features within the original range that was used during the training phase. We already have a scaler fit on our original data and we can use the same one to scale these features.

```python
user_vector_scaled = scaler.transform(user_vector)
```

The final cell should look like this:

```python
from sklearn.preprocessing import StandardScaler

user_vector = np.array([
     distance,
     1 if bachelors_degree else 0,
     tenure_months,
     Overtime_hours,
     months_after_college,
     SEX,
     Role,
     birth_year
 ]).reshape(1,-1)

user_dataframe = pd.DataFrame(user_vector, columns=StandardScaler.input_cols)
user_vector = StandardScaler.transform(user_dataframe)
user_vector.columns = [column_name.replace('"', "") for column_name in user_vector.columns]
user_vector = session.create_dataframe(user_vector)

user_dataframe = pd.DataFrame(user_vector, columns=scaler.input_cols)
user_vector = scaler.transform(user_dataframe)
user_vector.columns = [column_name.replace('"', "") for column_name in user_vector.columns]
user_vector = session.create_dataframe(user_vector)
```

We are now ready to make predictions. In the last code cell labeled **_Make predictions and get results_**, we will pass our user vector to the model's predict function, which will output its prediction. We will also obtain the probability for that prediction, allowing us to say: **_"The model is 65% confident that this user will attrition."_**

```python
predicted_value = model.predict(user_vector).toPandas()[['predicted_churn'.upper()]].values.astype(int).flatten()
user_probability = model.predict_proba(user_vector).toPandas()
probability_of_prediction = max(user_probability[user_probability.columns[-2:]].values[0]) * 100
prediction = 'attrition' if predicted_value == 1 else 'not attrition'
```

To display the results in our project, we can do so in a markdown cell. In this cell, we’ll use Jinja to provide the variables that we want to display on screen.

```markdown
{% if predict %}

#### The model is {{probability_of_prediction}}% confident that this user will {{prediction}}

{% else %}

#### No prediction has been made yet

{% endif %}
```

## Making Hex apps

Duration: 5

At this stage of the project, we have completed building out our logic and are ready to share it with the world. To make the end product more user-friendly, we can use the app builder to simplify our logic. The app builder enables us to rearrange and organize the cells in our logic to hide the irrelevant parts and only show what matters.

![](assets/app.gif)

Once you've arranged your cells and are satisfied with how it looks, use the share button to determine who can see this project and what level of access they have. Once shared, hit the publish button and your app will go live.

![](assets/share.gif)

## Conclusion And Resources

Duration: 1

Congratulations on making it to the end of this Lab where we explored attrition modeling using Snowflake and Hex. We learned how to import/export data between Hex and Snowflake, train a Random Forest model, visualize predictions, convert a Hex project into a web app, and make predictions for new users. You can view the published version of this [project here](https://app.hex.tech/hex-public/app/8bd7b9bb-7f6c-41f1-9b4c-ff563a7fcaea/latest)!

### What we've covered

- Use Snowflake’s “Partner Connect” to seamlessly create a Hex trial
- How to navigate the Hex workspace/notebooks
- How to train an Random forest model and deploy to Snowflake using UDFs

### Resources

- [Hex docs](https://learn.hex.tech/docs)
- [Modelbit docs](https://doc.modelbit.com/getting-started/)
- [Snowflake Docs](https://docs.Snowflake.com/en/)

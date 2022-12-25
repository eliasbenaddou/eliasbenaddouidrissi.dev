---
title: "Data Engineering Project - Personal Finances with Airflow, Docker, Great Expectations and Metabase"
date: 2022-12-06
draft: false
slug: ""
tags: ['data_engineering', 'python', 'airflow', 'docker']
categories: []
---

*TLDR:*
*A dockerised Airflow repository to automate API calls to your bank account (only Monzo currently supported) to store transactions in a Postgres database, check data quality with Great Expectations and visualise your data on Metabase. DAGs to pull transactions and store to a database, move money between Monzo Pots for automated budgetting and to create custom push notifications to your Monzo app.*

### About

Finding data to use in a project can be difficult and with all the tools to choose from it can become overwhelming. Recently I wrote about automating my budget tracking with python using Monzo as my preferred banking app as they have a really good public API. After hearing from others who were interested in how I did it, I decided to make it into a data engineering project. In this post I will go over the key components of the project, the tools used and how to use it.

Thanks to https://github.com/petermcd/monzo-api for the Python package used to interact with Monzo's public API.

With this pipeline you will be able to load transactions from your current account and create a dashboard to visualise your data with Metabase. The DAGs will update your database daily with new transactions and perform data quality checks on them to see which transactions lack metadata (e.g. notes about the transaction). 

It can also move money from your current account to any Monzo pots you have as part of an automated periodic budget allocation (e.g. ensuring £30 exists in your Subscriptions Pot at the beginning of each month). Monzo also allows you to create custom push notifications to your mobile app which can be customised to your liking within the pipeline. 

### Pre-requisites

In order to run this locally you'll need the following

- Monzo Current Account (will be adding in Plaid API soon for other banks)
- Git
- Docker Desktop 
- Docker Compose

At least 4 GB of RAM is recommended to run the docker containers due to Airflow.

### Architecture

{{< figure src="/images/de_project_architecture.png" >}}

### Project Structure

{{< figure src="/images/project_structure_1.png" width=500 >}}
{{< figure src="/images/project_structure_2.png" width=500 >}}

- Makefile contains shorthand Docker Compose commands to run the Docker containers, as well as creating and giving access permissions to directories and running tests.
- To view Great Expectations data docs we spin up a minimal Jupyter Notebook server in its own container, using the jupyter/minimal-notebook image https://hub.docker.com/r/jupyter/minimal-notebook. It is accessible via a browser on your host machine at localhost:8888. The jupyter-config.json file configures the notebook server with default password 'changeme'. If you clone the repository, you can add the "uncommitted" directory in the "great_expectations" folder to the *.gitignore* file to keep your data docs from committing and prevent any secrets from being stored on Github.
- In the dags directory we store the common python files that are used in the DAGs and add the folder to the .airflowignore file so that Airflow does not to import it as a DAG. Contains *categories.py* file where Monzo Plus users can add names for custom categories used in their app (Monzo custom categories appear as ID's in the data, so will be mapped here).
- The scripts folder holds SQL scripts that are executed at database creation and JSON scripts that are generated and used within the DAGs.
- Pre commit hooks are used to fix formatting with Black and linting with flake8. The *setup.cfg* file is used to ignore some flake8 errors such as max line length.
- Metabase is the visualisation tool we will use and is part of the Docker Compose services, accessible at localhost:3000.

### Setup

First you'll need to clone the [repository](https://github.com/eliasbenaddou/personal_finance_de_project) and create a new *.env* file in the root of the project to add the below environment variables into, adding your own details where required. This will serve both the Docker Compose file and is also pushed to the containers for use within them. 

For example, the Postgres login details will be used in the Docker Compose file for the database service and will also be used within the Python operators in the DAGs running on the Airflow container.

```shell
POSTGRES_USER=postgres
POSTGRES_PASSWORD=postgres
POSTGRES_DB=transactions
POSTGRES_HOST=transactions-warehouse
POSTGRES_PORT=5432
MONZO_SCHEMA=monzo
MONZO_TABLE=monzo_transactions

MONZO_CLIENT_ID=
MONZO_CLIENT_SECRET=
MONZO_REDIRECT_URL=http://127.0.0.1/monzo
MONZO_TOKENS=/opt/airflow/scripts/json/monzo_tokens.json
MONZO_TRANSACTIONS_JSON=/opt/airflow/scripts/json/monzo_transactions.json
PAST_WEEK_MONZO_TRANSACTIONS_JSON=/opt/airflow/scripts/json/past_week_monzo_transactions.json

SMTP_ADDRESS=smtp.gmail.com
SMTP_PORT=465
SENDER_LOGIN=
SENDER_PASSWORD=
SENDER_ALIAS=GreatExpectations
RECEIVER_EMAILS=
```
---

1) Postgres Database

These can be kept as default or you can change them as you wish. Ensure your "POSTGRES_HOST" value matches the name of the service in the Docker Compose file (line 100).

The Postgres container has a mapped volume for executing the docker entry point SQL scripts that create the Monzo schema and table for storing the transactions. Add to the *.env* file the schema and table name and ensure these are the same with what exists already in the sql scripts (these can be changed).

2) Monzo client 

In order to make API calls to Monzo endpoints, you will need to create a "Confidential" client on Monzo's developer site. Navigate to https://developers.monzo.com and sign in using your email connected to your Monzo account.

You will be asked to login via the link in your email sent by Monzo and you will be taken to the API Playground. Before being able to use it, you will need to approve the notification on your Monzo app to give access to the developer portal to your account. 

In the top navigation bar, go to "Clients" and select "New OAuth Client". Give it a name and description, set the "Redirect URL" to "http://127.0.0.1/monzo" and set the "Confidentiality" field to "Confidential". You'll now be able to copy to the Client ID, Client Secret and Redirect URL into your *.env* file. 

As part of the pipeline, the Airflow tasks will read and save JSON files, including a file to store your tokens to make API calls to Monzo (these are different to the Client details above and are automatically refreshed as needed). These JSON files are saved in the container's *opt/airflow* directory.

To generate access and refresh tokens, run the notebook 'monzo_generate_tokens.ipynb' and follow the instructions. Save the access token, refresh token and expiry into a file called "monzo_tokens.json" in a dictionary format within the "scripts" directory as this is the directory that will be searched for tokens when the pipelines run.

3) Email notifications

As part of the Great Expectations implementation, you can receive email notifications for the  data quality checking pipeline by adding your email details. With Gmail accounts, you'll need to first create a Google App password at https://security.google.com/settings/security/apppasswords to avoid using your regular password or two factor authentication. Fill in your email, app password and email to receive notifications. 

With all of this setup, you're now ready to go - from the root directory run the following command to start the Docker containers:

```shell
make up
```

The first run will take a little while to pull the containers and Airflow should take a couple minutes to initialise. Once running, you'll be able to launch the Airflow UI on your browser by going to http://localhost:8080 and login with the default username and password 'airflow'. You can take down the services using the command

```shell
make down
```

or if you would like to delete any data associated by removing the volumes, run

```shell
make down-volumes
```


### DAG Details

{{< figure src="/images/airflow_ui.png" caption="Main menu showing the DAGs paused once logged into the Airflow UI" >}}

Some DAGs have schedules already set to daily or weekly and some have no schedule as they will be triggered by other DAGs. Here are descriptions of each DAG:

### monzo_dq_checks

- Triggered by monzo_transactions or monzo_transactions_history
- Runs a Great Expectations Checkpoint against a batch request. The validation runs the 'monzo_transactions_dq_checks' Expectation Suite which holds the defined Expectations. 
- The Great Expectations folder contains all the files that store the data docs and information about each validation. 
- This Expectation Suite currently checks if the 'notes' column in the database view 'v_monzo_transactions' is null. This would mean that the underlying database table is missing either "notes" information or "suggested tags", which comes from Monzo's metadata. 

### monzo_pot_budget_allocation

- Automates the allocation of funds on a scheduled interval from your Monzo current account into your Pots. 
- Requires dags/get_monzo_transactions/pot_budget.py populated. You can find Pot ID's in transaction records in the database if a transaction is a transfer between the Pot and your main account. 
- Checks your current Pot balance and moves funds from your main account until the balance in your Pot is the same as the budget amount defined in *dags/get_monzo_transactions/pot_budget.py*.

### monzo_transactions

- Sends requests to Monzo's API using [monzo-api](https://github.com/petermcd/monzo-api) to fetch transactions from the past 90 days only.
- Checks against the database to either append or update records if they are new transactions or if they have changed (e.g. if a transaction has changed in amount or metadata is added since last pull).

### monzo_transactions_history

- One time historical loader to fetch all transactions on your account
- Requires a manual step of re-authentication via your Monzo app to fetch all your transactions. On your Monzo app, go to Privacy & Security -> Manage Apps and refresh permissions for your client.
- Will need to run the DAG soon after refreshing permissions as Monzo allows 5 minutes for querying transactions that are older than 90 days.

### monzo_weekly_notifications

- Triggers a push notification to your Monzo app with a weekly spending update message viewable on the app with information about your top spending category and total amount spent in the past week.
- Can be customised and can include external links or gifs.

## Visualisation 

Navigate to localhost:3000 to access the UI for Metabase. You will need to setup a user and connect your transactions Postgres database using the same details from your *.env* environment file. You can then create your own visualisations and or the preset explorations from Metabase suggestions as a start.

{{< figure src="/images/metabase_ui.png" caption="Example charts made by Metabase on transactions table" >}}

## Roadmap

- Add in Plaid's API to allow connecting to other banks through Plaid's open banking API solution.

Thanks for reading and please reach out via Github or Linkedin if you have any questions or suggestions!

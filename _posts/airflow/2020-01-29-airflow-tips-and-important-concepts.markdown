---
layout: post
category: airflow
date: 2020-01-29
title: Airflow Tips and Important Concepts
image: /assets/airflow-wide.png
permalink: /airflow/airflow-tips-concepts
redirect_from: /airflow-tips-concepts
---
<div class="wide-logos" markdown="1">
![airflow](/assets/airflow.png)
</div>


# Introduction

I recently had a session with my colleagues at Strands Labs where we have went through our DAGs and pinpointed key constructs we are using. 

Please note that we are, currently, not running Airflow in a distributed manner. That should change in the future due to the company's involvement in developing a proprietary AI Cloud Platform.

Despite majority of the stuff I have written is available online, especially through an incredibly-well organized and concise Airflow documentation, I am sharing what we found useful.

## Requirements

You know the drill, a virtual environment with Python 3.6+ and the following:

```bash
pip install airflow
pip install psycopg2
pip install pandas
```

So, let's start!

## Using DAG inside of a Context Manager

We will use DAG inside of a Context Manager to avoid assigning task to the DAG, in every creation of a TaskInstance.

```python
from datetime import datetime, timedelta

from airflow import DAG
from airflow.operators.dummy_operator import DummyOperator
```

    [2020-01-23 14:18:46,891] {settings.py:182} INFO - settings.configure_orm(): Using pool settings. pool_size=5, pool_recycle=1800, pid=7844

We have two options:

```python
# Using a Context Manager
DAG_ID = 'dummy_dag'

with DAG(dag_id=DAG_ID,
         start_date = datetime(2020, 1, 1),
         catchup=False,
         schedule_interval='@daily') as dag:

    start_task = DummyOperator(task_id='start')
```

```python
# Not using a Context Manager
DAG_ID = 'dummy_dag'

dag = DAG(dag_id=DAG_ID,
          start_date = datetime(2020, 1, 1),
          catchup=False,
          schedule_interval='@daily')

start_task = DummyOperator(task_id='start', dag=dag)
```

Why you should use a Context Manager?
*  Automatically assign new TaskInstances to the DAG
*  Avoid repeating `dag=dag` in each call to the TaskInstance constructor

## Use default arguments when instatiating DAG

`default_args` parameter in the DAG constructor receives a dictionary that will be used by all the TaskInstances, instead of passing same set of parameters to different TaskInstances.


```python
DAG_ID = 'dummy_dag'

default_args = {
    'owner': 'airflow',
    'email': 'nous@labs.com',
    'email_on_failure': True,
    'retries': 2,
    'retry_delay': timedelta(minutes=5),
    'provide_context': True,
    'stupid_var': 123
    }

with DAG(dag_id=DAG_ID,
         start_date = datetime(2020, 1, 1),
         catchup=False,
         schedule_interval='@daily',
         default_args=default_args) as dag:

    start_task = DummyOperator(task_id='start')
```

Why you should use a `default_args`?
*  Avoid repeating common parameters/arguments in each call to the TaskInstance constructor

## Variables

Airflow provides you with the notion of `Variable`, essentially a key-value store inside the Airflow "metabase"

```python
from airflow.models import Variable
```

Creating a Variable record:

Use `Variable.set()` to create new records.


```python
VARIABLES = {
    'FM_PROJECT': 'moneystrands',
    'NOUS_EMAIL': 'nous@labs.com',
    'RANDOM_VAR': 'random_var'
}

for key, value in VARIABLES.items():
    Variable.set(key=key, value=value)
```

    [2020-01-23 14:24:06,821] {__init__.py:183} WARNING - empty cryptography key - values will not be stored encrypted.


How it looks in the Airflow metabase?

We can check the values of the records in a database table `variable`.


```python
import psycopg2 as pg
import pandas as pd

conn = pg.connect(database='airflow', 
                  user='airflow',
                  password='a1rfl0w')
```


```python
df = pd.read_sql("SELECT * FROM variable WHERE key = 'STUPID_VAR'", conn)
df.head()
```

<div>
<style scoped>
    .dataframe tbody tr th:only-of-type {
        vertical-align: middle;
    }

    .dataframe tbody tr th {
        vertical-align: top;
    }

    .dataframe thead th {
        text-align: right;
    }
</style>
<table border="1" class="dataframe">
  <thead>
    <tr style="text-align: right;">
      <th></th>
      <th>id</th>
      <th>key</th>
      <th>val</th>
      <th>is_encrypted</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <th>0</th>
      <td>147</td>
      <td>RANDOM_VAR</td>
      <td>random_var</td>
      <td>False</td>
    </tr>
  </tbody>
</table>
</div>


Referencing a Variable record:

Use `Variable.get()` to check the values of records.


```python
random_var = Variable.get('RANDOM_VAR')
print(random_var)
```

    random_var

Why you should use a `Variable`?
*  Avoid having configuration file or saving key-value pairs in the environment variables


Caution?
Each call to the class method `Variable.get_value()` or `Variable.get()` is a call to the DB (open a session, make call), therefore, one potential usage would be to call it just once in the `default_args` dictionary and reuse it everywhere when needed.


## Connections


In order to reduce defining connections in the code, Airflow provides you with the `Connection` element, where you can define various objects to different datastores

```python
from airflow.models import Connection
from airflow.utils.db import merge_conn
```

Creating a Connection record:

```python
# WebHDFS Connection
merge_conn(
    Connection(
        conn_id=f'{Variable.get('FM_PROJECT')}',
        conn_type='HDFS',
        host='hw.nouslabs.com',
        port=50070
        )
    )
```

How it looks in the Airflow metabase?

```python
df = pd.read_sql("SELECT * FROM connection WHERE conn_id = 'hdfs_moneyhb_adm'", conn)
df.head()
```

<div>
<style scoped>
    .dataframe tbody tr th:only-of-type {
        vertical-align: middle;
    }

    .dataframe tbody tr th {
        vertical-align: top;
    }

    .dataframe thead th {
        text-align: right;
    }
</style>
<table border="1" class="dataframe">
  <thead>
    <tr style="text-align: right;">
      <th></th>
      <th>id</th>
      <th>conn_id</th>
      <th>conn_type</th>
      <th>host</th>
      <th>schema</th>
      <th>login</th>
      <th>password</th>
      <th>port</th>
      <th>extra</th>
      <th>is_encrypted</th>
      <th>is_extra_encrypted</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <th>0</th>
      <td>55</td>
      <td>moneystrands</td>
      <td>hdfs</td>
      <td>hw.nouslabs.com</td>
      <td>None</td>
      <td>None</td>
      <td>None</td>
      <td>50070</td>
      <td>None</td>
      <td>False</td>
      <td>False</td>
    </tr>
  </tbody>
</table>
</div>


How to use it?

Most of the `hooks` have the parameter `{hook_name}_conn_id` where you provide the connection.

```python
from airflow.hooks.webhdfs_hook import WebHDFSHook

hook = WebHDFSHook(webhdfs_conn_id='moneystrands')
```


```python
hook.check_for_path('/warehouse/tablespace/external/hive/')
```

    [2020-01-23 14:33:39,942] {client.py:192} INFO - Instantiated <InsecureClient(url='http://hortonworks.strands.com:50070')>.
    [2020-01-23 14:33:39,944] {client.py:312} INFO - Fetching status for '/'.
    [2020-01-23 14:33:39,950] {client.py:312} INFO - Fetching status for '/warehouse/tablespace/external/hive/'.

    True


## Create things dynamically


Use Python control flow statements to reduce development time.

```python
from collections import OrderedDict
```


```python
TABLES_TO_IMPORT = OrderedDict(
                       {'users':
                           OrderedDict(
                               {'usr_id': 'INT',
                                'usr_providerid': 'STRING',
                                'usr_tenure': 'INT'}),
                        'user_profile':
                           OrderedDict(
                               {'usr_id': 'INT',
                                'upr_income': 'DOUBLE',
                                'upr_numchildren': 'INT'}),
                        'categories':
                           OrderedDict(
                               {'cat_id': 'INT',
                                'cat_name': 'STRING'})})
```


```python
with DAG(dag_id=DAG_ID,
         start_date = datetime(2020, 1, 1),
         catchup=False,
         schedule_interval='@daily') as dag:

    for table in TABLES_TO_IMPORT.keys():
        dummy_task = DummyOperator(
            task_id=f'recreate_table_{table}_task')
```

```python
dag.tasks
```

    [<Task(DummyOperator): recreate_table_users_task>,
     <Task(DummyOperator): recreate_table_user_profile_task>,
     <Task(DummyOperator): recreate_table_categories_task>]

Why you should use Python control flow statements?
*  Reduce development time
*  Increase productivity and readability

## Trigger other DAGs


We can trigger other DAGs and pass some data to them, a useful concept when there are dependencies between different workflows (think about some).

```python
from airflow.operators.dagrun_operator import TriggerDagRunOperator
```

```python
DAG_ID = 'dummy_dag'
DOWNSTREAM_DAG_ID = 'dummy_downstream_dag'
```

```python
def dummy(context, dag_run_obj, **kwargs):
    dag_run_obj.payload = {
        'param1': context['params']['param1'],
        'param2': context['params']['param2']}
    
    return dag_run_obj

with DAG(dag_id=DAG_ID,
         start_date = datetime(2020, 1, 1),
         catchup=False,
         schedule_interval='@daily') as dag:

    trigger_task = TriggerDagRunOperator(
        task_id='dummy_task_trigger',
        trigger_dag_id=DOWNSTREAM_DAG_ID,
        python_callable=dummy,
        # Params dictionary overrides parameters from the DAG level
        params={'param1': 123,
                'param2': 345})
```

Why you should use `TriggerDagRunOperator`?

I can think of various things that are useful, but I find this one quite useful. Most of the times, you will have dependencies between tasks. That is, when one DAG finishes, another one has to be triggered.

Example would be, you make predictions and would like to serve them, but through another workflow. You need to trigger that workflow.

## `context` dictionary

Available through `PythonOperator` to your Python callables. Cue the usage in the example above, we pass the `params` dictionary in the `TriggerDagRunOperator` constructor and access it through `context` dictionary in the callable.

## `execution_date` mystery

Airflow has a strict dependency on a specific time: `the execution_date`. 

No DAG can run without an `execution_date`, and no DAG can run twice for the same `execution_date`. Do you have a specific DAG that needs to run twice, with both instantiations starting at the same time? Airflow doesn’t support that; there are no exceptions. Airflow simply decrees that such workflows do not exist. You’ll need to create two nearly-identical DAGs, or start them a millisecond apart, or employ other creative hacks to get this to work.

Additionally, the `execution_date` is not interpreted by Airflow as the start time of the DAG, but rather the end of an interval capped by the DAG’s start time. This was originally due to ETL orchestration requirements, where the job for May 2nd’s data would be run on May 3rd. Today, it is a source of major confusion and one of the most common misunderstandings new users have.

## Song of the Week

This post was dedicated to Kobe Bean Bryant, my childhood hero that tragically passed away on Sunday 26th of January 2020.

<iframe width="100%" height="166" scrolling="no" frameborder="no" allow="autoplay" src="https://w.soundcloud.com/player/?url=https%3A//api.soundcloud.com/tracks/750846412&color=%234838b8&auto_play=false&hide_related=false&show_comments=true&show_user=true&show_reposts=false&show_teaser=true"></iframe>

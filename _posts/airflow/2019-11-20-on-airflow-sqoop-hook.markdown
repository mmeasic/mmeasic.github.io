---
layout: post
category: airflow
date: 2019-11-20
title: On Airflow's Sqoop Hook
image: /assets/airflow-wide.png
permalink: /airflow/on-airflow-sqoop-hook
redirect_from: /on-airflow-sqoop-hook
---
<div class="wide-logos" markdown="1">
![airflow](/assets/airflow.png)
</div>

## That Guy Airflow
Airflow is a great tool. However, any great tool, if used in a wrong way, can become a problem. Something like having a bike in a city without bike lines or a car in a city where traffic jams are a big thing. I don’t know why I used this example, but you get the point. 

Poor Airflow, it was designed as an orchestrator and ended up being used as a data manager on steroids, able to talk with “anyone”, issue and execute commands, pass data around and what not. But I digress, let’s talk about Sqoop Hook and the current implementation.

## That Guy Sqoop
Apache Sqoop is a tool designed for efficiently transferring bulk data between Apache Hadoop and structured data-stores such as relational databases. Simple as that. You have a source (your relational database) and you have a target (HDFS/Hive). Obviously, you want to move some data from source to target. Sqoop will allow you to do that, seamlessly. Again, I will not discuss the weakness of Apache Sqoop and how its “batch-like” jobs are kind of 2015ish.

## The Fuss
So, Airflow allows you to define [connections][1] that can be used by hooks (interfaces to interact with external systems). [SqoopHook][2] is provided through Airflow, allowing you to move data between aforementioned source and target.

OK, Mario, what is so special about it? Why are you writing an article just to state facts that I could’ve looked up myself?

Well, let’s put it nicely, the hook is not working.

In order to connect to a database server, one has to provide a connect-string, as Sqoop defines it.

```bash
sqoop import --connect jdbc:mysql://database.example.com/employees --username --password 12345
```

In order to connect, Sqoop needs a JDBC driver and a JDBC connector. Different databases require different JDBC URLs. In case you would like to know more about the JDBC URLs for different databases, check this [link][3].

All good. Let's see how [SqoopHook][2] defines the connect string we saw above, just an excerpt from the `_prepare_command()` method.

```python
connect_str = self.conn.host
if self.conn.port:
    connect_str += ":{}".format(self.conn.port)
if self.conn.schema:
    connect_str += "/{}".format(self.conn.schema)
connection_cmd += ["--connect", connect_str]
```

**JDBC URL is non-existent. This will not work.**

## How To Fix It?
Obviously, you can solve this changing the [SqoopHook class][2].
The way I see it, you need to adjust your Connection objects to include JDBC URL, which should be different for different types of connections.

I provide an excerpt for how to define Connection object that will later be used by the hooks.

```python
from os import getenv

from airflow.models import Connection
from airflow.models import Variable
from airflow.utils.db import merge_conn
from dotenv import load_dotenv

# Oracle RDBMS Connection
merge_conn(
    Connection(
        conn_id=getenv('RDBMS_CONN_ID'),
        conn_type=getenv('RDBMS_CONN_TYPE'),
        host=getenv('RDBMS_HOST'),
        schema=getenv('RDBMS_SCHEMA'),
        login=getenv('RDBMS_USER'),
        password=getenv('RDBMS_PWD'),
        port=getenv('RDBMS_PORT'),
        {% raw %}
        extra=f"""{{"dsn": "{getenv('RDBMS_HOST')}",
                    "service_name": "{getenv('RDBMS_SERVICE_NAME')}",
                    "sid": "{getenv('RDBMS_SERVICE_NAME')}",
                    "jdbc_url": "jdbc:oracle:thin:@{getenv('RDBMS_HOST')}:{getenv('RDBMS_PORT')}/{getenv('RDBMS_SERVICE_NAME')}"}}"""
        {% endraw %}
        )
    )
```

Here, I provide the example for the Oracle database since that’s the one I’m using in my day-to-day activities. You can adjust it to the RDBMS of your choice following the link provided above.

Then, you modify the `sqoop_hook.py` file and redefine the [SqoopHook class'][2] methods.

You simply need to modify the constructor `__init__()` with adding the following line:

```python
self.jdbc_url = connection_parameters.get('jdbc_url', None)
```

Finally, you modify the `_prepare_command()` method:

```python
if self.jdbc_url:
    connect_str = f"{jdbc_url}"
else:
    raise ValueError("provide a valid JDBC url.")
```

That's it. You fixed the Sqoop Hook and you can create a PR for the Apache Airflow. Have a drink, relax and enjoy the life.

## Song of the Week

<iframe width="560" height="315" src="https://www.youtube.com/embed/4hZ_wTx_kWg" frameborder="0" allow="accelerometer; autoplay; encrypted-media; gyroscope; picture-in-picture" allowfullscreen></iframe>

[1]: https://github.com/apache/airflow/blob/6afb12f0e5c18e8634daa0119d6e5797aa770b80/airflow/models/connection.py
[2]: https://github.com/apache/airflow/blob/6afb12f0e5c18e8634daa0119d6e5797aa770b80/airflow/contrib/hooks/sqoop_hook.py
[3]: https://vladmihalcea.com/jdbc-driver-connection-url-strings/
---
layout: post
title:  "Custom Airflow Timetables work only with Cloud Composer 2"
date:   2022-05-13 10:08:10 +0900
categories: airflow
---
# TLDR

You need to use Cloud Composer 2 (not 1) to install and use your custom Airflow Timetable.

# What is Airflow Timetable

[Airflow Timetable][timetable] is a new concept introduced in Airflow 2.2. You can [customize DAG scheduling with your custom Timetable][custom-timetable] to create your own custom schedule using Python. It allows you to define your own schedules in Python code.

# How to use your custom Timetable with Cloud Composer 2

By the steps below, you can install your custom Timetable as an Airflow plugin and use it for you DAGs:
1. Create your Cloud Composer 2 environment
2. Create your custom Timetable .py file locally. Say [`workday_timetable.py`][workday.py] in this example.
3. Install the Timetable as an Airflow plugin to the Cloud Composer environment by:
    ```shell
    gcloud composer environments storage plugins import \
      --environment YOUR_CLOUD_COMPOSER \
      --location YOUR_LOCATION \
      --source /path/to/your/workday_timetable.py
    ```
4. Create your DAG python script using the Timetable. Say `workday_dag.py` in this example:
    ```python
    import os

    import pendulum
    from airflow import DAG
    from airflow.operators.dummy import DummyOperator

    from workday_timetable import AfterWorkdayTimetable

    with DAG(
        os.path.basename(__file__).replace('.', '_'),
        start_date=pendulum.datetime(2021, 1, 1),
        catchup=False,
        timetable=AfterWorkdayTimetable(cron='@daily', timezone=pendulum.timezone("UTC"))
    ) as dag:
        DummyOperator(task_id="dummy")
    ```
5. Upload the DAG python script to the Cloud Composer environment by:
    ```shell
    gcloud composer environments storage dags import \
      --environment YOUR_CLOUD_COMPOSER \
      --location YOUR_LOCATION \
      --source /path/to/your/workday_dag.py
    ```

# It doesn't work with Cloud Composer 1

Taking the same steps on Cloud Composer 1, you'll find the DAG works properly, however, you can't open the DAG detail by the Web UI. Instead you'll an error page with a stacktrace indicating the Timetable class is not registered.

It's because Airflow workers and schedulers loaded the custom Timetable. However, Airflow webserver doesn't have access to the Timetable imported in the GCS bucket. It's because Airflow webserver is running on GAE (Google App Engine) in Cloud Composer 1.

Google Cloud provides the Cloud Composer environment architecture of [Cloud Composer 1][arch-composer1] and [Cloud Composer 2][arch-composer2] here.

[workday.py]: https://airflow.apache.org/docs/apache-airflow/2.2.0/_modules/airflow/example_dags/plugins/workday.html
[timetable]: https://airflow.apache.org/docs/apache-airflow/2.2.0/concepts/timetable.html
[custom-timetable]: https://airflow.apache.org/docs/apache-airflow/2.2.0/howto/timetable.html
[arch-composer1]: https://cloud.google.com/composer/docs/concepts/architecture
[arch-composer2]: https://cloud.google.com/composer/docs/composer-2/environment-architecture

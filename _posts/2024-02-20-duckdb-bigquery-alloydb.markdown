---
layout: post
title: "Moving data from BigQuery to AlloyDB using DuckDB and Airflow"
date: 2024-02-20 13:00:00 +0200
categories: bigquery alloydb duckdb 
---
I had a requirement to export data from a BigQuery table to AlloyDB recently. BigQuery is a powerful datawarehouse, but querying it often can be costly, and if you need to be able to run low-latency analytical queries with predictable access patterns, such as a dashboard built in Grafana, BigQuery can be limiting as well. In these cases, AlloyDB might be a better choice. 

AlloyDB is Google's Postgres-compatible HTAP database, and includes an in-memory columnar engine for analytical workloads. The default number of connections is 1000, but this can scale up to an impressive 240,000.

The problem I had is that my data was already in BigQuery, so I needed a way to export that data and insert it into AlloyDB. BigQuery can export data in Parquet format to Google Cloud Storage, but AlloyDB (being a Postgres-compatible database) doesn't natively support reading from Parquet files. This is where DuckDB comes to the rescue. DuckDB can not only read Parquet data from GCS, it can also attach to a Postgresql database and insert data into a table directly. And combining this with Airflow, you can schedule this export and import on a regular basis.

# Creating a table in AlloyDB
There are lots of things to consider when creating the table in AlloyDB, such as partitioning based on a timestamp using `pg_partman`, but I'm not going into those details here. What's important to keep in mind is the conversion from BigQuery data types to Postgres data types (via Parquet.) This can be tricky. The BigQuery [documentation](https://cloud.google.com/bigquery/docs/exporting-data#parquet_export_details) has a conversion table for BigQuery to Parquet. My table didn't have any complex data types (such as nested and repeated fields), but it did have STRING columns which needed to be converted to VARCHAR(n) in AlloyDB, where n is the length limit (number of characters) it can store. So I had to do some exploration in BigQuery to find out what the maximum number of characters for each STRING column was.

Be extra careful with FLOAT64 and NUMERIC columns in BigQuery. These will have to be converted to decimal or numeric columns in AlloyDB and you don't want to lose any precision.

Creating the table in AlloyDB is straightforward. You can use a tool like `psql` or `pg_admin` to run a `CREATE TABLE` statement:

```sql
CREATE TABLE IF NOT EXISTS schema.table
(
    string_col character varying(32) NOT NULL,
    ts timestamp without time zone NOT NULL,
    metric integer
);
```

# Preparing the Airflow environment
We will be using DuckDB to read from Google Cloud Storage, so the Airflow workers need to have the correct Python packages. The packages are: `duckdb`, `fsspec` and `gcsfs`. If you're on Cloud Composer, then `fsspec` and `gcsfs` are already available. You just need to add `duckdb` as a dependency.

Also, be sure to add the AlloyDB connection to the Airflow Connections. Create a new connection of type Postgres and fill in the details: host, schema, login, password, port.

# Exporting the BigQuery table in Airflow
Exporting an entire table to Google Cloud Storage using Airflow can be done with the `BigQueryToGCSOperator`:

```python
from airflow.providers.google.cloud.transfers.bigquery_to_gcs import BigQueryToGCSOperator

bigquery_to_gcs = BigQueryToGCSOperator(
    task_id="bigquery_to_gcs",
    source_project_dataset_table=f"{DATASET_NAME}.{TABLE}",
    destination_cloud_storage_uris=[f"gs://{BUCKET_NAME}/{BUCKET_FILE}"],
    export_format="PARQUET"
)
```

Of course, the table shouldn't be too big or else DuckDB will run out of memory. If you run into this issue, you should extract the data out of BigQuery in increments. This can be done with the `BigQueryInsertJobOperator` in which you can run a `EXPORT DATA` query to export only the specific increment.

# Loading the Parquet data into AlloyDB using DuckDB
To do this, we will create a function with will be called by the `PythonOperator`:

```python
from airflow.hooks.base_hook import BaseHook
from airflow.operators.python import PythonOperator

def gcs_to_alloydb(conn_id, **kwargs):
    import duckdb
    from fsspec import filesystem

    conn_info = BaseHook.get_connection(conn_id)
    conn = duckdb.connect(":memory:")
    conn.execute("INSTALL postgres")

    fs = filesystem("gcs")
    conn.register_filesystem(fs)

    conn.execute(f"ATTACH '{conn_info.get_uri()}' AS alloydb (TYPE postgres)")
    conn.execute("INSERT INTO alloydb.schema.table SELECT * FROM read_parquet('gcs://bucket/path/to/*.parquet')")

gcs_to_alloydb = PythonOperator(
    task_id="gcs_to_alloydb",
    python_callable=gcs_to_alloydb,
    op_args=["conn_id"],
    provide_context=True
)


bigquery_to_gcs >> gcs_to_alloydb
```

Note: The above is just an example of how it could work. In reality, running this every day may cause duplicate data in your AlloyDB table. You may want to truncate the table every time before the INSERT statement, or something like that, to prevent duplicates.

Also be aware that the above example might not be 100% safe from SQL injection. But as long as you don't work with user input the risk of this should be minimal.

# Conclusion

DuckDB can be a powerful tool for basic ETL pipelines. While this use case might be very specific, the above example can be easily generalized. For example, you may have some Parquet data coming from a different source, that you want to insert into a normal Postgresql database. DuckDB can also handle CSV files, and attach to MySQL databases.

I hope you found this article helpful!

---
title: Dockerizing  
date: last-modified
description: Dockerizing the Ingestion Script
---

# Dockerizing the Ingestion Script

## Terminology

- `pgAdmin`: PGAdmin is a web-based Graphical User Interface (GUI) management application used to communicate with Postgres and derivative relational databases on both local and remote servers.
- `Postgres`: is a powerful, open source object-relational database system that uses and extends the SQL language combined with many features that safely store and scale the most complicated data workloads.
- `Docker networking`: Docker networking enables a user to link a Docker container to as many networks as the user requires. 
- `psychopg2`: is the most popular PostgreSQL database adapter for the Python programming language.

## Useful links

- pgAdmin [official page](https://www.pgadmin.org/).

## Converting the notebook to a python script

We could do something like:

```bash
jupyter nbconvert --to=script upload-data.ipynb
```
to convert the notebook, where we download the data, to a python file. But that's to slow, so we are just gonna create a new file called `ingest_data.py`, and we will use the  `argparse` library to pass some cli arguments:

```python
#Cleaned up version of data-loading.ipynb
import argparse, os, sys
from time import time
import pandas as pd 
import pyarrow.parquet as pq
from sqlalchemy import create_engine


def main(params):
    user = params.user
    password = params.password
    host = params.host
    port = params.port
    db = params.db
    tb = params.tb
    url = params.url
    
    # Get the name of the file from url
    file_name = url.rsplit('/', 1)[-1].strip()
    print(f'Downloading {file_name} ...')
    # Download file from url
    os.system(f'curl {url.strip()} -o {file_name}')
    print('\n')

    # Create SQL engine
    engine = create_engine(f'postgresql://{user}:{password}@{host}:{port}/{db}')

    # Read file based on csv or parquet
    if '.csv' in file_name:
        df = pd.read_csv(file_name, nrows=10)
        df_iter = pd.read_csv(file_name, iterator=True, chunksize=100000)
    elif '.parquet' in file_name:
        file = pq.ParquetFile(file_name)
        df = next(file.iter_batches(batch_size=10)).to_pandas()
        df_iter = file.iter_batches(batch_size=100000)
    else: 
        print('Error. Only .csv or .parquet files allowed.')
        sys.exit()


    # Create the table
    df.head(0).to_sql(name=tb, con=engine, if_exists='replace')


    # Insert values
    t_start = time()
    count = 0
    for batch in df_iter:
        count+=1

        if '.parquet' in file_name:
            batch_df = batch.to_pandas()
        else:
            batch_df = batch

        print(f'inserting batch {count}...')

        b_start = time()
        batch_df.to_sql(name=tb, con=engine, if_exists='append')
        b_end = time()

        print(f'inserted! time taken {b_end-b_start:10.3f} seconds.\n')
        
    t_end = time()   
    print(f'Completed! Total time taken was {t_end-t_start:10.3f} seconds for {count} batches.')    



if __name__ == '__main__':
    #Parsing arguments 
    parser = argparse.ArgumentParser(description='Loading data from .parquet file link to a Postgres datebase.')

    parser.add_argument('--user', help='Username for Postgres.')
    parser.add_argument('--password', help='Password to the username for Postgres.')
    parser.add_argument('--host', help='Hostname for Postgres.')
    parser.add_argument('--port', help='Port for Postgres connection.')
    parser.add_argument('--db', help='Databse name for Postgres')
    parser.add_argument('--tb', help='Destination table name for Postgres.')
    parser.add_argument('--url', help='URL for .paraquet file.')

    args = parser.parse_args()
    main(args)
```
We can go to `pgAdmin` and delete the table by doing:

```SQL
DROP TABLE 
    yellow_taxi_data;
```

And now we don't have any data, so if we run our script by doing:

```bash
python3 ingest_data.py \
    --user=root \
    --password=root \
    --host=localhost \
    --port=5432 \
    --db=ny_taxi \
    --tb=yellow_taxi_trips \
    --url="https://d37ci6vzurychx.cloudfront.net/trip-data/yellow_tripdata_2021-01.parquet" 
```

And if we go to `pgAdmin` and refresh and run:

```SQL
SELECT 
    COUNT(1) 
FROM 
    yellow_taxi_trips;
```

we will see all our data again.

## Dockerizing the script

For this, we need to change our `Dockerfile` we have in our directory to:

```dockerfile
FROM python:3.9

# We need to install wget to download the parquet file
RUN apt-get install wget
# psycopg2 is a postgres db adapter for python: sqlalchemy needs it
RUN pip install pandas sqlalchemy psycopg2 pyarrow

WORKDIR \app

COPY ingest_data.py ingest_data.py 

ENTRYPOINT [ "python", "ingest_data.py" ]
```
and we can build this doker image by doing:

```bash
docker build -t taxi_ingest:v001 .
```

remember here:

- `build` is to build the image from a Dockerfile.
- `-t` Name and optionally a tag in the `name:tag` format
- `.` builds the image in the actual repository

and we run the image by doing:

```bash
docker run -it \
    --network=pg-network \
    taxi_ingest:v001 \
    --user=root \
    --password=root \
    --host=pg-database \
    --port=5432 \
    --db=ny_taxi \
    --tb=yellow_taxi_trips \
    --url="https://d37ci6vzurychx.cloudfront.net/trip-data/yellow_tripdata_2021-01.parquet" 
```
Few notes:

- We need to provide the network for Docker to find the Postgres container. It goes before the name of the image.
- Since Postgres is running on a separate container, the host argument will have to point to the container name of Postgres.
- You can drop the table in pgAdmin beforehand if you want, but the script will automatically replace the pre-existing table.


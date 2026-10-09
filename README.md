# MS SQL Server to Postgres conversions 

We have a number of applications where the data is held in a SQL Server database. It's useful for us to be able to convert those databases into Postgres databases (which we are happier with). This documents the current process. There might be a better way of doing this, but this does work, it's just how much time do we want to spend making a better way!

## Requirements

 
* Either a `.bacpac` or a `.bak` database dump of the SQL Server database.
* [Azure Data Studio](https://learn.microsoft.com/en-us/previous-versions/azure-data-studio/download-azure-data-studio) (since discontinued) installed on your computer
* Local postgres installation
* Docker installed
* [Docker SQL Server 2022 image](https://hub.docker.com/r/microsoft/mssql-server) - `docker pull mcr.microsoft.com/mssql/server:2022-latest`
* Set up the SQL Server instance - `docker run -e "ACCEPT_EULA=Y" -e "MSSQL_SA_PASSWORD=<password>" -p 1433:1433 -d mcr.microsoft.com/mssql/server:2022-latest`
* Fire up Azure Data Studio and see if you can connect to the SQL Server instance
* Create local postgres database using `createdb [your desired postgres database name]`

## Getting data into SQL Server

### Using a .bak file

If you are using a `.bak` file, then you need to copy it to the docker container with the DB in it and then use Azure Data Studio to restore the DB.

For the docker copy, you can do this, where `sql2022` is the name of the docker container containing the SQL Server DB:

`docker cp filename.bak sql2022:/var/opt/mssql/data/filenam.bak`

Then in Data Studio, right click on `Databases` and select the _Restore Database_ option. In the dialogue which then opens, change 'Restore from' to `backup file` - then you can follow the wizard through.

### Using a .bacpac file

You don't need to copy the file to the docker container, you can use the Data tier wizard. Right click on the server (Not databases) and select 'Data-tier application wizard' - you can then select bacpack from the available options and restore the file directly.

## Getting this data imported using docker pgloader

After much faffing about, running this docker command, to run [pgloader](https://github.com/dimitri/pgloader/) in the container, mounting the current directory as /work for the container, does the trick.

Have a `secrets.env` file in the _repo_ directory and add:

```
MSSQL_USERNAME=[by default this is sa]
MSSQL_PASSWORD=[this is what you set it to above]
POSTGRES_USER=[this is usually your OS username]
```

Then this is the command to run in the subdirectory where the `migrate.load`

```
docker run --rm --env-file secrets.env -v $(pwd):/work dimitri/pgloader pgloader /work/migrate.load
```

migrate.load looks like this for example - this is for deposited papers, note that:

- SQL Server database name is `DepositedPapers`
- Postgres database name is `ms-sql-deposited-papers-dump-October-2026`
- both dbs require host.docker.internal as the command itself is being run in a docker container
- postgres requires a username

```
LOAD DATABASE
FROM mssql://{{MSSQL_USERNAME}}:{{MSSQL_PASSWORD}}@host.docker.internal:1433/DepositedPapers
INTO postgresql://{{POSTGRES_USER}}@host.docker.internal:5432/ms-sql-deposited-papers-dump-October-2026

WITH include drop,
create tables,
create indexes,
reset sequences;
```

## Column casting

Sometimes you have weird colmns, so after the WITH section, you can use CAST to set them to varchars, for example

```
...
WITH include drop,
      create tables,
      create indexes,
      reset sequences

CAST column dbo.weirdcolumn to varchar;
```

## Verification

Run this in Azure to count table rows:

```

SELECT
s.name AS SchemaName,
t.name AS TableName,
p.rows
FROM sys.tables t
INNER JOIN sys.schemas s ON t.schema_id = s.schema_id
INNER JOIN sys.partitions p ON t.object_id = p.object_id
WHERE p.index_id IN (0, 1) -- 0 = heap, 1 = clustered index
ORDER BY p.rows DESC;

```

And this in Postgres to count converted rows

```

SELECT
schemaname,
relname AS table_name,
n_live_tup AS row_count
FROM pg_stat_user_tables
ORDER BY n_live_tup DESC;

```

# DuckDB Processor

The DuckDB processor is a component that allows running SQL queries on a DuckDB database. For more information about DuckDB, visit the [DuckDB Documentation](https://duckdb.org/docs/).

## Configuration

The component supports two **modes of operation**:
- Simple Mode (default)
- Advanced Mode

### Simple Mode
In simple mode, each query operates on a single table in DuckDB. This table can be created from a specific table defined by name or from multiple tables matching a pattern (along with an arbitrary number of files). 
The query output is exported using the name of the input table.

![simple.png](docs/imgs/simple.png)

In simple mode, the parameter **queries** is an array of queries to be executed. Each query has its own input, query, and output, and is executed in isolation.
Each query can use the following parameters:

- **input:** A string specifying the name of the table, or an object containing the following parameters:
  - **input_pattern** (required): The name of the table or a [glob pattern](https://duckdb.org/docs/data/multiple_files/overview#glob-syntax).
  - **duckdb_destination** (required when using a glob pattern in the previous parameter): The name of the table in DuckDB.
  - **dtypes_mode:** Determines how data types are handled. Options are:
    - all_varchar (default): Treats all columns as text.
    - auto_detect: Automatically infers data types.
    - from_manifest: Uses data types from the input manifest (if a wildcard is used, the manifest of the first table is used).
  - **skip_lines:** The number of lines to skip.
  - **delimiter:** A string specifying the delimiter.
  - **quotechar:** A string specifying the quote character.
  - **column_names:** A list of column names.
  - **date_format:** A string specifying the date format.
  - **timestamp_format:** A string specifying the timestamp format.
  - **add_filename_column:** A boolean indicating whether the filename should be added as a column.
- **query** (required): The query to be executed.
- **output:** A string specifying the output table name or an object containing the following parameters:
  - **kbc_destination:** The name of the output table.
  - **primary_key:** A list of primary keys.
  - **incremental:** A boolean indicating whether the data should be loaded incrementally.

#### Example configuration

```
{
    "before": [],
    "after": [
        {
            "definition": {
                "component": "keboola.processor-duckdb"
            },
            "parameters": {
                "mode": "simple",
                "queries": [
                    {
                        "input": "sales.csv",
                        "query": "SELECT sales_representative, SUM(turnover) AS total_turnover FROM sales.csv GROUP BY sales_representative"
                    }
                ]
            }
        }
    ]
}
```

```
{
    "before": [],
    "after": [
        {
            "definition": {
                "component": "keboola.processor-duckdb"
            },
            "parameters": {
                "mode": "simple",
                "queries":[
                  {
                    "input": {
                      "input_pattern": "/data/in/tables/*.csv",
                      "duckdb_destination": "products",
                      "dtypes_mode": "auto_detect",
                      "skip_lines": 1,
                      "delimiter": ",",
                      "quotechar": "\"",
                      "column_names": ["id", "name", "category_id"],
                      "date_format": "YYYY-MM-DD",
                      "timestamp_format": "YYYY-MM-DD HH:MM:SS",
                      "add_filename_column": true
                    },
                    "query" : "SELECT p.*, cat.name as category_name FROM products AS p LEFT JOIN '/data/in/files/categories.parquet' AS cat on p.category_id = cat.id ORDER BY p.id",
                    "output": {
                      "kbc_destination": "out-products",
                      "primary_key": ["id"],
                      "incremental": true
                    }
                  }]
            }
        }
    ]
}
```


### Advanced Mode
In advanced mode, [relations](https://duckdb.org/docs/api/python/relational_api) are created from all specified input tables.
Then, all defined queries are processed. Finally, the output tables specified in out_tables are exported to Keboola Storage.

![advanced.png](docs/imgs/advanced.png)

Parameters:
 - **input:** An array of input tables from Keboola, defined either as a string containing the name or as an object containing the following parameters:
    - **input_pattern** (required): The name of the table or a [glob pattern](https://duckdb.org/docs/data/multiple_files/overview#glob-syntax).
    - **duckdb_destination** (required when using a glob pattern in the previous parameter): The name of the table in DuckDB.
    - **dtypes_mode:** Determines how data types are handled. Options are: 
      - all_varchar (default): Treats all columns as text.
      - auto_detect: Automatically infers data types.
      - from_manifest: Uses data types from the input manifest (if a wildcard is used, the manifest of the first table is used).
    - **skip_lines:** The number of lines to skip.
    - **delimiter:** A string specifying the delimiter.
    - **quotechar:** A string specifying the quote character.
    - **column_names:** A list of column names.
    - **date_format:** A string specifying the date format.
    - **timestamp_format:** A string specifying the timestamp format.
    - **add_filename_column:** A boolean indicating whether the filename should be added as a column.
- **queries:** A list of SQL queries to be executed.
- **output:** An array of output tables, defined either as a string specifying the output table name or as an object containing the following parameters:
  - **kbc_destination:** The name of the output table.
  - **primary_key:** A list of primary keys.
  - **incremental:** A boolean indicating whether the data should be loaded incrementally.

#### Example configuration

**Example 1: Load two tables, run queries, and export data**

This configuration loads two input tables, performs queries (including exporting a Parquet file to the `/data/out/files` folder, and finally exports a table to Keboola Storage.

```
{
    "before": [],
    "after": [
        {
            "definition": {
                "component": "keboola.processor-duckdb"
            },
            "parameters": {
                "mode": "advanced",
                "input": [
                    "sales1.csv",
                    "sales2.csv"
                ],
                "queries": [
                    "CREATE view sales_all AS SELECT * FROM sales1.csv UNION ALL SELECT * FROM sales2.csv",
                    "CREATE view sales_first AS SELECT * FROM sales1.csv LIMIT 1",
                    "COPY sales_all TO '/data/out/files/sales_all.parquet' (FORMAT PARQUET)"
                ],
                "output": [
                    "sales_first"
                ]
            }
        }
    ]
}
```

**Example 2: Load data using a glob pattern and join with a Parquet file**

This example demonstrates a full configuration that loads data from storage using a glob pattern. In the queries, we join a table created from multiple CSV files with a Parquet file from storage. For the output, we define the destination table name, primary key (PK), and enable incremental load.
```
{
    "before": [],
    "after": [
        {
            "definition": {
                "component": "keboola.processor-duckdb"
            },
            "parameters": {
                "mode": "advanced",
                "input": [{
                  "input_pattern": "/data/in/tables/*.csv",
                  "duckdb_destination": "products",
                  "dtypes_mode": "auto_detect",
                  "skip_lines": 1,
                  "delimiter": ",",
                  "quotechar": "\"",
                  "column_names": [
                    "id",
                    "name",
                    "category_id"
                  ],
                  "date_format": "YYYY-MM-DD",
                  "timestamp_format": "YYYY-MM-DD HH:MM:SS",
                  "add_filename_column": true
                }],
                "queries": [
                  "CREATE TABLE category AS SELECT * FROM '/data/in/files/categories.parquet'",
                  "CREATE VIEW out AS SELECT p.*, category.name as category_name FROM products AS p LEFT JOIN category on p.category_id = category.id ORDER BY p.id"
                ],
                "output": [
                  {
                    "duckdb_source": "out",
                    "kbc_destination": "out",
                    "primary_key": [
                      "id"
                    ],
                    "incremental": true
                  },
                  "category"
                ]
              }
        }
    ]
}
```

**Example 3: Load data from a URL**

This configuration demonstrates loading a CSV file from a URL and saving it as a table. You can also use [DuckDB extensions](https://duckdb.org/docs/extensions/overview) to load data from other sources.

A common use case when input tables are not defined, is when a file extractor downloads files (e.g., Parquet, JSON, or EXCEL), and you simply want to query them and save the result as a table in Keboola Storage.

```
{
    "before": [],
    "after": [
        {
            "definition": {
                "component": "keboola.processor-duckdb"
            },
            "parameters": {
                "mode": "advanced",
                "queries":["CREATE view cars AS SELECT * FROM 'https://github.com/keboola/developers-docs/raw/3f1e8a4331638a2300b29e63f797a1d52d64929e/integrate/variables/countries.csv'"],
                "output": ["cars"]
            }
        }
    ]
}
```

More configuration examples for both modes can be found in the [tests directory](../tests/) (always inside source/data/config.json).

# Local Debugging

If you need to debug the component—whether for testing SQL queries or optimizing performance—you can run it locally. Here’s how:

## 1. Clone the Repo
First, clone the repository:

```
git clone https://github.com/keboola/processor-duckdb
```

## 2. Open in VS Code & Add Dependencies
- Open the project in VS Code.
- Add `pandas` to `requirements.txt`. This is required to display tabular data in the **Data Wrangler** plugin.
- Press **CMD + Shift + R / Ctrl + Shift + R**, type "Create Environment", and select the option to install the dependencies.


## 3. Copy a Data Folder
- You can either download real data from a Keboola debug job from the step preceeding the DuckDB processor.
- Or, you can use a data folder from the examples under `tests/functional/ANY/source` and copy it to the root of the component directory.
- Inside the data folder, customize the content as needed:
  - `in/tables/` — contains CSV tables.
  - `in/files/` — contains files.
  - `config.json` — contains the configuration.

## 4. Set `data_path_override`
- Get the absolute path to your copied data folder.
- Add it to `data_path_override` on **line 50** of `src/component.py`:

```
ComponentBase.__init__(self, data_path_override="abs/path/component/data")
```

## 5. Add Breakpoints
- Set breakpoints at:
  - **Line 87** for simple mode.
  - **Line 131** for advanced mode.

## 6. Run Debugging Queries
- In the **debug console**, you can run:

  ```
  self._connection.execute("SELECT * FROM table").fetchall()
  ```

- Want to see results in **Data Wrangler**? Store them as a numpy array first:

  ```
  out = self._connection.execute("SELECT * FROM table").fetchnumpy()
  ```

- Then, right-click on the variable and select **Display in Data Wrangler**.  
  **Note:** This doesn’t always work reliably. If it’s acting up, consider using **PyCharm** instead.

---

## Useful Debugging Commands

### Mimic Keboola's Limits
```
SET memory_limit = '2GB';
SET threads = 4;
SET max_temp_directory_size = '20GB';
```

### Check the Query Execution Plan
```
EXPLAIN;
```

### Profile a Query
```
EXPLAIN ANALYZE;
```

### See Temp File Storage Usage
```
SELECT path, round(size/10**6)::INT AS "size_MB" FROM duckdb_temporary_files();
```

### Check Memory Usage
```
SELECT tag, 
       round(memory_usage_bytes/10**6)::INT AS "mem_MB", 
       round(temporary_storage_bytes/10**6)::INT AS "storage_MB" 
FROM duckdb_memory();
```





# Development

This example contains a runnable container with simple unittests. For local testing, it is useful to include the `data` folder
in the root and use docker-compose commands to run the container or execute tests.

If required, change the local data folder (the `CUSTOM_FOLDER` placeholder) path to your custom path:

```yaml
    volumes:
      - ./:/code
      - ./CUSTOM_FOLDER:/data
```

Clone this repository, initialize the workspace, and run the component with the following commands:

```
git clone https://bitbucket.org:kds_consulting_team/kds-team.processor-rename-headers.git my-new-component
cd my-new-component
docker-compose build
docker-compose run --rm dev
```

Run the test suite and lint check using this command:

```
docker-compose run --rm test
```

# Testing

The preset pipeline scripts contain sections allowing pushing a testing image into the ECR repository and automatic
testing in a dedicated project. These sections are by default commented out.

# Integration

For information about deployment and integration with Keboola, please refer to
the [deployment section of the developer documentation](https://developers.keboola.com/extend/component/deployment/). 

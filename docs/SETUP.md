# Setup and execution guide

[Back to README](../README.md)

This guide recreates the workflow manually from the committed ADF source and notebooks. No additional job exports, resource templates, or deployment bundles are required. You must configure your own Azure resources and permissions.

## 1. Clone and access the source

```bash
git clone https://github.com/avneetxsingh/netflix-azure-data-engineering.git
cd netflix-azure-data-engineering
```

While the repository is private, only authorized GitHub users can clone it. For Databricks, connect your GitHub account and create a Git folder from this URL:

```text
https://github.com/avneetxsingh/netflix-azure-data-engineering.git
```

The notebooks are directly under `databricks/`, not a `databricks/notebooks/` subfolder.

## 2. Create the cloud environment

Prepare:

- Azure Data Factory.
- An ADLS Gen2 storage account with hierarchical namespace enabled.
- An Azure Databricks workspace with Unity Catalog and compute appropriate for Auto Loader and Lakeflow Spark Declarative Pipelines.
- An Azure Databricks Access Connector and storage credential/external locations when using managed identity access through Unity Catalog.

Create `raw`, `bronze`, and `silver` containers. Configure managed storage for the catalog and the gold schema. The original environment also had `gold` and `metastore` containers; container creation alone does not configure Unity Catalog.

Grant ADF access to write bronze data and validate the raw titles file. Grant the Databricks identity storage access and grant the executing principal the required catalog, schema, table, and external-location privileges. Scope access to the paths used by this project.

Create or select a writable catalog, such as `netflix_adp`, and a schema such as `net_schema`. The autoloader notebook creates the schema but does not create the catalog.

Azure compute, storage, and pipeline execution can incur charges. Stop unused resources after experimentation.

## 3. Replace environment-specific values

| Current reference | What to configure |
|---|---|
| `netflixxdata` | Your ADLS Gen2 account in all notebook ABFSS paths and the ADF ADLS linked service |
| `netflix_adp.net_schema` | Your writable catalog and schema in `autoloader.ipynb` |
| `NetflixManagement` | Your ADF factory identity/configuration when importing into another factory |
| Existing encrypted ADF credential | Reconfigure authentication in your own factory; do not reuse the exported credential |
| Gold target catalog/schema | Set in your Databricks pipeline configuration |

Keep secrets in Azure Key Vault, Databricks secret scopes, or the service's supported identity mechanism. An ADF `encryptedCredential` is environment-specific connection metadata, not a portable deployment secret.

The original code uses hardcoded paths. Adjust every occurrence before running against a different environment.

## 4. Connect ADF

In ADF Studio, open **Manage → Git configuration → Configure** and select:

| Setting | Value |
|---|---|
| Repository provider | GitHub |
| Repository | Your authorized copy of this repository |
| Collaboration branch | `main` |
| Root folder | `/adf` |
| Publish branch | `adf_publish` |

When connecting a new factory to existing source, use the existing repository definitions. Importing live factory resources is for capturing work from an existing factory; do not overwrite the repository with an empty factory.

Review the two datasets and linked services, test the connections, and validate the pipeline before publishing. A Git commit saves source; ADF publishing updates the live factory. See [ADF source control documentation](https://learn.microsoft.com/en-us/azure/data-factory/source-control).

### Fix the HTTP base URL for the multi-file loop

The exported `LS_github_cast_Source` URL ends in `netflix_cast.csv`, but `DS_Github_Source` also appends `sFileName`. Configure the linked-service URL as the directory base:

```text
https://raw.githubusercontent.com/anshlambagit/Netflix_Azure_Data_Engineering_Project/main/RawData_AND_Notebooks/
```

Keep the dataset relative URL as:

```text
@dataset().sFileName
```

Verify each source URL resolves to the intended CSV. Upstream availability and terms can change; use another authorized source if needed.

### Prepare the titles prerequisite

Obtain an authorized copy of `netflix_titles.csv` and place it at:

```text
abfss://raw@<storage-account>.dfs.core.windows.net/Netflix/netflix_titles.csv
```

The pipeline's `ValidateFiles` activity waits for this file before starting the lookup loop. The default `parray` does **not** download titles.

The `Get_Metadata` web activity currently requests a GitHub HTML page and stores its `Response` as `pmetadata`. That variable does not drive the copy loop. Verify the activity succeeds in your environment; if metadata discovery is needed, adapt the activity to a suitable structured endpoint and update its response expression. Do not assume the HTML response is a CSV file listing.

### Lookup inputs

| folder_name | file_name |
|---|---|
| `netflix_cast` | `netflix_cast.csv` |
| `netflix_category` | `netflix_category.csv` |
| `netflix_countries` | `netflix_countries.csv` |
| `netflix_directors` | `netflix_directors.csv` |

ADF passes `@item().file_name` to the source and destination filename parameters, and `@item().folder_name` to the destination folder. The destination container is `bronze`.

## 5. Run title ingestion

Open `databricks/autoloader.ipynb` on supported Databricks compute and run its cells in order.

It:

1. Creates `netflix_adp.net_schema` if missing.
2. Reads raw CSV data with Auto Loader, multiline handling, and quote escaping.
3. Writes the available data to `netflix_adp.net_schema.raw_titles`.
4. Writes the same source stream to bronze Delta at `bronze/netflix_titles`.

The source points at the raw container root. Keep unrelated CSVs out of that root's ingest scope, or narrow the source path to the titles directory before running.

The two streaming writes use separate checkpoints. Preserve those checkpoints for normal incremental operation. Resetting or changing them can cause source files to be processed again.

## 6. Run bronze-to-silver notebooks

Run the ADF pipeline to populate the bronze lookup CSVs, then run `Silver.ipynb` once for each lookup pair from the table above. Set its `folder_name` and `file_name` widgets before each run.

The notebook also contains exploratory cells that read bronze titles, so complete title ingestion before running the entire notebook. It reads lookup CSVs with headers and schema inference and overwrites the matching silver Delta directory.

Run `Silver_titles.ipynb` after bronze titles exist. It:

- Filters null `show_id` values.
- Deduplicates on `show_id`.
- Trims `title`.
- Casts `release_year` to integer.
- Overwrites `silver/netflix_titles` as Delta.

A manual notebook run is sufficient. To reproduce the original `Netlix_B_T_S` job, create a For each task over the four lookup input objects, pass `{{input.folder_name}}` and `{{input.file_name}}` to a `Silver` notebook task, and add a dependent `Silver_titles` task after the loop succeeds. Select the copied notebooks in your own workspace; the job definition is intentionally not included.

## 7. Configure and run gold

Create a Lakeflow Spark Declarative Pipeline in Databricks:

1. Choose a descriptive name, such as `Netflix Silver to Gold`.
2. Configure an authorized target catalog and schema.
3. Add `databricks/GOLD_Pipeline.ipynb` as the existing source notebook.
4. Use supported pipeline compute with access to the silver Delta paths.
5. Validate, then run a triggered refresh.

If your pipeline editor requires a source file ending in `.py` or `.sql`, export the notebook's Python source as `GOLD_Pipeline.py` in the workspace and attach that file. Do not reference an extensionless regular file. See the [pipeline editor documentation](https://learn.microsoft.com/en-us/azure/databricks/dlt/dlt-multi-file-editor).

Expected materialized views:

- `gold_netflix_titles`
- `gold_netflix_cast`
- `gold_netflix_category`
- `gold_netflix_countries`
- `gold_netflix_directors`

The titles path also has two intermediate views: `staging_netflix_titles` and `transformed_netflix_titles`. Gold storage placement follows the configured pipeline/catalog settings.

## 8. Validate and continue development

Use the checks in the [README](../README.md#validate-your-run). Inspect pipeline expectations and compare source/target counts for your source snapshot.

For future updates:

```bash
git switch -c docs/my-update
# Edit files and review the changes.
git add README.md docs/
git commit -m "Improve project documentation"
git push -u origin docs/my-update
```

Open a pull request on GitHub. Pull changes into the Databricks Git folder before making additional edits there.

## Troubleshooting

| Symptom | Check |
|---|---|
| Private repository cannot be cloned | Confirm your GitHub identity has repository access and Databricks Git authentication is connected |
| ADF HTTP 404 or malformed path | Use the raw directory base URL plus filename-only dataset relative URL |
| ADF pauses at validation | Confirm `raw/Netflix/netflix_titles.csv` exists and ADF can access it |
| ADF fails before the copy loop | Inspect `Get_Metadata` and the response expression used by `Set variable1` |
| Storage permission error | Check the cloud identity's storage role, storage credential/external location, and Databricks principal grants |
| Catalog or schema missing | Create the catalog, choose permitted managed storage, and grant schema creation/use rights |
| CSV read fails or produces unexpected columns | Check header handling, source files, inferred schema, and multiline/escape settings |
| Gold import or decorator error | Run the source through supported declarative pipeline compute |
| Unsupported pipeline file type | Attach the notebook correctly or use an exported `.py` source file |
| Duplicate bronze rows after rerun | Check whether checkpoints were reset, changed, or reused incorrectly |
| Lookup output missing from silver | Run `Silver.ipynb` for all four parameter pairs, not just its cast defaults |

## Reference documentation

- [Azure Data Factory source control](https://learn.microsoft.com/en-us/azure/data-factory/source-control)
- [Databricks Git folders](https://learn.microsoft.com/en-us/azure/databricks/repos/git-operations-with-repos)
- [Lakeflow pipeline editor](https://learn.microsoft.com/en-us/azure/databricks/dlt/dlt-multi-file-editor)

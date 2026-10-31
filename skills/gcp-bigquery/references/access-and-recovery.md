# Access and recovery

## Roles

A query is a job, so running one takes two grants in two places. The job runs and bills in the project it starts from, even when it reads data in another project.

| Role | Allows | Grant at |
| --- | --- | --- |
| BigQuery Job User | Run jobs, including queries | Project or above, not dataset |
| BigQuery Data Viewer | Read data and metadata | Project, dataset or table |
| BigQuery Data Editor | Viewer plus create and change tables | Project, dataset or table |
| BigQuery Data Owner | Editor plus delete the dataset and manage access | Project, dataset or table |
| BigQuery Metadata Viewer | See resources, not data | Project, dataset or table |

Data Viewer on the project reads every dataset in it. Grant it on the one dataset when others must stay hidden. Basic roles (Owner, Editor, Viewer) are too wide for production data.

The IAM page caches. A role just granted may not show until you refresh, and the output of the `gcloud` command is the reliable record.

For a permission error on a service, ask which service account is acting before looking at the person's own roles. See `gcp-build-playbook`.

## Policy tags

A policy tag is a label on a column, defined once in a taxonomy and reusable across tables. It denies by default, even to a project owner. Access comes from a separate role granted on the tag: Fine-Grained Reader (`roles/datacatalog.categoryFineGrainedReader`) for real values, or Masked Reader for masked ones.

Once columns are tagged, `SELECT *` fails for anyone without the role, and this is expected. Leaving the tagged columns out works:

```sql
SELECT * EXCEPT(address, phone, postal_code)
FROM my_dataset.customers;
```

Grant the reader role to the one group that needs the columns. Everyone else keeps querying the same table, minus those fields.

The Add policy tag button appears only in Edit schema mode. The read-only Schema tab lets you tick fields but shows no button.

Not run here: whether tags survive a schema change. Check the docs before changing a tagged table, and check the tags again afterward.

## Row-level security

Not run here: everything in this section. Check it against the docs.

```sql
CREATE ROW ACCESS POLICY apac_filter
ON my_dataset.sales
GRANT TO ('group:apac-team@example.com')
FILTER USING (region = 'APAC');
```

Anyone not named in a policy sees no rows once a policy exists, so add policies for every group before relying on the table. Access from several policies combines.

The policy filters by who runs the query. A BI tool that queries as one shared service account cannot be told apart this way. Looker uses `access_filter` with a user attribute instead.

## Authorized views

Not run here. An authorized view shares a query result with people who have no access to the source tables. They need Job User on the project and Data Viewer on the view's dataset, and the view itself is authorized on the source dataset. Use it to share a subset. To hide one column with no extra views, a policy tag is less work.

## Encryption

Encryption at rest and in transit is on by default, so there is nothing to switch on when nothing requires otherwise.

| Option | Key lives | Use when |
| --- | --- | --- |
| Google-managed | With Google | No rule says you must control keys |
| CMEK (customer-managed) | Cloud KMS, under your control | You must rotate, disable or audit the key |
| CSEK (customer-supplied) | Outside Google | Not supported by BigQuery |

A default key on a dataset applies to tables created afterward, not existing ones.

## Sharing and finding

BigQuery sharing (used to be Analytics Hub) sends data out: publish a dataset once, and subscribers get a read-only linked dataset with no copy. Knowledge Catalog (used to be Dataplex) is for finding and understanding your own data: search, lineage and quality scans. Neither moves or transforms data.

Dataset names such as `raw` and `curated` are only a habit. SQL gives them no meaning, and real governance on those zones lives in Knowledge Catalog.

## Recovery

Not run here: time travel, snapshots and clones. The syntax below comes from the docs, so try it on a scratch table first.

### Time travel

Read or restore a table as it was earlier, within a window of 2 to 7 days. The default is 7, set per dataset or project.

```sql
SELECT *
FROM my_dataset.orders
  FOR SYSTEM_TIME AS OF TIMESTAMP_SUB(CURRENT_TIMESTAMP(), INTERVAL 1 HOUR);
```

A deleted table is restored by copying it from a moment before the delete, with the time in epoch milliseconds:

```bash
bq cp my_dataset.deleted_table@1760000000000 my_dataset.restored_table
```

Restoring a table with row-level policies needs an extra permission that `roles/bigquery.admin` has and `roles/owner` does not.

### Fail-safe

Keeps deleted data for 7 more days after the time travel window. It cannot be configured, and you cannot query it. Recovery goes through Cloud Customer Care, so contact them early.

### Snapshots and clones

| | Table snapshot | Table clone |
| --- | --- | --- |
| What it is | A read-only copy at a moment in time | A writable, lightweight copy |
| Use it for | A known good copy before a risky change | Testing changes on real data |
| Stored cost | Nothing at first. You pay for data the base table later changes or deletes | You pay for what changes |

```sql
CREATE SNAPSHOT TABLE my_dataset.orders_before_fix
CLONE my_dataset.orders;

CREATE TABLE my_dataset.orders_test
CLONE my_dataset.orders;
```

To change a snapshot's data, restore a standard table from it and change that. Copying across regions makes a full copy and adds storage cost.

Object versioning is a Cloud Storage feature. It does not recover a BigQuery table.

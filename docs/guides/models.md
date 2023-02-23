# Models guide

## Prerequisites

---

Before adding a model, ensure that you have [already created your project](/guides/projects) and that you are working in a [dev environment](/concepts/environments).

---

## Adding a model

To add a model:

1. Within your `models` folder, create a new database file. For example, `example_new_model.sql`.
2. Within the file, define your model as follows:

        MODEL (
            name sqlmesh_example.example_new_model,
            kind incremental_by_time_range (
                time_column (ds, '%Y-%m-%d'),
            ),
        )

        SELECT *
        FROM sqlmesh_example.example_incremental_model
        WHERE ds BETWEEN @start_ds and @end_ds

    **Note:** The last line in this file is required if your model is incremental. Refer to [model kinds](../concepts/models/model_kinds.md) for more information about the kinds of models you can create.

## Editing an existing model

To edit an existing model:

1. Open the model file you wish to edit in your preferred editor and make a change.
2. To preview an example of what your change looks like without actually creating a table, use the `evaluate` command. Refer to [evaluating a model](#evaluating-a-model).
3. To materialize this change, use the `plan` command. Refer to [previewing changes using the `plan` command](#previewing-changes-using-the-plan-command).

### Evaluating a model

The `evaluate` command will run a query against your database or engine and return a DataFrame. It is used to test or iterate on models without side effects, and since SQLMesh isn't materializing any data, with minimal cost.

To evaluate a model:

1. Run the `evaluate` command using either the [CLI](../reference/cli.md) or [Notebook](../reference/notebook.md). An example of the `evaluate` command using the `example_incremental_model` from the [quickstart](../quick_start.md) is below:

        $ sqlmesh evaluate sqlmesh_example.example_incremental_model --start=2020-01-07 --end=2020-01-07

        id  item_id          ds
        0   7        1  2020-01-07

2. Once the `evaluate` command detects the changes made to your model, observe that the output from your new model is shown.

### Previewing changes using the `plan` command

When SQLMesh runs the `plan` command on your environment, it will show you whether any downstream models are impacted by your changes. If so, SQLMesh will prompt you to classify the changes as [Breaking](../concepts/plans.md#breaking-change) or [Non-Breaking](../concepts/plans.md#non-breaking-change) before applying the changes.

To preview changes using `plan`:

1. Enter the `sqlmesh plan <environment name>` command. 
2. Enter `1` to classify the changes as `Breaking`, or enter `2` to classify the changes as `Non-Breaking`:

        $ sqlmesh plan dev
        ======================================================================
        Successfully Ran 1 tests against duckdb
        ----------------------------------------------------------------------
        Summary of differences against `dev`:
        ├── Directly Modified:
        │   └── sqlmesh_example.example_incremental_model
        └── Indirectly Modified:
            └── sqlmesh_example.example_full_model
        ---      
        +++   
        @@ -1,6 +1,7 @@         
        SELECT                
        id,              
        item_id,              
        +  1 AS new_column,         
        ds              
        FROM (VALUES
        (1, 1, '2020-01-01'),
        Directly Modified: sqlmesh_example.example_incremental_model
        └── Indirectly Modified Children:
            └── sqlmesh_example.example_full_model
        [1] [Breaking] Backfill sqlmesh_example.example_incremental_model and indirectly modified children
        [2] [Non-breaking] Backfill sqlmesh_example.example_incremental_model but not indirectly modified children: 2
        Models needing backfill (missing dates):
        └── sqlmesh_example.example_incremental_model: (2020-01-01, 2023-02-17)
        Enter the backfill start date (eg. '1 year', '2020-01-01') or blank for the beginning of history: 
        Enter the backfill end date (eg. '1 month ago', '2020-01-01') or blank to backfill up until now: 
        Apply - Backfill Tables [y/n]: y

        All model batches have been executed successfully

        sqlmesh_example.example_incremental_model ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━ 100.0% • 1/1 • 0:00:00

For more information, refer to [plans](../concepts/plans.md).

## Reverting a change to a model

---

Before reverting a change, ensure that you have already made a change and that you have run the `sqlmesh plan` command.

---

To revert your change:

1. Open the model file you wish to edit in your preferred editor, and undo a change you made earlier. For this example, we'll remove the column we added from the [quickstart](/quick_start).
2. Run `sqlmesh plan` and apply your changes. Enter `y` to run a logical update.

        $ sqlmesh plan dev
        ======================================================================
        Successfully Ran 1 tests against duckdb
        ----------------------------------------------------------------------
        Summary of differences against `dev`:
        ├── Directly Modified:
        │   └── sqlmesh_example.example_incremental_model
        └── Indirectly Modified:
            └── sqlmesh_example.example_full_model
        ---     
        +++     
        @@ -1,7 +1,6 @@ 

        SELECT
        id,          
        item_id,              
        -  1 AS new_column,
        ds
        FROM (VALUES
            (1, 1, '2020-01-01'),
        Directly Modified: sqlmesh_example.example_incremental_model (Non-breaking)
        └── Indirectly Modified Children:
            └── sqlmesh_example.example_full_model
        Apply - Logical Update [y/n]: y

        Logical Update executed successfully

### Logical updates

Reverting to a previous version is a quick operation as no additional work is being done. For more information, refer to [plan application](../concepts/plans.md#plan-application) and [logical updates](../concepts/plans.md#logical-updates).

**Note:** The janitor runs periodically and automatically to clean up SQLMesh artifacts no longer being used, and determines the time-to-live (TTL) for tables (how much time can pass before reverting is no longer possible).

## Validating changes to a model

### Automatic model validation

SQLMesh automatically validates your models in order to ensure the quality and accuracy of your data. This is done by the following:

* Running unit tests by default when you run the `plan` command. This ensures all changes to applied to any environment is logically validated. Refer to [testing](/concepts/tests) for more information.
* Running audits whenever data is loaded to a table (both backfill and loading on a cadence). This way you know all data present in any table has passed defined audits. Refer to [auditing](/concepts/audits) for more information.

CI/CD is another way SQLMesh provides automatic validation, since the preview environment is automatically created.

### Manual model validation

To manually validate your models, you can perform one or more of the following tasks:

* [Evaluating a model](#evaluating-a-model)
* [Testing a model using unit tests](/guides/testing#testing-changes-to-models)
* [Auditing a model](/guides/testing#auditing-changes-to-models)
* [Previewing changes using the `plan` command](#previewing-changes-using-the-plan-command)

## Viewing the DAG

---

Before generating the DAG, ensure that you have already installed the graphviz package. 

To install the graphviz package, enter the following command:

```bash
pip install graphviz
```

Alternatively, enter the following command:

```bash
sudo apt-get install graphviz
```

---

To view the DAG, enter the following command:

`sqlmesh dag`

The generated files (.gv and .jpeg formats) will be placed at the root of your project folder.

## Deleting a model

---

Before deleting a model, ensure that you have already run `sqlmesh plan`.

---

To delete a model:

1. Within your `models` folder, delete the file containing the model you wish to remove. For this example, we'll delete the `example_full_model.sql` file from our [quickstart](/quick_start) project.
2. Run the `sqlmesh plan <environment>` command, and enter the environment to which you want to apply the change. In the following example, we apply the change to our development environment:

        ```
        $ sqlmesh plan dev
        ======================================================================
        Successfully Ran 0 tests against duckdb
        ----------------------------------------------------------------------
        Summary of differences against `dev`:
        └── Added Models:
            └── sqlmesh_example.example_incremental_model
        Models needing backfill (missing dates):
        └── sqlmesh_example.example_incremental_model: (2020-01-01, 2023-02-17)
        Enter the backfill start date (eg. '1 year', '2020-01-01') or blank for the beginning of history: 
        Enter the backfill end date (eg. '1 month ago', '2020-01-01') or blank to backfill up until now: 
        Apply - Backfill Tables [y/n]: y

        All model batches have been executed successfully

        sqlmesh_example.example_incremental_model ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━ 100.0% • 1/1 • 0:00:00
        ```

    **Note:** If you have other files that reference the model you wish to delete, an error message will note the file(s) containing the reference. You will need to also delete these files in order for the change to be applied.

3. Plan and apply your changes to production, and enter `y` for the logical update. By default, the `sqlmesh plan` command targets your production environment:

        ```
        $ sqlmesh plan
        ======================================================================
        Successfully Ran 0 tests against duckdb
        ----------------------------------------------------------------------
        Summary of differences against `prod`:
        └── Removed Models:
            └── sqlmesh_example.example_full_model
        Apply - Logical Update [y/n]: y

        Logical Update executed successfully
        ```

4. Verify that the `example_full_model.sql` model was removed from the output.
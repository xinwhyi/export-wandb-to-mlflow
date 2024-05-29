# export-wandb-to-mlflow

export-wandb-to-mlflow is a tool for people to easily export WandB projects to MLflow.
Basically the mapping between WandB and MLflow is:

- WandB projects <=> MLflow experiments
- WandB runs <=> MLflow runs

To use the tool, first clone and install the package:

```bash
git clone https://github.com/chenmoneygithub/export-wandb-to-mlflow.git
cd export-wandb-to-mlflow.git
pip install -e "."
```

Then run to kick off the data migration:

```bash
w2m --wandb_project_name="{your-wandb-project-name}" [Options]
```

Available options are listed below:

| Option                   | Type    | Explanation                                                                                                                                                                       |
| :----------------------- | :------ | :-------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | --- |
| mlflow_experiment_name   | String  | Use the designated name as the MLflow experiment name, instead of automatically creating one.                                                                                     |
| verbose                  | Bool    | If verbose logging is enabled.                                                                                                                                                    |
| resume_from_crash        | Bool    | If True, the job will be run in resumption mode: half-complete runs will be deleted and re-migrated, finished runs will be skipped. This is useful when the previous run crashed. |
| use_nested_run           | Bool    | If using nested MLflow run to represent wandb group.                                                                                                                              |
| log_dir                  | Str     | If set, the logging will be written to the log_dir as well as the stdout.                                                                                                         |
| wandb_run_names          | List    | If set, only runs specified by wandb_run_names are migrated.                                                                                                                      |
| exclude_metrics          | List    | Metrics matching patterns in exclude_metrics will not be migrated                                                                                                                 |
| dry_run                  | Bool    | If True, the data will be written to files in `dry_run_save_dir`                                                                                                                  |
| resume_from_dry_run      | Bool    | If True, the data will be read from files in `dry_run_save_dir` instead of wandb server                                                                                           |
| dry_run_save_dir         | String  | The path to directory to save data in dry run mode or resume from dry run mode                                                                                                    |
| dry_run_thread_pool_size | Integer | The size of threadpool that reads from wandb server when running in dry run mode.                                                                                                 |     |

If `mlflow_experiment_name` is not set, an MLflow experiment of the same name as WandB experiment will be created, and all MLflow
runs will be created using the corresponding WandB runs' names. If WandB runs are grouped, the group name will be logged to MLflow
as a tag `run_group`, and users can use this tag to group MLflow runs on MLflow UI.

## Dry Run Mode

### Data Export

The migration script supports a dry run mode, in which mode the data will be saved to your local disk as specified in
`dry_run_save_dir`. This is useful when you want to keep your experiment data while not ready to move them to MLflow
server. To run the migration script in dry run mode, and assume you have a wandb project `test-llama3`, you can use the command below:

```shell
w2m --wandb_project_name="test-llama3" --dry_run=True --dry_run_save_dir="./migrations"
```

The command above will create a directory called `migrations` under your current working directory if not exists, and save
data to it. Assume your wandb project is called `test-llama3`, and it has 2 runs with id `abc123` and `def456` separatley, the
saved files will take the following structure:

```
migrations/
├─ test-llama3/
│  ├─ abc123/
│  │  ├─ tags.csv
│  │  ├─ system_metrics/
│  │  ├─ params.json
│  ├─ def456/
│  │  ├─ tags.csv
│  │  ├─ params.json
│  │  ├─ metrics/
│  │  ├─ system_metrics/
│  ├─ tags.csv
```

### Data Import

Once you are ready to import data saved earlier in dry run mode to MLflow, you can use `resume_from_dry_run=True` to tell
the migration script to do the job. Assume your data is saved in `./migrations` as you followed the example above, you can
run the command below:

```shell
w2m --wandb_project_name="test-llama3" --resume_from_dry_run=True --dry_run_save_dir="./migrations"
```

We highly recommend users use dry run export/import to migrate large experiments, which is much faster than the normal mode because
we do arbitrary grouping while saving to files in dry run mode.

## Resume from Crashing

Sometimes your migration crashes in the middle, to avoid the headache of starting over, we provide a flag `resume_from_crash`. When
`resume_from_crash=True`, the script will try finding the existing MLflow experiment, and delete runs that started but not finished migrations.
For already finished runs, the script will skip the migration. `resume_from_crash` can be used together with dry run mode.

## Keep Your Logging

If you are willing to keep the log of running the migration script, then you can set `log_dir` to your desired path of saving
logs. If you set `log_dir`, logs will be both printed to console and saved to files.

## Limitations

For now this tool only supports migrating WandB config (params) and metrics. Artifacts migration need to be done manually.

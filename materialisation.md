# Scheduler

## Feast CLI: `materialize`

The Feast CLI's `materialize` command is a critical component in the Feast workflow, enabling the transfer of data from the offline store to the online store for real-time access. This process is essential for keeping the online store updated with the latest feature data, ensuring that machine learning models have access to the most recent and relevant information.

When the `materialize` command is executed, it triggers a series of operations:

1. **Initialization**: The command starts by initializing a `FeatureStore` instance. This is done by reading the configuration from the `feature_store.yaml` file, which contains all the necessary settings and definitions for the feature store.

2. **Materialization Process**: The core of the materialization process involves reading data from the specified time range (between `start_date` and `end_date`) from the offline store and writing it to the online store. This is done for each feature view specified in the command or for all registered feature views if none are specified. 

3. **Provider Interaction**: The `FeatureStore` instance interacts with the provider (in this case Snowflake - the offline store) to execute the materialization for each feature view. Snowflake, as the offline store, stores the historical feature data. This data is used for training machine learning models or for materialization into the online store. Any compute-intensive operations, like aggregating or transforming historical data, are performed within Snowflake.

The offline store provider is responsible for the actual data transfer between the offline and online stores. Snowflake (our offline store) in this case acts as the compute engine and storage for the offline features.  , so it handles the data processing and storage, but it's the Python script that orchestrates what data to fetch and where to store it.

4. **Registry Update**: After the materialization is complete, the `FeatureStore` updates the registry with information about the materialized data. This step is crucial for tracking the state and history of materialized data in the feature store.

<br>

### Detailed Workflow

**Initialization Step**
The `materialize_command` function in the CLI handler initiates the materialization process. It creates a `FeatureStore` instance using the repository configuration.

Source: [cli.materialize_command()](https://github.com/feast-dev/feast/blob/052182bcca046e35456674fc7d524825882f4b35/sdk/python/feast/cli.py#L531)
```py
def materialize_command(
    ctx: click.Context, start_ts: str, end_ts: str, views: List[str]
):
    ... # Omitted for brevity
    store = create_feature_store(ctx)
    store.materialize(
        feature_views=None if not views else views,
        start_date=utils.make_tzaware(parser.parse(start_ts)),
        end_date=utils.make_tzaware(parser.parse(end_ts)),
    )
```

<br>

**Feature Store Creation**
The `create_feature_store` function is responsible for setting up the FeatureStore instance. It prepares the environment and configuration necessary for the feature store to operate.

Source: [repo_operations.create_feature_store()](https://github.com/feast-dev/feast/blob/052182bcca046e35456674fc7d524825882f4b35/sdk/python/feast/repo_operations.py#L334)
```py
def create_feature_store(
    ctx: click.Context,
) -> FeatureStore:
    ... # Omitted for brevity
    repo_path = Path(tempfile.mkdtemp())
    with open(repo_path / "feature_store.yaml", "wb") as f:
        f.write(config_bytes)
    return FeatureStore(repo_path=ctx.obj["CHDIR"], fs_yaml_file=fs_yaml_file)
```

<br>

**Materialization Execution**
The `materialize` function in the FeatureStore class is where the actual materialization process is called. It iterates over each feature view and instructs the provider to compute the values from historic data offline store to the online store.

Source: [FeatureStore.materialize()](https://github.com/feast-dev/feast/blob/052182bcca046e35456674fc7d524825882f4b35/sdk/python/feast/feature_store.py#L1342C9-L1342C20)
```py
def materialize(
    self,
    start_date: datetime,
    end_date: datetime,
    feature_views: Optional[List[str]] = None,
) -> None:
    ... # Omitted for brevity
    provider = self._get_provider()
    feature_views_to_materialize = self._get_feature_views_to_materialize(feature_views)

    for feature_view in feature_views_to_materialize:
        provider.materialize_single_feature_view(
            config=self.config,
            feature_view=feature_view,
            start_date=start_date,
            end_date=end_date,
            registry=self._registry,
            project=self.project,
            tqdm_builder=lambda length: tqdm(total=length, ncols=100),
        )
        self._registry.apply_materialization(feature_view, self.project, start_date, end_date)
```

<br>

**Materialization Engine**
The `materialize_single_feature_view()` method uses a Provide passthrough to specify which engine (in our case [Snowflake](https://github.com/feast-dev/feast/blob/052182bcca046e35456674fc7d524825882f4b35/sdk/python/feast/infra/materialization/snowflake_engine.py#L211)) to compute the transformation on.

Source: [PassthroughProvider.materialize_single_feature_view()](https://github.com/feast-dev/feast/blob/052182bcca046e35456674fc7d524825882f4b35/sdk/python/feast/infra/passthrough_provider.py#L226)

```python
def materialize_single_feature_view(
    self,
    config: RepoConfig,
    feature_view: FeatureView,
    start_date: datetime,
    end_date: datetime,
    registry: BaseRegistry,
    project: str,
    tqdm_builder: Callable[[int], tqdm],
) -> None:
    ... # Omitted for brevity
    task = MaterializationTask(
        project=project,
        feature_view=feature_view,
        start_time=start_date,
        end_time=end_date,
        tqdm_builder=tqdm_builder,
    )
    jobs = self.batch_engine.materialize(registry, [task]) # Where the batch engine in our case in Snowflake
    assert len(jobs) == 1
    if jobs[0].status() == MaterializationJobStatus.ERROR and jobs[0].error():
        e = jobs[0].error()
        assert e
        raise e
```

<br>

## Caveats

In essence, the environment running the Feast FeatureStore is the mediator between Snowflake (the offline store) and the online store. It retrieves data from Snowflake, processes it if necessary, and then transfers it to the online store for quick access during online feature serving.

- Snowflake, as the offline store, stores the historical feature data. This data is used for training machine learning models or for materialization into the online store. Any compute-intensive operations, like aggregating or transforming historical data, are performed within Snowflake.

- When you run a materialization job in Feast (like materialize_incremental()), Feast interacts with Snowflake to retrieve the necessary feature data.
Feast then takes this data and writes it to the online store. The online store could be a different system, such as Redis, DynamoDB, or another database optimized for low-latency access.
Data Transfer:

- The actual transfer of materialized data from Snowflake (offline store) to the online store is managed by Feast. Snowflake provides the data, but it's Feast that retrieves this data and then pushes it to the online store.  This process involves reading the relevant data from Snowflake, possibly processing it further if needed, and then writing it to the online store.

<br>

## Solutions

**Scheduler's Role:**
- The scheduler (like Apache Airflow or Prefect) is responsible for triggering Feast materialization jobs according to a defined schedule or set of conditions.
- It does not interact directly with the feature_store.yaml file or the Feast API. Instead, it executes jobs or scripts that do.

**Jobs/Scripts Executed by the Scheduler:**
- The actual Python script or Feast job that is triggered by the scheduler needs access to the feature_store.yaml.
- This script is responsible for initializing the FeatureStore instance and performing operations like materialize or materialize_incremental.
- The script should be executed in an environment where Feast is installed and configured, with access to the necessary configuration files and credentials to interact with both the offline and online stores.

<br>

### Option 1: FastAPI Service as a Feast Materialization Handler

**Design Overview:**
- The FastAPI service acts as an intermediary between the scheduler and Feast. It exposes an endpoint that, when hit by an HTTP(S) request, triggers the Feast materialization process.
- The scheduler is responsible for sending HTTP(S) requests to the FastAPI service based on a predefined schedule or conditions. This could be a simple cron job, Apache Airflow DAG, or any other scheduling tool.
- In this approach, the scheduler makes HTTP requests to your FastAPI server, which calls Feast's FeatureStore `materialize` method. The FastAPI server acts as an intermediary between the scheduler and Feast. 

**Considerations:**
- Depending on the scale, ensure that the FastAPI service has access to sufficient resources (CPU, memory) to handle the Feast operations without performance degradation.

**Advantages:**
- Separating the scheduler from direct Feast operations enhances modularity and maintainability.
- FastAPI's asynchronous nature allows for handling multiple materialization requests efficiently, which is crucial for high-load scenarios.
- Having a single point for Feast operations simplifies management, logging, and monitoring.

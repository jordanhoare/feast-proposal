# Server

## Feast CLI: `serve`

The Feast CLI's `serve` command is designed to start a local feature server, enabling real-time feature serving. This command is particularly useful for testing and development purposes. However, when scaling to a production-grade solution, several considerations and adaptations are necessary.

When the `serve` command is executed, it performs the following operations:

1. **Initialization**: It starts by creating a `FeatureStore` instance, which is essential for managing and serving feature data.

2. **Server Startup**: The command then initializes and starts a local server, making the feature store's capabilities available over a network.

<br>

### Detailed Workflow

**Initialization Step**
The `serve_command` function in the CLI handler initiates the server startup process. It creates a `FeatureStore` instance using the repository configuration.

Source: [cli.serve_command()](https://github.com/feast-dev/feast/blob/052182bcca046e35456674fc7d524825882f4b35/sdk/python/feast/cli.py#L675C1-L698C6)
```py
def serve_command(
    ctx: click.Context,
    host: str,
    port: int,
    type_: str,
    no_access_log: bool,
    no_feature_log: bool,
    workers: int,
    keep_alive_timeout: int,
    registry_ttl_sec: int = 5,
):
    """Start a feature server locally on a given port."""
    store = create_feature_store(ctx)
    store.serve(
        host=host,
        port=port,
        type_=type_,
        no_access_log=no_access_log,
        no_feature_log=no_feature_log,
        workers=workers,
        keep_alive_timeout=keep_alive_timeout,
        registry_ttl_sec=registry_ttl_sec,
    )
```

<br>


**Feature Store Creation**
The `create_feature_store` function sets up the FeatureStore instance, preparing the necessary environment and configuration.

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

**Server Execution**
The `serve` method in the FeatureStore class handles the actual server startup, configuring and launching a local feature server.

Source: [FeatureStore.serve()](https://github.com/feast-dev/feast/blob/052182bcca046e35456674fc7d524825882f4b35/sdk/python/feast/feature_store.py#L2222)
```py
    def serve(
        self,
        host: str,
        port: int,
        type_: str,
        no_access_log: bool,
        no_feature_log: bool,
        workers: int,
        keep_alive_timeout: int,
        registry_ttl_sec: int,
    ) -> None:
        """Start the feature consumption server locally on a given port."""
        ... # Omitted for brevity
        feature_server.start_server(
            self,
            host=host,
            port=port,
            no_access_log=no_access_log,
            workers=workers,
            keep_alive_timeout=keep_alive_timeout,
            registry_ttl_sec=registry_ttl_sec,
        )
```

<br>

**Server Startup**
The `start_server` function in the feature_server module is responsible for configuring and running the Gunicorn server with Feast-specific settings.

Source: [feature_server.start_server()](https://github.com/feast-dev/feast/blob/052182bcca046e35456674fc7d524825882f4b35/sdk/python/feast/feature_server.py#L225)
```py
class FeastServeApplication(gunicorn.app.base.BaseApplication):
    def __init__(self, store: "feast.FeatureStore", **options):
        self._app = get_app(
            store=store,
            registry_ttl_sec=options.get("registry_ttl_sec", 5),
        )
        self._options = options
        super().__init__()

    ... # Omitted for brevity
        
def start_server(
    store: "feast.FeatureStore",
    host: str,
    port: int,
    no_access_log: bool,
    workers: int,
    keep_alive_timeout: int,
    registry_ttl_sec: int = 5,
):
    FeastServeApplication(
        store=store,
        bind=f"{host}:{port}",
        accesslog=None if no_access_log else "-",
        workers=workers,
        keepalive=keep_alive_timeout,
        registry_ttl_sec=registry_ttl_sec,
    ).run()
```

<br>

## Caveats in Scaling Feast's Serve Command

Scaling the out-of-the-box `serve` command in a production environment presents several challenges:

- **Consistency is Key**: The `feature_store.yaml` configuration must be consistent across all services and instances. Any variation can lead to inconsistent feature serving, impacting reliability and trust in the system.
- **Handling High Traffic**: The default server setup may not be fully optimized for high-traffic scenarios, potentially leading to performance bottlenecks.
- **Secure Access**: Managing secure access and permissions in a distributed environment can be complex and requires careful planning.

<br>

## Implications and Solutions

To overcome these challenges, a viable solution is to create an abstraction layer using a FastAPI server that encapsulates a FeatureStore instance. This approach brings several benefits:

1. **Centralized Configuration**: Hosting the `feature_store.yaml` on a central server ensures uniformity and consistency across all instances.
2. **Enhanced Scalability**: FastAPI offers greater flexibility and scalability compared to the default Feast server. This allows for efficient handling of high traffic and complex workloads.
3. **Streamlined Security**: Implementing security measures and access control is more straightforward with a dedicated server setup.

### Customized Feature Serving
Directly using the Feast API through this abstraction allows for the creation of custom endpoints, tailored to specific use cases. This approach enables the design of API routes and response formats that align closely with your application's needs, enhancing the efficiency and user experience.

For instance, here are some examples of how Feast's out-of-the-box `serve` command can be extended and customized:

### [Retrieving features](https://docs.feast.dev/reference/feature-servers/python-feature-server#retrieving-features)
After the server starts, custom requests can be executed to retrieve features. This flexibility allows for more specific and efficient data retrieval.

```bash
$  curl -X POST \
  'http://0.0.0.0:8000/materialize' \
  -d '{
    "features": [
      "driver_hourly_stats:conv_rate",
      "driver_hourly_stats:acc_rate",
      "driver_hourly_stats:avg_daily_trips"
    ],
    "entities": {
      "driver_id": [1001, 1002, 1003]
    }
  }'
```

<br>

### [Pushing features to the online and offline stores](https://docs.feast.dev/reference/feature-servers/python-feature-server#pushing-features-to-the-online-and-offline-stores)
The server also exposes endpoints for pushing data to the online and offline stores, enabling dynamic and real-time data updates.

```bash
curl -X POST \
    'http://0.0.0.0:8000/materialize' 
    -d '{
    "push_source_name": "driver_stats_push_source",
    "df": {
            "driver_id": [1001],
            "event_timestamp": ["2022-05-13 10:59:42+00:00"],
            "created": ["2022-05-13 10:59:42"],
            "conv_rate": [1.0],
            "acc_rate": [1.0],
            "avg_daily_trips": [1000]
    },
    "to": "online_and_offline"
  }'
```
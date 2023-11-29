# UI

## Feast CLI: `ui`

The Feast CLI's `ui` command initializes and serves the Feast UI locally.

When the `ui` command is executed, it triggers a series of operations:

1. **Feature Store Initialization**: Initializes the Feast Feature Store.
2. **UI Server Setup**: Sets up and starts a local UI server using FastAPI.
3. **Static File Serving**: Serves the pre-built React application.

<br>


### Detailed Workflow

**CLI Handler**
The `ui` command in the CLI handler initializes the Feast Feature Store and starts the UI server.

Source: [cli.ui()](https://github.com/feast-dev/feast/blob/e06588314be6bde35e07681a53c41730e21b884f/sdk/python/feast/cli.py#L158)

```py
@click.pass_context
def ui(
    ctx: click.Context,
    host: str,
    port: int,
    registry_ttl_sec: int,
    root_path: Optional[str] = "",
):
    """
    Shows the Feast UI over the current directory
    """
    # Pass in the registry_dump from: https://github.com/feast-dev/feast/blob/052182bcca046e35456674fc7d524825882f4b35/sdk/python/feast/repo_operations.py#L371
    store = create_feature_store(ctx)

    store.serve_ui(
        host=host,
        port=port,
        get_registry_dump=registry_dump,
        registry_ttl_sec=registry_ttl_sec,
        root_path=root_path,
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

**UI Server Initialization**
The `serve_ui` method in the FeatureStore class is responsible for starting the UI server locally.

Source: [FeatureStore.serve_ui()](https://github.com/feast-dev/feast/blob/052182bcca046e35456674fc7d524825882f4b35/sdk/python/feast/feature_store.py#L2256)
```py
    def serve_ui(
        self,
        host: str,
        port: int,
        get_registry_dump: Callable,
        registry_ttl_sec: int,
        root_path: str = "",
    ) -> None:
        ... # Omitted for brevity
        ui_server.start_server(
            self,
            host=host,
            port=port,
            get_registry_dump=get_registry_dump,
            project_id=self.config.project,
            registry_ttl_sec=registry_ttl_sec,
            root_path=root_path,
        )
```

<br>

**Static File Serving**
The `start_server` function in `ui_server` sets up the FastAPI application to serve the built React UI.

Source: [ui_server.start_server()](https://github.com/feast-dev/feast/blob/052182bcca046e35456674fc7d524825882f4b35/sdk/python/feast/feature_store.py#L2256)
```py

def get_app(
    store: "feast.FeatureStore",
    project_id: str,
    registry_ttl_secs: int,
    root_path: str = "",
):
    app = FastAPI()

    ... # Omitted for brevity

    # Initialize with the projects-list.json file
    ui_dir_ref = importlib_resources.files(__name__) / "ui/build/"
    with importlib_resources.as_file(ui_dir_ref) as ui_dir:
        with ui_dir.joinpath("projects-list.json").open(mode="w") as f:
            projects_dict = {
                "projects": [
                    {
                        "name": "Project",
                        "description": "Test project",
                        "id": project_id,
                        "registryPath": f"{root_path}/registry",
                    }
                ]
            }
            f.write(json.dumps(projects_dict))

    ... # Omitted for brevity


    @app.api_route("/p/{path_name:path}", methods=["GET"])
    def catch_all():
        filename = ui_dir.joinpath("index.html")

        with open(filename) as f:
            content = f.read()

        return Response(content, media_type="text/html")

    app.mount(
        "/",
        StaticFiles(directory=ui_dir, html=True),
        name="site",
    )

    return app

def start_server(
    store: "feast.FeatureStore",
    host: str,
    port: int,
    get_registry_dump: Callable,
    project_id: str,
    registry_ttl_sec: int,
    root_path: str = "",
):
    app = get_app(
        store,
        project_id,
        registry_ttl_sec,
        root_path,
    )
    uvicorn.run(app, host=host, port=port)
```

<br>

## Feast UI Build

When you install the Feast Python package (via `pip install feast`), it includes the built React application within its package structure. When ui_server.start_server(), starts a FastAPI server that serves this pre-built React application. This setup allows you to quickly start a local instance of the Feast UI without needing to manually build the React application yourself.

![Feast UI Build](../docs/feast-ui-build.png)


<br>

## Feast NPM Package & Caveats

The [FeastUI.tsx](https://github.com/feast-dev/feast/blob/v0.34-branch/ui/src/FeastUI.tsx) sets up the main React components and routing for the application. It uses [FeastUISansProviders.tsx](https://github.com/feast-dev/feast/blob/v0.34-branch/ui/src/FeastUISansProviders.tsx) and passes `FeastUIConfigs` as a prop. This suggests some level of configurability, as `FeastUIConfigs` could be used to customize certain aspects of the UI.

This file appears to be the core of the UI, where the main components and routes are defined. It accepts `FeastUIConfigs` as a prop, which indicates that some level of customization is possible, as referenced in their [official start up guide](https://docs.feast.dev/reference/alpha-web-ui#importing-as-a-module-to-integrate-with-an-existing-react-app). 

The extent of what can be customized and how easy it is to implement these customizations would depend on the specific implementation details within these components and the designed limitations of the FeastUIConfigs prop. The `FeastUIConfigs interface` includes the following properties:

- tabsRegistry: This allows for the customization of tabs within the UI.
- featureFlags: This is used for enabling or disabling certain features within the UI.
- projectListPromise: This is a promise that resolves to a list of projects. It seems to be used for customizing how the list of projects is fetched and displayed.

<br>

## Solutions

By directing our UI application to the FastAPI server running Feast's FeatureStore, we create a clear separation between the frontend and backend - leading to a more modular and manageable architecture. We can do so by containerising our React application for deployment. Then we can consider our implement requirements:

**Option 1: Modify Source Code Directly**

To modify specific texts or elements on a specific page, you would likely need to modify the source code of those pages directly. This could involve forking the repository and making changes to the React components as needed. Now this new React app can be re-built (and deployed).

By directing our UI application to the FastAPI server running Feast's FeatureStore, we create a clear separation between the frontend and backend - leading to a more modular and manageable architecture. 

<br>

**Option 2: Build with out-the-box Feast UI from NPM**
...
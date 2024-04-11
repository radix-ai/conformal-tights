[![Open in Dev Containers](https://img.shields.io/static/v1?label=Dev%20Containers&message=Open&color=blue&logo=visualstudiocode)](https://vscode.dev/redirect?url=vscode://ms-vscode-remote.remote-containers/cloneInVolume?url=https://github.com/radix-ai/conformal-tights) [![Open in GitHub Codespaces](https://img.shields.io/static/v1?label=GitHub%20Codespaces&message=Open&color=blue&logo=github)](https://github.com/codespaces/new?hide_repo_select=true&ref=main&repo=765698489&skip_quickstart=true)

# 👖 Conformal Tights

Conformal Tights is a Python package that exports:
- a [scikit-learn](https://github.com/scikit-learn/scikit-learn) [meta-estimator](https://scikit-learn.org/stable/glossary.html#term-meta-estimator) that adds [conformal prediction](https://en.wikipedia.org/wiki/Conformal_prediction) of coherent [quantiles](https://en.wikipedia.org/wiki/Quantile) and [intervals](https://en.wikipedia.org/wiki/Prediction_interval) to any [scikit-learn regressor](https://scikit-learn.org/stable/glossary.html#term-regressor)
- a [Darts](https://github.com/unit8co/darts) [forecaster](https://unit8co.github.io/darts/generated_api/darts.models.forecasting.regression_model.html) that adds conformally calibrated [probabilistic time series forecasting](https://unit8co.github.io/darts/userguide/forecasting_overview.html#probabilistic-forecasts) to any scikit-learn regressor

## Features

1. 🍬 *Sklearn meta-estimator*: add conformal prediction of quantiles and intervals to any scikit-learn regressor
2. 🔮 *Darts forecaster:* add conformally calibrated probabilistic time series forecasting to any scikit-learn regressor
3. 🌡️ *Conformally calibrated:* accurate quantiles, and intervals with reliable [coverage](https://en.wikipedia.org/wiki/Coverage_probability)
4. 🚦 *Coherent quantiles:* quantiles increase monotonically instead of [crossing](https://github.com/dmlc/xgboost/issues/9848) [each other](https://github.com/microsoft/LightGBM/issues/3447)
5. 👖 *Tight quantiles:* selects the lowest [dispersion](https://en.wikipedia.org/wiki/Statistical_dispersion) that provides the desired coverage
6. 🎁 *Data efficient:* requires only a small number of calibration examples to fit
7. 🐼 *Pandas support:* optionally predict on DataFrames and receive DataFrame output

## Using

### Installing

First, install this package with:

```sh
pip install conformal-tights
```

### Predicting quantiles

Conformal Tights exposes a meta-estimator called `ConformalCoherentQuantileRegressor` that you can use to wrap any scikit-learn regressor, after which you can use `predict_quantiles` to predict conformally calibrated quantiles. Example usage:

```python
from conformal_tights import ConformalCoherentQuantileRegressor
from sklearn.datasets import fetch_openml
from sklearn.model_selection import train_test_split
from xgboost import XGBRegressor

# Fetch dataset and split in train and test
X, y = fetch_openml("ames_housing", version=1, return_X_y=True, as_frame=True, parser="auto")
X_train, X_test, y_train, y_test = train_test_split(X, y, test_size=0.15, random_state=42)

# Create a regressor, wrap it, and fit on the train set
my_regressor = XGBRegressor(objective="reg:absoluteerror")
conformal_predictor = ConformalCoherentQuantileRegressor(estimator=my_regressor)
conformal_predictor.fit(X_train, y_train)

# Predict with the wrapped regressor
ŷ_test = conformal_predictor.predict(X_test)

# Predict quantiles with the conformal wrapper
ŷ_test_quantiles = conformal_predictor.predict_quantiles(X_test, quantiles=(0.025, 0.05, 0.1, 0.9, 0.95, 0.975))
```

When the input data is a pandas DataFrame, the output is also a pandas DataFrame. For example, printing the head of `ŷ_test_quantiles` yields:

|   house_id |    0.025 |     0.05 |      0.1 |    0.9 |   0.95 |   0.975 |
|-----------:|---------:|---------:|---------:|-------:|-------:|--------:|
|       1357 | 114784   | 120894   | 131618   | 175761 | 188052 |  205449 |
|       2367 |  67416.6 |  80073.7 |  86754   | 117854 | 127583 |  142322 |
|       2822 | 119423   | 132048   | 138725   | 178526 | 197246 |  214206 |
|       2126 |  94030.6 |  99850   | 110891   | 150249 | 164703 |  182528 |
|       1544 |  68996.2 |  81516.3 |  88231.6 | 121774 | 132425 |  147110 |

Let's visualize the predicted quantiles on the test set:

<img src="https://github.com/radix-ai/conformal-tights/assets/4543654/af22b3c8-7bc6-44fd-b96b-24e8d28bc1fd" width="512">

<details>
<summary>Expand to see the code that generated the graph above</summary>

```python
import matplotlib.pyplot as plt
import matplotlib.ticker as ticker

%config InlineBackend.figure_format = "retina"
plt.rcParams["font.size"] = 8
idx = (-ŷ_test.sample(50, random_state=42)).sort_values().index
y_ticks = list(range(1, len(idx) + 1))
plt.figure(figsize=(4, 5))
for j in range(3):
    end = ŷ_test_quantiles.shape[1] - 1 - j
    coverage = round(100 * (ŷ_test_quantiles.columns[end] - ŷ_test_quantiles.columns[j]))
    plt.barh(
        y_ticks,
        ŷ_test_quantiles.loc[idx].iloc[:, end] - ŷ_test_quantiles.loc[idx].iloc[:, j],
        left=ŷ_test_quantiles.loc[idx].iloc[:, j],
        label=f"{coverage}% Prediction interval",
        color=["#b3d9ff", "#86bfff", "#4da6ff"][j],
    )
plt.plot(y_test.loc[idx], y_ticks, "s", markersize=3, markerfacecolor="none", markeredgecolor="#e74c3c", label="Actual value")
plt.plot(ŷ_test.loc[idx], y_ticks, "s", color="blue", markersize=0.6, label="Predicted value")
plt.xlabel("House price")
plt.ylabel("Test house index")
plt.yticks(y_ticks, y_ticks)
plt.tick_params(axis="y", labelsize=6)
plt.grid(axis="x", color="lightsteelblue", linestyle=":", linewidth=0.5)
plt.gca().xaxis.set_major_formatter(ticker.StrMethodFormatter("${x:,.0f}"))
plt.gca().spines["top"].set_visible(False)
plt.gca().spines["right"].set_visible(False)
plt.legend()
plt.tight_layout()
plt.show()
```
</details>

### Predicting intervals

In addition to quantile prediction, you can use `predict_interval` to predict conformally calibrated prediction intervals. Compared to quantiles, these focus on reliable coverage over quantile accuracy. Example usage:

```python
# Predict an interval for each example with the conformal wrapper
ŷ_test_interval = conformal_predictor.predict_interval(X_test, coverage=0.95)

# Measure the coverage of the prediction intervals on the test set
coverage = ((ŷ_test_interval.iloc[:, 0] <= y_test) & (y_test <= ŷ_test_interval.iloc[:, 1])).mean()
print(coverage)  # 96.6%
```

When the input data is a pandas DataFrame, the output is also a pandas DataFrame. For example, printing the head of `ŷ_test_interval` yields:

|   house_id |    0.025 |   0.975 |
|-----------:|---------:|--------:|
|       1357 | 107203   |  206290 |
|       2367 |  66665.1 |  146005 |
|       2822 | 115592   |  220315 |
|       2126 |  85288.1 |  183038 |
|       1544 |  67889.9 |  150646 |

### Time series forecasting

```python
from darts.datasets import ElectricityConsumptionZurichDataset

# Load a forecasting dataset
ts = ElectricityConsumptionZurichDataset().load()
ts = ts.resample("h")

# Split the dataset into features X and target y
X = ts.drop_columns(["Value_NE5", "Value_NE7"])
y = ts["Value_NE5"]  # NE5 = Household energy consumption

# Add categorical features to X
X = X.add_holidays(country_code="CH")
X = X.add_datetime_attribute("month")
X = X.add_datetime_attribute("dayofweek")
X = X.add_datetime_attribute("hour")
```

Printing the head of `X.pd_dataframe()` yields:

| Timestamp           |   Hr [%Hr] |   RainDur [min] |   StrGlo [W/m2] |   T [°C] |   WD [°] |   WVs [m/s] |   WVv [m/s] |   p [hPa] |   holidays |   month |   dayofweek |   hour |
|:--------------------|-----------:|----------------:|----------------:|---------:|---------:|------------:|------------:|----------:|-----------:|--------:|------------:|-------:|
| 2022-08-30 20:00:00 |       70.2 |             0.0 |             0.0 |     19.9 |    290.2 |         1.7 |         1.5 |     968.5 |        0.0 |     7.0 |         1.0 |   20.0 |
| 2022-08-30 21:00:00 |       70.1 |             0.0 |             0.0 |     19.5 |    239.2 |         1.0 |         0.7 |     968.1 |        0.0 |     7.0 |         1.0 |   21.0 |
| 2022-08-30 22:00:00 |       71.3 |             0.0 |             0.0 |     19.5 |     28.9 |         1.5 |         1.3 |     967.9 |        0.0 |     7.0 |         1.0 |   22.0 |
| 2022-08-30 23:00:00 |       80.4 |             0.0 |             0.0 |     18.9 |     24.3 |         1.6 |         1.1 |     967.9 |        0.0 |     7.0 |         1.0 |   23.0 |
| 2022-08-31 00:00:00 |       81.6 |             1.0 |             0.0 |     18.7 |    293.5 |         0.9 |         0.3 |     967.8 |        0.0 |     7.0 |         2.0 |    0.0 |

```python
from conformal_tights import DartsForecaster, ConformalCoherentQuantileRegressor
from pandas import Timestamp
from xgboost import XGBRegressor

# Split the dataset into train and test
test_cutoff = Timestamp("2022-06-01")
y_train, y_test = y.split_after(test_cutoff)
X_train, X_test = X.split_after(test_cutoff)

# Now let's:
# 1. Create an sklearn regressor of our choosing, in this case `XGBRegressor`
# 2. Add conformal quantile prediction to the regressor with `ConformalCoherentQuantileRegressor`
# 3. Add probabilistic forecasting to the conformal regressor with `DartsForecaster`
regressor = XGBRegressor()
conformal_regressor = ConformalCoherentQuantileRegressor(estimator=regressor)
forecaster = DartsForecaster(
    model=conformal_regressor,
    # Add last 5 days of the target to the prediction features
    lags=5 * 24,
    # Add the current day's covariates to the prediction features
    lags_future_covariates=[0],
    # Indicate which of the covariates are categorical
    categorical_future_covariates=["holidays", "month", "dayofweek", "hour"],
)

# Fit the forecaster given the target and covariates
forecaster.fit(y_train, future_covariates=X_train)
```

```python
# Make a probabilistic forecast 5 days into the future
forecast = forecaster.predict(n=5 * 24, future_covariates=X_test, num_samples=500)
```

Let's visualize the forecast and its prediction interval on the test set:

<img src="https://github.com/radix-ai/conformal-tights/assets/..." width="512">

<details>
<summary>Expand to see the code that generated the graph above</summary>

```python
import matplotlib.pyplot as plt
import matplotlib.ticker as ticker

%config InlineBackend.figure_format = "retina"
plt.rc("font", family="DejaVu Sans", size=10)
plt.figure(figsize=(8, 4.5))
y_train[-2 * 24 :].plot(label="Actual (train)")
y_test[: len(forecast)].plot(label="Actual (test)")
forecast.plot(label="Forecast with 90% PI")
plt.gca().set_xlabel("")
plt.gca().yaxis.set_major_formatter(ticker.FuncFormatter(lambda x, _: f"{x/1000:,.0f} MWh"))
plt.gca().tick_params(axis="both", labelsize=10)
plt.gca().spines["top"].set_visible(False)
plt.gca().spines["right"].set_visible(False)
legend = plt.legend(loc="upper right", title="Energy consumption")
legend.get_title().set_fontweight("bold")
plt.tight_layout()
plt.show()
```
</details>

## Contributing

<details>
<summary>Prerequisites</summary>

<details>
<summary>1. Set up Git to use SSH</summary>

1. [Generate an SSH key](https://docs.github.com/en/authentication/connecting-to-github-with-ssh/generating-a-new-ssh-key-and-adding-it-to-the-ssh-agent#generating-a-new-ssh-key) and [add the SSH key to your GitHub account](https://docs.github.com/en/authentication/connecting-to-github-with-ssh/adding-a-new-ssh-key-to-your-github-account).
1. Configure SSH to automatically load your SSH keys:
    ```sh
    cat << EOF >> ~/.ssh/config
    Host *
      AddKeysToAgent yes
      IgnoreUnknown UseKeychain
      UseKeychain yes
    EOF
    ```

</details>

<details>
<summary>2. Install Docker</summary>

1. [Install Docker Desktop](https://www.docker.com/get-started).
    - Enable _Use Docker Compose V2_ in Docker Desktop's preferences window.
    - _Linux only_:
        - Export your user's user id and group id so that [files created in the Dev Container are owned by your user](https://github.com/moby/moby/issues/3206):
            ```sh
            cat << EOF >> ~/.bashrc
            export UID=$(id --user)
            export GID=$(id --group)
            EOF
            ```

</details>

<details>
<summary>3. Install VS Code or PyCharm</summary>

1. [Install VS Code](https://code.visualstudio.com/) and [VS Code's Dev Containers extension](https://marketplace.visualstudio.com/items?itemName=ms-vscode-remote.remote-containers). Alternatively, install [PyCharm](https://www.jetbrains.com/pycharm/download/).
2. _Optional:_ install a [Nerd Font](https://www.nerdfonts.com/font-downloads) such as [FiraCode Nerd Font](https://github.com/ryanoasis/nerd-fonts/tree/master/patched-fonts/FiraCode) and [configure VS Code](https://github.com/tonsky/FiraCode/wiki/VS-Code-Instructions) or [configure PyCharm](https://github.com/tonsky/FiraCode/wiki/Intellij-products-instructions) to use it.

</details>

</details>

<details open>
<summary>Development environments</summary>

The following development environments are supported:

1. ⭐️ _GitHub Codespaces_: click on _Code_ and select _Create codespace_ to start a Dev Container with [GitHub Codespaces](https://github.com/features/codespaces).
1. ⭐️ _Dev Container (with container volume)_: click on [Open in Dev Containers](https://vscode.dev/redirect?url=vscode://ms-vscode-remote.remote-containers/cloneInVolume?url=https://github.com/radix-ai/conformal-tights) to clone this repository in a container volume and create a Dev Container with VS Code.
1. _Dev Container_: clone this repository, open it with VS Code, and run <kbd>Ctrl/⌘</kbd> + <kbd>⇧</kbd> + <kbd>P</kbd> → _Dev Containers: Reopen in Container_.
1. _PyCharm_: clone this repository, open it with PyCharm, and [configure Docker Compose as a remote interpreter](https://www.jetbrains.com/help/pycharm/using-docker-compose-as-a-remote-interpreter.html#docker-compose-remote) with the `dev` service.
1. _Terminal_: clone this repository, open it with your terminal, and run `docker compose up --detach dev` to start a Dev Container in the background, and then run `docker compose exec dev zsh` to open a shell prompt in the Dev Container.

</details>

<details>
<summary>Developing</summary>

- This project follows the [Conventional Commits](https://www.conventionalcommits.org/) standard to automate [Semantic Versioning](https://semver.org/) and [Keep A Changelog](https://keepachangelog.com/) with [Commitizen](https://github.com/commitizen-tools/commitizen).
- Run `poe` from within the development environment to print a list of [Poe the Poet](https://github.com/nat-n/poethepoet) tasks available to run on this project.
- Run `poetry add {package}` from within the development environment to install a run time dependency and add it to `pyproject.toml` and `poetry.lock`. Add `--group test` or `--group dev` to install a CI or development dependency, respectively.
- Run `poetry update` from within the development environment to upgrade all dependencies to the latest versions allowed by `pyproject.toml`.
- Run `cz bump` to bump the package's version, update the `CHANGELOG.md`, and create a git tag.

</details>

# Instrumenting phoenix with Prometheus
## Why?
- Instrumentation is about collecting metrics for your application. You can collect database query time, response time, basically anything you want. If you have a particular thing to measure, prometheus elixir package makes it really easy to collect metrics for your phoenix app.
- Even if there isn't anything in particular you want to measure, some prometheus adapters provide you with more granular metrics out of the box (Ecto's query's average time, controller average response time, view average render time…)

We will cover the installation of the prometheus adapters on your phoenix app and the deployment with docker.
We will be
- [collecting ecto metrics](https://github.com/deadtrickster/prometheus-ecto)
- [collecting controller and view metrics](https://github.com/deadtrickster/prometheus-phoenix)

## How 
add the following dependencies to your mix.exs

```
{:prometheus, "~> 4.0", override: true},
{:prometheus_ex, "~> 3.0"},
{:prometheus_ecto, "~> 1.0"},
{:prometheus_phoenix, "~> 1.2"},
{:prometheus_plugs, "~> 1.0"},
{:prometheus_process_collector, "~> 1.3"},
```
(here we are using override just to use the latest version of prometheus itself)
in your `config/config.exs` add 

```
# under your endpoint config
config :my_app, MyAppWeb.Endpoint,
  # leave your current settings here, just add the following under
  instrumenters: [MyApp.PhoenixInstrumenter]


config :prometheus, MyApp.PhoenixInstrumenter,
controller_call_labels: [:controller, :action],
duration_buckets: [
10,
25,
50,
100,
250,
500,
1000,
2500,
5000,
10_000,
25_000,
50_000,
100_000,
250_000,
500_000,
1_000_000,
2_500_000,
5_000_000,
10_000_000
],
registry: :default,
duration_unit: :microseconds


config :prometheus, MyApp.PipelineInstrumenter,


labels: [:status_class, :method, :host, :scheme, :request_path],
duration_buckets: [
10,
100,
1_000,
10_000,
100_000,
300_000,
500_000,
750_000,
1_000_000,
1_500_000,
2_000_000,
3_000_000
],
registry: :default,
duration_unit: :microseconds


config :my_app, MyApp.Repo, loggers: [MyApp.RepoInstrumenter, Ecto.LogEntry]
```

`MyApp.RepoInstrumenter` , `MyApp.PhoenixInstrumenter` and `MyApp.PipelineInstrumenter` correspond to three modules that we will create next (of course you can name the modules whatever you want). I put these modules under `myapp/instrumenters/repo_instrumenter` for example, but that is also a matter of taste.
add a `myapp/instrumenters/repo_instrumenter.ex` with the following

```
defmodule MyApp.RepoInstrumenter do
@moduledoc """
Instrumentation of Ecto activity.
"""
use Prometheus.EctoInstrumenter
end
```

add a `myapp/instrumenters/phoenix_instrumenter.ex` with the following

```
defmodule MyApp.PhoenixInstrumenter do
@moduledoc """
Provides instrumentation for Phoenix specific metrics.
"""
use Prometheus.PhoenixInstrumenter
end
```

add a `myapp/instrumenters/pipeline_instrumenter.ex` with the following

```
defmodule MyApp.PipelineInstrumenter do
@moduledoc """
Instrumentation for the entire plug pipeline.
"""
use Prometheus.PlugPipelineInstrumenter
@spec label_value(:request_path, Plug.Conn.t()) :: binary
def label_value(:request_path, conn) do
conn.request_path
end
end
```

now in `myapp/application.ex` add
```
alias MyApp{
  PhoenixInstrumenter,
  PipelineInstrumenter,
  PrometheusExporter,
  RepoInstrumenter,
  VersionInstrumenter
}
#!! add these inside the start function (anywhere is fine)
require Prometheus.Registry
PhoenixInstrumenter.setup()
PipelineInstrumenter.setup()
RepoInstrumenter.setup()


if :os.type() == {:unix, :linux} do
Prometheus.Registry.register_collector(:prometheus_process_collector)
end


PrometheusExporter.setup()
```

## Setting up prometheus and grafana
prometheus will collect the metrics, but you need something to visualize them. Let's use grafana and let's set everything up with docker.
if you haven't set up your deployment with docker, check the [docker part](/deployement/docker.md).
once you have a docker-compose.yml file, add the following under the services list

```
prometheus:
  image: prom/prometheus
  ports:
  - "9090:9090"
  volumes:
  - ./prometheus/:/etc/prometheus/
  - prometheus_data:/prometheus
  command:
  - '--config.file=/etc/prometheus/prometheus.yml'
  - '--storage.tsdb.path=/prometheus'
  - '--  web.console.libraries=/usr/share/prometheus/console_libraries'
  - '--web.console.templates=/usr/share/prometheus/consoles'
  restart: always


grafana:
  image: grafana/grafana:5.1.0
  user: "104"
  volumes:
  - grafana_data:/var/lib/grafana
  - ./grafana/provisioning/:/etc/grafana/provisioning/
  links:
  - prometheus
  ports:
  - "3000:3000"
  restart: always
```

add a prometheus/prometheus.yml file on your local with (this is just the default prometheus config file, feel free to customize)

```
global:
scrape_interval:     15s # By default, scrape targets every 15 seconds.
# Attach these labels to any time series or alerts when communicating with
# external systems (federation, remote storage, Alertmanager).
external_labels:
monitor: 'codelab-monitor'
# A scrape configuration containing exactly one endpoint to scrape:
# Here it's Prometheus itself.
scrape_configs:
# The job name is added as a label `job=<job_name>` to any timeseries scraped from this config.
- job_name: 'prometheus'
# Override the global default and scrape targets from this job every 5 seconds.
scrape_interval: 5s
static_configs:
- targets: ['web:4000']
```
so basically we are creating a volume to store the prometheus config and we are linking prometheus and grafana services together. [Here](deployment/docker-compose.yml) is how your docker-compose file should look like.

verify that your docker setup is working with `docker-compose up`
you can go to `localhost:3000` in your browser and should see the grafana interface. Default user and password is admin (for both).

Next you need to add a datasource to Grafana. On the left, on the settings icon, click on datasource to add a new datasource. Name it whatever you want (ex MyApp Instrumentation). choose prometheus as the source type. Use http://prometheus:9090 as the url. And click on save & test. It should confirm that the setup is correct.

Next you need to add a dashboard. You can create your own dashboard, but let's use some open source one at first. On Grafana's interface on the left, click on the + and import. You can directly paste some JSON in there. There is [a repo with lots of wonderful dashboards](https://github.com/deadtrickster/beam-dashboards). Three are particularly useful
[the BEAM dashboard](https://github.com/deadtrickster/beam-dashboards/blob/master/BEAM-dashboard.json) (you can directly copy and paste the json from the repo into the import part of grafana)
[The beam allocators dashboard](https://github.com/deadtrickster/beam-dashboards/blob/master/BEAM-memory_allocators.json)
and the [elixir dashboard](https://github.com/deadtrickster/beam-dashboards/blob/master/Elixir-dashboard.json) (which is the one with the controllers and ecto metrics)
These three dashboard will work out of the box with our setup.

That's it! Check out your dashboards and bask in your own glory 🙌

based on the [really good article written in 2016](https://aldusleaf.org/2016-09-30-monitoring-elixir-apps-in-2016-prometheus-and-grafana.html) (I wrote an article, just because I got a stuck when I tried to apply that article). By the way, if you ever get stuck, the #prometheus channel on elixir slack, is just awesome.

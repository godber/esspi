# TeraPi - Raspberry Pi SenseHat Cluster Status Monitor

This project gathers high level Elasticsearch cluster status information from
multiple ES clusters and then pushes that high level data to a backend,
which is then retrieved by another tool that visualizes the status information
on the RaspberryPi SenseHat which has an 8x8 RGB LED matrix.

There are two primary components:

* `terapi_get` - Python script that queries the clusters and posts metric
  information to the central location.
* `terapi_show` - Python script that retrieves the stored cluster metrics
  and "displays" that information on the SenseHat.

The general concept assumes that we display `green`, `yellow`, or `red` statuses
derived from four metrics for up to sixteen clusters.

## Shared Configuration File

Both utilities share the same configuration file, which is shown below.  This
file can contain multiple cluster definitions.

```json
{
    "cluster01" : {
        "cluster_url" : "http://cluster01:9200",
        "shared_location" : {
                "type": "elasticsearch5",
                "url": "http://shared_location:9200/terapi/cluster",
                "auth": ""
            },
        "thresholds": {
            "relo": {
                "operator": ">",
                "values": [0, 5]
            },
            "init": {
                "operator": ">",
                "values": [0, 5]
            },
            "unassign": {
                "operator": ">",
                "values": [0, 5]
            }
        }
    }
}
```

## terapi_get

This is a daemonized process that runs in the datacenter where the clusters are
located and gather the needed information at the frequency provided in the
configuration file.

* read clusters from configuration file
* query specified clusters
* post metric data to backend

The current status for each cluster will be stored in it's own uniquely named
JSON file, like `cluster01.json`, with the following structure:

```json
{
    "timestamp": "2017-01-29T17:26:17.356Z",
    "metrics": {
        "status": "green",
        "relo": 0,
        "init": 0,
        "unassign": 0
    }
}
```

## terapi_show

This is a daemonized process that runs on the RaspberryPu and reads cluster
status metrics from the backend at the frequency provided in the
configuration file and then "display" that status information on the
RaspberryPi SenseHat.

* read configuration file
* retrieve the necessary cluster information
* interpret the cluster metrics using the `thresholds` information in the
  configuration file
* update the SenseHat

The SenseHat is an 8x8 grid of individually addressable LEDs.  TeraPi will
display the status of each cluster using 4 LEDs.  The ultimate goal would be to
make the exact meaning of each LED configurable but for the time being the
metrics will be:

* `status` - Cluster status: `green`, `yellow`, `red`
* `relo` - Number of shards in `relocation` state.
* `init` - Number of shards in `init` state.
* `unassign` - Number of shards in `unassign` state.

Each cluster configuration has a list of two `thresholds` that are configurable
for each metric.  Each threshold also has an associated operator.  The
assumption is that using the specified operator the first threshold in the list
represents the transition from `green` to `yellow` and the second is the
transition from `yellow` to `red`.

## Backend

The backend is the backend datastore where the data is posted.  Right now I am
assuming this will be another `elasticsearch5` that is accessible by both
scripts.

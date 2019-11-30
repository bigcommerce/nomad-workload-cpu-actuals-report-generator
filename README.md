# nomad-workload-cpu-actuals-report

## Summary

A tool used for balacning workloads -- finding over and under allocated jobs/tasks particularly when the Nomad Client cluster has non-uniform CPU classes / clockspeeds.

## Usage

```
./nomadWorkloadCPUActualsReport --help
error: Missing required option: e
usage: ./nomadWorkloadCPUActualsReport -e <comma delimited environments> <other args>
Nomad Workload CPU Actuals Report
 -adstats,--advancedStats              Writes advanced stats including Std Dev, Kurtosis, and Skewness
 -avg,--avgKnownNomadClients           Use instead of --nomadFallbackClock to average the known Nomad clients for historical workloads
 -d1d,--disableOneDayQuery             Disable 1 day query
 -d1h,--disableOneHourQuery            Disable 1 hour query
 -d30d,--disableThirtyDayQuery         Disable 30 day query
 -d7d,--disableSevenDayQuery           Disable 7 day query
 -e,--environments <arg>               Environments (comma delimited)
 -e60d,--enableSixytDayQuery           Enable 60 day query
 -e90d,--enableNinetyDayQuery          Enable 90 day query
 -fbc,--nomadFallbackClock <arg>       The approximate clockspeed of out-of-service Nomad nodes that were used to run historical workloads tracked in Prometheus [defaults to 2300]
 -h,--help                             Usage Information
 -jobs,--targetJobs <arg>              Target specific jobs (comma delimited)
 -jst,--taskSleepTime <arg>            Task sleep time in seconds [defaults to 5]
 -miss,--includeMissRate               Write the miss rate
 -nc,--nomadTLSCertFilename <arg>      Nomad TLS Key Filename [defaults to %env%.nomad.key]
 -nca,--nomadTLSCACertFilename <arg>   Nomad TLS CA Certificate Filename [defaults to nomadca.crt]
 -nh,--nomadHost <arg>                 Nomad host [defaults to https://nomad.service.%env%-dc1.consul:4646]
 -nk,--nomadTLSKeyFilename <arg>       Nomad TLS Certificate Filename [defaults to %env%.nomad.crt]
 -ph,--prometheusHost <arg>            Prometheus host [defaults to http://prometheus-read.service.%env%-dc1.consul:9090]
 -qst,--querySleepTime <arg>           Query sleep time in seconds [defaults to 5]
Environment queries are run in parallel to reduce report generation time. Use %env% to inject environment into --nomadTLSKeyFilename, --nomadTLSCertFilename, --nomadHost, --prometheusHost
```

## Prometheus Queries

There are two metrics pulled out of prometheus:

1. `nomad_client_allocs_cpu_total_percent` - A guage of the percentage of the total CPU resources consumed by the task across all cores.
2. `nomad_client_allocs_memory_usage` - A guage of the total amount of memory bytes used by the task. 

For more information see [the Nomad metrics](https://www.nomadproject.io/docs/telemetry/metrics) page.

## Queries

Depending on what intervals are enabled per the aforementioned list of arguments, a number of ranges are used. By default the following are enabled:

- 1 Hour
- 1 Day
- 7 Days
- 30 Days

These flags can be disabled. Also available are 60 day and 90 day windows, which require explicitly enabling (are disabled by default). The queries all use `avg_over_time` over the two metrics for various range vector selectors (for time duration of the input metrics) and windows/range selectors for the `avg_over_time` function:

| Period     | Range Selectors  |
| ---------- | ---------------- |
| 1 Hour     | `[5s])[1h:5s]`   |
| 1 Day      | `[1m])[1d:1m]`   |
| 7 Days     | `[1m])[7d:1m]`   |
| 30 Days    | `[1h])[30d:6h]`  |
| 60 Days    | `[1h])[60d:12h]` |
| 90 Days    | `[1h])[90d:24h]` |

Query examples as issued to `/api/v1/query?query=`:

```
avg_over_time(nomad_client_allocs_cpu_total_percent{task="container-job-x"}[5s])[1h:5s]
avg_over_time(nomad_client_allocs_memory_usage{task="container-job-x"}[5s])[1h:5s]
avg_over_time(nomad_client_allocs_cpu_total_percent{task="container-job-x"}[1m])[1d:1m]
avg_over_time(nomad_client_allocs_memory_usage{task="container-job-x"}[1m])[1d:1m]
avg_over_time(nomad_client_allocs_cpu_total_percent{task="container-job-x"}[1m])[7d:1m]
avg_over_time(nomad_client_allocs_memory_usage{task="container-job-x"}[1m])[7d:1m]
avg_over_time(nomad_client_allocs_cpu_total_percent{task="container-job-x"}[1h])[30d:6h]
avg_over_time(nomad_client_allocs_memory_usage{task="container-job-x"}[1h])[30d:6h]
avg_over_time(nomad_client_allocs_cpu_total_percent{task="container-job-x"}[1h])[60d:12h]
avg_over_time(nomad_client_allocs_memory_usage{task="container-job-x"}[1h])[60d:12h]
avg_over_time(nomad_client_allocs_cpu_total_percent{task="container-job-x"}[1h])[90d:24h]
avg_over_time(nomad_client_allocs_memory_usage{task="container-job-x"}[1h])[90d:24h]
```

## Understand the report

Reports are output with the timestamp from which the report generation process was started. Reports have a tab for each environment passed to the command line application. There are shaded or colored sections of the workbook that compare the following values to the `Nomad Job -> Task Group -> Tasks's` Allocated MHz:

1. Mean
2. 50th Percentile
3. 95th Percentile
4. Max

... with the colors:

- green: meaning a value is somewhere bewteen 0 and the Allocated MHz, richer meaning further (0 is darkest or richest green while closest to Allocated MHz is more green/white or white)
- yellow: meaning a value is somewhere between the Allocated MHz and 2x the Allocated MHz, richer meaning further (2x Allocated MHz is darkest or richest yellow, while closest to Allocated MHz is more yellow/white or white)
- red: meaning a value is or is greather than 2x the Allocated MHz

### See Also

- [Using Prometheus to Monitor Nomad Metrics](https://www.nomadproject.io/guides/operations/monitoring-and-alerting/prometheus-metrics.html)
- [Shape of data](https://brownmath.com/stat/shape.htm)
- [Understanding Descriptive Statistics](https://towardsdatascience.com/understanding-descriptive-statistics-c9c2b0641291)

# devops-case-study

## Notes

The original project from the GitHub repo showed some issues on my local machine regarding the timings of pod startup and the correct initialization of the database schema. I added an init container based wait to the `backend-api` to resolve that issue and increased the bootstrap wait timeouts.

Password for login into Grafana (via port forward) is hardcoded admin/changeme and of course not meant for production usage. 

# Goal and Context

Provide a monitoring stack for the services `backend-api` and `ml-api` that are currently running in a K8S cluster. Both services already expose Prometheus compatible metrics endpoints, `backend-api` uses a PostgreSQL database as storage backend. The task is timeboxed to roughly 4 hours. The cluster and its workloads are manged and deployed via `fluxcd` 

## Tech Stack

[kube-prometheus-stack](https://github.com/prometheus-community/helm-charts/tree/main) provides a consistent out-of-the-box experience for a full Grafana/Prometheus Stack. It already automatically scrapes key metrics from the K8S cluster where it is deployed and is easy to integrate with services exposing Prometheus metrics via `ServiceMonitor`. Datasources are already wired up, and managing custom dashboards via ConfigMaps is also supported, so the solution is fully IaC managed. The scraped data is (mostly) compatible with popular and tested Dashboards for K8S and PostgreSQL. Although multi-cluster deployments might need some additional work to ensure consistent labeling and tagging, given the time constraints kube-prometheus-stack is the best choice in this particular context. The provided Helm charts are also a naturally fit for the flux `HelmRelease` integration.  

## Dashboard design

The dashboard `devops-case-study` is divided into groups, roughly following the layers of the deployed application, from the application itself, through the database layer down into Kubernetes pods. Focus of the dashboard is to show the health of the two deployed services and its dependencies not the overall cluster state. When this dashboard shows issues or abnormal metrics, more detailed dashboard should be used for further investigation, like the included K8S dashboard in case of pod scheduling or pod performance issues, or the PostgreSQL dashboard for deeper database insights. 


The overview panel is intended as a quick-one-stop status, everything green means the system is up and running. The metrics here should enable people without any context to check if the services are up and healthy. The panel is als suitable for e.g. a central wall-display (in a world where not everybody is remote).

An important part of the overview is, to not only show the technical metrics like response times, or error rates, but also a business metric like `ml-api predictions`. This gives an indication that the service is not only running on a technical level, but also that it is actually servings its business purpose.

The next level `service requests` provides a view not only on the current state, but also the recent history, giving a quick indication whether the services are degrading or recovering, e.g. response times or error rates going up or down over time.

The next layer is the database that underpins the `backend-api` showing some key DB metrics that might influence the service performance. Not strictly needed because most database issues will also be revealed in the service metrics. 

The logs view shows an aggregated view of all error logs (services and database). It's important to note that this log only provides value when all services adhere to the same semantics for log level. The assumption here is that everything that logged as error needs to be investigated and rectified by an operator. A constant stream of errors logs will lead to people just ignoring the output. 

The `pod activity` shows all abnormal information about pod scheduling and the pod lifecycle. This view should always be empty/green, if there is something going on it needs to be investigated. 


Finally, the `pod resource` layers shows how the pods are utilizing the assigned resources and gives indication when resource requests and limit are reached and might need to be adjusted.


## Alarms

The alarms are chosen to give a high-level summary to the on-call person without overwhelming them. Having to many alarms can result in alert fatigue where people start just acknowledging alarms because there are too many or get overwhelmed. Also having constant alarms going of can lead to normalization of deviation where a certain rate of alarms is viewed as an accepted state and important alarms start slipping through. 

In a real scenario the alarm thresholds as well as the metrics dashboard need feedback from real operations to constantly refine acceptable thresholds and the definition of which metrics indicate a smoothly running platform.

The alarm design follows the layers from the dashboard, and tries to have a few selected metrics that represent each layer. Especially the pod activity might lead to some noise and will need refinement in the beginning or depending on the scaling and deployment activity another approach. Here again I only added an explicit OOMKilled alarm, the other alarms like pod churn and failed pods pod restarts need some real life values from daily cluster operation to be useful as alarm indicators.

 The alarm `severity` can be used to drive the notification levels. A pod nearing its limit might be something an operator should look at during normal business hours, but nothing he needs to be woken up for during the night. An error rate of 100% on the other hand needs to be acted on as soon as possible.

The error logs are explicitly not included in the alarms, as this typically needs some real-life experience over the typical error rates and eventually some filtering to again avoid alarm-fatigue. 

## Open Topics

* the percentiles in the overview are not 100% accurate, since the service only returns fixed response-time buckets, making the display 95% percentile a relatively accurate but still approximated value, the detailed buckets are visible in the history panel below
* For the alarms as well as for the metrics, once more data is available a comparison with previous points in time might also be valuable. E.g. when the application has a predictable weekly recurring load-curve it could be useful to compare the current req/s with the same range a week ago to get a feeling if everything is whiting normal bounds. 
* There are some default alerts firing caused by some missing components in K3S that must be taken care of, to avoid alarm fatigue for the on-call personal
* the dashboard might become crowded once the service gains more endpoints and pods, then a more high level per-service summary with service specific dashboards might be a good idea
* monitoring of the K8S nodes is explicitly not a part the dashboard, this is more in the scope of the team operating the cluster not for the teams that runs the services. If a node has issues like CPU or I/O pressure this will show up indirectly in the service metrics anyway.
* the credentials for the postgres-exporter should be a dedicated RO role and not shared with the application
* add dedicated dashboards for `backen-api` and `ml-api` with full service log, and more service specific metrics.
* add drilldown from the `devops-case-study` graphs to the dashboards with more details (K8S, PostgreSQL, Service dashboards) 
* adding tracing could ease debugging allowing the operator e.g. to correlate a service request to a specific database query, or in case of cross-service communication to trace requests through all services 
* currently the log error level is derived from the log text, structured logging (JSON) from service and metrics would make this more explicit and also enable the services to log additional metadata, e.g. when an error is logged also provide some context about which customer was affected.
* the alarms would benefit from some links to dashboards for deeper root-cause investigation
* the alarm and metrics that catches the false positive case (not logs/metrics) could be improved on. currently it only fires when everything drops to zero, real life scenarios might be more very close to zero instead of absolutely zero 
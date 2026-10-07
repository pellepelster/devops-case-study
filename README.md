# devops-case-study

## Goal and Context

Provide a monitoring stack for the services `backend-api` and `ml-api` that are currently running in a K8S cluster. Both services already expose Prometheus compatible metrics endpoints, `backend-api` uses a PostgreSQL database as storage backend. The task is timeboxed to roughly 4 hours. The cluster and its workloads are managed and deployed via `fluxcd`.

## Tech Stack

[kube-prometheus-stack](https://github.com/prometheus-community/helm-charts/tree/main) provides a consistent out-of-the-box experience for a full Grafana/Prometheus Stack. It already automatically scrapes key metrics from the K8S cluster where it is deployed and is easy to integrate with services exposing Prometheus metrics via `ServiceMonitor`. Datasources are already wired up, and managing custom dashboards via ConfigMaps is also supported, so the solution is fully IaC managed. The scraped data is (mostly) compatible with popular and tested Dashboards for K8S and PostgreSQL. Although multi-cluster deployments might need some additional work to ensure consistent labeling and tagging, given the time constraints kube-prometheus-stack is the best choice in this particular context. The provided Helm charts are also a natural fit for the flux `HelmRelease` integration.

## Dashboard design

The dashboard `devops-case-study` is divided into groups, roughly following the layers of the deployed application, from the application itself, through the database layer down into Kubernetes pods. Focus of the dashboard is to show the health of the two deployed services and their dependencies not the overall cluster state. When this dashboard shows issues or abnormal metrics, more detailed dashboards should be used for further investigation, like the included K8S dashboard in case of pod scheduling or pod performance issues, or the PostgreSQL dashboard for deeper database insights.

The overview panel is intended as a quick one-stop status, everything green means the system is up and running. The metrics here should enable people without any context to check if the services are up and healthy. The panel is also suitable for e.g. a central wall-display (in a world where not everybody is remote).

An important part of the overview is to not only show the technical metrics like response times, or error rates, but also a business metric like `ml-api predictions`. This gives an indication that the service is not only running on a technical level, but also that it is actually serving its business purpose.

The next level `service requests` provides a view not only on the current state, but also the recent history, giving a quick indication whether the services are degrading or recovering, e.g. response times or error rates going up or down over time.

The next layer is the database that underpins the `backend-api` showing some key DB metrics that might influence the service performance. Not strictly needed because most database issues will also be revealed in the service metrics.

The logs view shows an aggregated view of all error logs (services and database). It's important to note that this log only provides value when all services adhere to the same semantics for log level. The assumption here is that everything that is logged as error needs to be investigated and rectified by an operator. A constant stream of error logs will lead to people just ignoring the output.

The `pod activity` shows all abnormal information about pod scheduling and the pod lifecycle. This view should always be empty/green, if there is something going on it needs to be investigated.

Finally, the `pod resources` layer shows how the pods are utilizing the assigned resources and gives an indication when resource requests and limits are reached and might need to be adjusted.

## Alerts

The alerts are chosen to give a high-level summary to the on-call person without overwhelming them. Having too many alerts can result in alert fatigue where people start just acknowledging alerts because there are too many or get overwhelmed. Also having constant alerts going off can lead to normalization of deviation where a certain rate of alerts is viewed as an accepted state and important alerts start slipping through.

In a real scenario the alert thresholds as well as the metrics dashboard need feedback from real operations to constantly refine acceptable thresholds and the definition of which metrics indicate a smoothly running platform.

The alert design follows the layers from the dashboard, and tries to have a few selected metrics that represent each layer. Especially the pod activity might lead to some noise and will need refinement in the beginning or depending on the scaling and deployment activity another approach. Here again I only added an explicit OOMKilled alert, the other alerts like pod churn and failed pods or pod restarts need some real-life values from daily cluster operation to be useful as alert indicators.

The alert `severity` can be used to drive the notification levels. A pod nearing its limit might be something an operator should look at during normal business hours, but nothing they need to be woken up for during the night. An error rate of 100% on the other hand needs to be acted on as soon as possible.

The error logs are explicitly not included in the alerts, as this typically needs some real-life experience over the typical error rates and possibly some filtering to again avoid alert fatigue.

## Open Topics

* the percentiles in the overview are not 100% accurate, since the service only returns fixed response-time buckets and the percentiles are interpolated between fixed histogram buckets, the detailed buckets are visible in the history panel below
* For the alerts as well as for the metrics, once more data is available a comparison with previous points in time might also be valuable. E.g. when the application has a predictable weekly recurring load-curve it could be useful to compare the current req/s with the same range a week ago to get a feeling if everything is within normal bounds.
* There are some default alerts firing caused by some missing components in K3S that must be taken care of, to avoid alert fatigue for the on-call personnel
* the dashboard might become crowded once the service gains more endpoints and pods, then a more high level per-service summary with service specific dashboards might be a good idea
* monitoring of the K8S nodes is explicitly not a part of the dashboard, this is more in the scope of the team operating the cluster not for the teams that run the services. If a node has issues like CPU or I/O pressure this will show up indirectly in the service metrics anyway.
* the credentials for the postgres-exporter should be a dedicated RO role and not shared with the application
* add dedicated dashboards for `backend-api` and `ml-api` with full service log, and more service specific metrics.
* add drilldown from the `devops-case-study` graphs to the dashboards with more details (K8S, PostgreSQL, Service dashboards)
* adding tracing could ease debugging allowing the operator e.g. to correlate a service request to a specific database query, or in case of cross-service communication to trace requests through all services
* currently the log error level is derived from the log text, structured logging (JSON) from service and metrics would make this more explicit and also enable the services to log additional metadata, e.g. when an error is logged also provide some context about which customer was affected.
* the alerts would benefit from some links to dashboards for deeper root-cause investigation
* the alerts and metrics that catch the false positive case (not logs/metrics) could be improved. Currently they only fire when everything drops to zero, real life scenarios might be very close to zero instead of absolutely zero

## Notes

The original project from the GitHub repo showed some issues on my local machine regarding the timings of pod startup and the correct initialization of the database schema. I added an init container based wait to the `backend-api` to resolve that issue and increased the bootstrap wait timeouts.

Password for login into Grafana (via port forward) is hardcoded admin/changeme and of course not meant for production usage.

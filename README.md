### Demo deployment

[Alertmanager UI](https://alertmanager-ui-equinsuocha.cloud.okteto.net/)

[Grafana UI](https://grafana-equinsuocha.cloud.okteto.net/)

### Okteto deployment guide

1. Register an account in www.okteto.com
2. Install `okteto-cli`: https://www.okteto.com/docs/getting-started/#installing-okteto-cli
3. Configure `okteto-cli`: https://www.okteto.com/docs/getting-started/#configuring-okteto-cli-with-okteto-cloud
4. Get okteto kubeconfig: https://www.okteto.com/docs/getting-started/#configuring-okteto-cli-with-okteto-cloud
6. Set namespace value to your okteto namespace name in `overlays/demo-okteto/kustomization.yaml`
7. run `kubectl apply -k overlays/demo-okteto`

Once deployment is done, public components will be available
under following urls:
- https://alertmanager-ui-{{ your namespace name }}.cloud.okteto.net/
- https://grafana-{{ your namespace name }}.cloud.okteto.net/

### Private cluster deployment guide

1. Add namespace manifest to the resources
2. Make sure namespace is specified in your `kustomization.yaml`
3. Get you cluster kubeconfig
4. Override grafana and alertmanager-ui service manifests to expose it to consumers in a preferable way (i.e. NodePort)
5. run `kubectl apply -k overlays/your-overlay-folder`

### Stack overview

Components:
 - grafana: UI, alerting
 - mysql8: Grafana database
 - prometheus: metric collection, alerting
 - prometheus-alertmanager: alerting
 - karma-ui: alertmanager ui service
 - loki: log collection 
 - grafana-agent: metrics collection, log forwarding, log collection
 - vector: metrics collection, log forwarding, log collection

Known issues: due to okteto restrictions, ClusterRoles can't be created, so log scraping from k8s pod stdout won't work, this is a subject to customize for on-premises setup. General guideline would be to replace collector role defined in `base/roles/prometheus-k8s-sd.yaml` with a ClusterRole and create ClusterRoleBinding for grafana-agent or vector ServiceAccounts; in this case kubernetes logs config can be enabled in those services;

All components are deployed in a single instance, because there's a limitation on a number of pods allowed to run in demo plan.

### Train of thought

Since the goal was a bit abstract, I was struggling finding balance between execution time/effort and demonstration smoothness, since many quick solutions would render themselves barely portable and potentially can run into numerous environment issues when reproduced by the other party. So my initial goal was to keep it simple and portable. One of biggest roadblocks I expected to run into is specifically platform integration: how do we authenticate and execute deployment pipeline against the platform of choice. I was looking for smth with direct access to k8s control plain, and as I expected there were not too many free or even trial options (which I still don't want to subscribe to). I that was not the case, I might have required to learn some platfrom-specific APIs to fullfill that request. Luckily, there was one option, so I could cut down prerequisites to pretty much having kubectl installed and kubeconfig present on a machine to execute deployment against k8s cluster.

Once I got that off my way, I could focus on platform components configuration and wiring.
The main idea was to bring up a standard LGTM stack (without tracing part since it was not mentioned in task description) and demonstrate main components provisioned and operating. So there was a list of things to do:
- Configure Grafana with provisioned datasources, demo dashboard and custom administrator password;
- Configure prometheus with some predefined stack alerts;
- Configure prometheus scrapes for pods based on annotations;
- Configure alertmanager and alertmanager-ui as a slight metamonitoring touch: setup always firing alert rule, if it's not there, prometheus is down;
- Configure grafana to send logs to loki via vector syslog forwarder - because we're not permitted read node files and we still want to see logs in our feed;

This stack would bring a good half of "automatic" observability to every pod exposing metrics and annotated with specific annotations. If we had access to Node API scope, we would also have automatic log collection for all pods.

### Customization

The project is implemented in Kustomize. It allows to solve main customization points in context of the demo:
- configure namespace to deploy to;
- configure grafana admin password;

additionally it allows to override arbitrary configuration parts, as an example:
- grafana db credentials;
- prometheus alert rule files;

```
    namespace: equinsuocha          # set your namespace name here before deploying the project
    secretGenerator:
    - name: grafana-db-grafana-credentials
        literals:
        - username=grafana
        - password=grafana-secret    
    - name: grafana-admin-credentials
        literals:
        - username=admin            # set grafana administrator username here 
        - password=what$0ever       # set grafana administrator password here 
    resources:
    - ../../base/
    configMapGenerator:
    - name: prometheus-alert-rules
        behavior: replace                   
        files:                              #  list your alert rule files here
        - ./app-config/alertrules.yaml
```

Kustomize works for most of scenarios, i.e. for on-prem deployment Roles and RoleBindings could be replaced with ClusterRoles and ClusterRoleBindings to allow automatic log collection could have been implemented relatively easy.

However, the tradeoff behind this solution is that is does not embed any config templating, which results into some wonky things, like specifying plain-text db credentials in mysql-init file. This could have been solved by invoking templating engine before customize, but I found it unnecessary for the sake of demo, as it would require me to either move that wonkyness to other level (pass credentials from elsewhere) or provide centralized secret store for this project which is completely decoupled from target environment.

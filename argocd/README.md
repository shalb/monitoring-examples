# Info

Example code to integate with existing Argocd code.  
Pay attention to substring `example` in any file, it is just default value, which you should customize.

# Code structure

All applications in directory `argocd-apps/monitoring/`  
All additional custom code in directory `argocd-apps-manifests`  
You may not follow the same structure, but some logic should be applyed anaway, to maintain some order.

# How-tos

## Integrate Alertmanager with Opsgenie

* Request Opsgenie `API key` and `Team` from appropriate person.  
* You should copy example application (`argocd-apps/monitoring/kube-prometheus-stack.yaml`)  
  or only `config` section (`alertmanager.config`)  
* Replace in the config `exampleAPIKey` by the `API key` and `exampleTeam` by the `Team`
* Apply, create some sest alerts to check it

## Integrate Kubernetes cluster with Opsgenie heartbeat

* Request Opsgenie `API key` and `Project name` from appropriate person.  
* You should copy example application (`argocd-apps/monitoring/opsgenie-heartbeat.yaml`) 
* Replace value `exampleAPIKey` by the `API key` and `exampleProject` by the `Project name`
* Replace values `prometheus_test_url` and `alertmanager_test_url` if Prometheus and Alertmanager services  
  have different names in your cluster
* Apply, then ask appropriate person to check the heartbeat status

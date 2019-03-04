# Install Federation v2

Note this has been tested with federation v0.0.4. It may not work with other releases.

This playbook will istall Federation V2 and register a set of clusters to the federation control plane.
See an example of the inventory [here](./ansible/inventory) and customize it for your clusters.
Here is a minimum inventory:
```
clusters:
- name: <cluster_1_name>
  url: <cluster_1_master_api_url>
  username: <cluster_1_username>
  password: <cluster_1_password>
  federation_control_plane: true  
- name: <cluster_2_name>
  url: <cluster2_master_api_url>
  username: <cluster_2_username>
  password: <cluster_2_password>
  federation_control_plane: false 
```
You must have exaclty one cluster with the `federation_control_plane` variable set to `true`.

You can run the playbook as follows:

```
ansible-playbook -i <inventory> ./ansible/playbooks/install-federation/install-federation.yaml
```


# Deploy bookinfo with Federation V2 and Istio Multicluster

This tutorial assume that you have deploy Istio-multicluster. One way to do it is to follow the instructions [here](https://github.com/raffaelespazzoli/openshift-istio-multicluster).

The istio ingress gateway that exposes the app needs to be injected with a certificate. To generate the certificate we are going to use [cert-manager](https://github.com/jetstack/cert-manager) with a self-signing root CA.

## Enable admission controller
In case you have done it yet, it becomes now mandatory to have admission controllers enabled for the rest of the tutorial to work.
For each cluster run the following:
```
ansible masters -i <inventory> -m copy -a 'src=master-config.patch dest=/etc/origin/master/'
ansible masters -i <inventory> -m copy -a 'remote_src=yes src=/etc/origin/master/master-config.yaml dest=/etc/origin/master/master-config.yaml.prepatch'
ansible masters -i <inventory> -m shell -a 'oc ex config patch /etc/origin/master/master-config.yaml.prepatch -p "$(cat /etc/origin/master/master-config.patch)" > /etc/origin/master/master-config.yaml'
ansible masters -i <inventory> -m shell -a '/usr/local/bin/master-restart api && /usr/local/bin/master-restart controllers'
```

## Deploy cert-manager

You'll need cert-manager deployed in every cluster that your are joining. One way to do it is to run this playbook:
```
ansible-playbook -i <inventory> ./example/cert-manager/ansible/playbooks/deploy-cert-manager/deploy-cert-manager.yaml
```

## Deploy custom federated resources

In order to propagate the certificate object, we need to create the federated versions of it.
Assuming $FED_CLUSTER containes the context of the cluster with the federation control plane, run the following:
```
oc --context $FED_CLUSTER apply -f example/bookinfo/federated-crds/federated-certificate.yaml -n federation-system
```

## Deploy bookinfo
Notice that up to this point we made some preparation steps. These steps would be ususally done once and for all by a cluster administrator.
Now we can deploy our app.
Again we need some preparation steps because of the fact the istio pods cannot run with the default scc, this should be resolved in the future.
For each federated cluster run:
```
oc --context $CLUSTER adm policy add-scc-to-user privileged -z default -n bookinfo
oc --context $CLUSTER adm policy add-scc-to-user anyuid -z default -n bookinfo
oc --context $CLUSTER adm policy add-scc-to-user anyuid -z istio-ingressgateway-service-account -n bookinfo
oc --context $CLUSTER adm policy add-cluster-role-to-user istio-ingressgateway-istio-system -z istio-ingressgateway-service-account -n bookinfo
```
At this point we need to collect the info needed to process the federated bookinfo template (we assume here that the istio control plane and the federation control plane are in the same cluster, change accordingly if not true):
```
export PILOT_SVC_IP=$(oc --context $FED_CLUSTER -n istio-system get svc istio-pilot -o jsonpath='{.spec.clusterIP}')
export POLICY_SVC_IP=$(oc --context $FED_CLUSTER -n istio-system get svc istio-policy -o jsonpath='{.spec.clusterIP}')
export STATSD_SVC_IP=$(oc --context $FED_CLUSTER -n istio-system get svc istio-statsd-prom-bridge -o jsonpath='{.spec.clusterIP}')
export TELEMETRY_SVC_IP=$(oc --context $FED_CLUSTER -n istio-system get svc istio-telemetry -o jsonpath='{.spec.clusterIP}')
export ZIPKIN_SVC_IP=$(oc --context $FED_CLUSTER -n istio-system get svc jaeger-collector -o jsonpath='{.spec.clusterIP}')
export bookinfo_domain=example.com

```
Finally we can apply the template:
```
oc --context $FED_CLUSTER new-project bookinfo
helm template example/bookinfo/charts/federated-bookinfo --set zipkin_address=$ZIPKIN_SVC_IP,statsd_address=$STATSD_SVC_IP,pilot_address=$PILOT_SVC_IP,bookinfo_domain=$bookinfo_domain,cluster_array='\[raffa1\, raffa2\]' --namespace bookinfo | oc --context $FED_CLUSTER apply -f - -n bookinfo
```

## Introduction
This is an implementation guide for enable cluster metrics. For demonstration purposes, this guide uses ephemeral
storage for metrics. It also uses a self-signed certificate for Hawkular metrics. Neither of these are required and
 would not be used in a production environment.

## Prerequisites
A cluster adminstration user account is required in order to setup metrics for the cluster.

## Metrics Setup
*  Move into the `openshift-infra` namespace
```
 oc project openshift-infra
```
* A service account is required for the metrics deployer. The deployer will deploy and configure metrics. 
```
oc create -f - <<-EOF
apiVersion: v1
kind: ServiceAccount
metadata:
  name: metrics-deployer
  labels:
    openshift-infra: metrics
secrets:
    - name: metrics-deployer
EOF
```
* The metrics deployer must have the ability to edit the `openshift-infra` project. Allow the service account created
 above to create edit the project as follows.
```
oadm policy add-role-to-user edit system:serviceaccount:openshift-infra:metrics-deployer
```
* The Heapster service account requires access to the /stats endpoint of each node. In order to permit this, the Heapster
service accounts requires the `cluster-reader` role.
```
oadm policy add-cluster-role-to-user cluster-reader system:serviceaccount:openshift-infra:heapster
```
* Even though a self-signed certificate will be used, a secret is still required in order for certificates to be generated.
```
 oc secrets new metrics-deployer nothing=/dev/null
```
* The deployer template is generated during installation. Copy the template into a local directory and modify it as needed. 
Alternatively, the environment variables can be passed into when processing the template. This will be done. The template location
is listed below.
```
/usr/share/ansible/openshift-ansible/roles/openshift_examples/files/examples/v1.1/infrastructure-templates/enterprise/metrics-deployer.yaml
```
* `HAWKULAR_METRICS_HOSTNAME` is the only required environment variable (required in the template). As this will metrics deployment 
is not using persistent storage, `USE_PERSISTENT_STORAGE` is set to false when processing the template, as shown below. Be sure to
set the hostname to a resolvable hostname. It can simply be a hostname similar to one used for an application.

```
oc process -f metrics.yaml -v \
HAWKULAR_METRICS_HOSTNAME=hawkular-metrics.cloudapps.rhc-ose.labs.redhat.com\
,USE_PERSISTENT_STORAGE=false \
| oc create -f -
```
Following creation of the deployer pod, something similar to the following should be shown.
```
pod "metrics-deployer-xrg9w" created
```
Once the deployer pod has completed running, it will launch additional pods that will be response for dispalying and processing metrics.
The following shows an example of the pods that will start following completion of the deployer.
```
NAME                         READY     STATUS      RESTARTS   AGE
hawkular-cassandra-1-g4k1d   0/1       Pending     0          33s
hawkular-metrics-v4jlm       0/1       Pending     0          35s
heapster-fz5w8               1/1       Running     0          35s
metrics-deployer-faliw       0/1       Completed   0          2m
```
* When the deployer is complete and cassandra, hawkular, and cassandra-cluster pods are running, edit the master config file to include
 the URL for the Hawkular metrics that was used for the `HAWKULAR_METRICS_HOSTNAME` in the step above.

```
/etc/origin/master-config.yaml 

assetConfig:
  ...
  metricsPublicURL: "https://hawkular-metrics.cloudapps.rhc-ose.labs.redhat.com/hawkular/metrics"
  ...
```
* Restart the OpenShift master in order for the chnage to take place.

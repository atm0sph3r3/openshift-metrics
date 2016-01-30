## Introduction
This is an implementation guide for enable cluster metrics. For demonstration purposes, this guide uses ephemeral
storage for metrics. It also uses a self-signed certificate for Hawkular metrics. Neither of these are required and
 would not be used in a production environment.

## Prerequisites
A cluster adminstration user is required in order to setup metrics for the cluster.

## Metrics Setup
1. Move into the "openshift-infra" namespace
    oc project openshift-infra
2. A service account is required for the metrics deployer. The deployer will deploy and configure metrics. 
    oc create -f - <<EOF
    apiVersion: v1
    kind: ServiceAccount
    metadata:
      name: metrics-deployer
    secrets:
    - name: metrics-deployer
    EOF
3. The metrics deployer must have the ability to edit the "openshift-infra" project. Allow the service account created
 above to create edit the project as follows.
    oadm policy add-role-to-user edit system:serviceaccount:openshift-infra:metrics-deployer
4. The Heapster service account requires access to the /stats endpoint of each node. In order to permit this, the Heapster
service accounts requires the "cluster-reader" role.
    oadm policy add-cluster-role-to-user cluster-reader system:serviceaccount:openshift-infra:heapster
5. Even though a self-signed certificate will be used, a secret is still required in order for certificates to be generated.
     oc secrets new metrics-deployer nothing=/dev/null
6. The deployer template is generated during installation. Copy the template into a local directory and modify it as needed. 
Alternatively, the environment variables can be passed into when processing the template. This will be done. The template location
is listed below.
    /usr/share/ansible/openshift-ansible/roles/openshift_examples/files/examples/v1.1/infrastructure-templates/enterprise/metrics-deployer.yaml
7. "HAWKULAR_METRICS_HOSTNAME" is the only required environment variable (required in the template). As this will metrics deployment 
is not using persistent storage, "USE_PERSISTENT_STORAGE" is set to false when processing the template, as shown below. Be sure to
set the hostname to a resolvable hostname. It can simply be a hostname similar to one used for an application.
    oc process -f metrics.yaml -v \
      HAWKULAR_METRICS_HOSTNAME=hawkular-metrics.cloudapps.rhc-ose.labs.redhat.com/,USE_PERSISTENT_STORAGE=false \
      | oc create -f -
8. When the deployer is complete and cassandra, hawkular, and cassandra-cluster pods are running, edit the master config file to include
 the URL for the Hawkular metrics that was used for the "HAWKULAR_METRICS_HOSTNAME" in the step above.
    /etc/origin/master-config.yaml 

    assetConfig:
    ...
    metricsPublicURL: "https://hawkular-metrics.cloudapps.rhc-ose.labs.redhat.com/hawkular/metrics"
9. Restart the OpenShift master in order for the chnage to take place.

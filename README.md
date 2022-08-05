# Leveraging the RHACM Observability to support OCP3 to OCP4.x migration
by Luiz Bernardo Levenhagen

Since its release back in 2015, OpenShift Container Platform 3 gained a lot of traction in the market, with more than more than 1,000 organizations across industries and around the world making some use of it.
With OpenShift Container Platform 3, administrators individually deployed Red Hat Enterprise Linux (RHEL) hosts, and then installed OpenShift Container Platform on top of these hosts to form a Kubernetes cluster. Administrators were responsible for properly configuring these hosts and performing updates.

During the Red Hat Summit in 2019, OpenShift Container Platform 4 was introduced and presented a significant change in the way that OpenShift Container Platform clusters were managed and deployed. OpenShift Container Platform 4 included new technologies and functionality, such as Operators, machine sets, and the immutable operating system Red Hat Enterprise Linux CoreOS (RHCOS), which are core to the operation of the cluster. This technology shift enabled clusters to self-manage some functions previously performed by administrators. Key takeaways were platform stability and consistency, and simplified installation and scaling.

Despistes the ease of setup and day-2 configurations from OCP4, upgrading from 3.x to 4.x is not available and users must spin up a new installation of OpenShift Container Platform 4. Hence, migrating workloads from OpenShift 3 to OpenShift 4 requires some planning and possibly additional tooling, depending on the case. Ideally, organizations could redeploy applications to the new environment making use of their CI/CD pipeline and subsequently copying any persistent volume data. If the latter is not an option, Red Hat also provides a solution called Migration Toolkit for Containers, which we’ll talk more about in a moment. 
Regardless, most organizations will follow a similar migration journey, as below:

<img src="https://github.com/levenhagen/blog_rhacm_observability_migration_OCP3-4/blob/main/migration-journey-OCP3-4.png" width="1000">

## Migration considerations
When it comes to migration concerns, cluster administrators will have to analyze storage, networking, authentication and a few other components, throughout all the steps of this journey defined above, in order to successfully migrate. Next section will explain how Red Hat Advanced Cluster Management can help. We’ll focus on the **Setup and Certification** and make sure we have a nice environment setup to certify the migration process.

### Environment Specifications
As mentioned previously, we’re going to make use of an OpenShift Cluster with RHACM installed, and having the source and target clusters as managed clusters. The environment used in our example has the following specifications:
- Red Hat OpenShift Container Platform 4.11
  - 3 Control Plane Nodes (vCPU 4 ,RAM 16GB)
  - 3 Compute Nodes (CPU 4,RAM 16GB)
  - At least  one ```Storageclass``` defined
- Red Hat Advanced Cluster Management 2.5.1
- S3-compatible Object Store

## Installing the Multicluster Observability add-on
When installing the RHACM operator, you will get some observability right away - you get the Search capability and it provides an overview page that shows some dials and details about managed components. This includes an add-on (search-collector) that runs on the managed cluster and it collects kubernetes resources (CRDs, secrets, kinds, pods, storage, etc, just all the 'raw' resources that are in a cluster) and allows the user to search through these things across-clusters.  This is enabled by default, out of the box.

Multicluster Observability, which we mean in this case specifically the platform metrics and alerts, is not turned on by default on the ACM hub. It requires the user to configure an objectStorage and have that ready for Thanos to store all cluster platform metrics into.

Hence, the bullet point here that we are commenting on, this is included with RHACM but still must be turned on first, and that’s what we’re going to do now.

On your ACM Hub Cluster, run this to create a project:
```
$ oc create namespace open-cluster-management-observability
```

After successfully creating the namespace, you’ll need your pull-secret. You can get it from your account in cloud.redhat.com or extract it from the openshift-config namespace as below:

```
DOCKER_CONFIG_JSON=`oc extract secret/pull-secret -n openshift-config --to=-`
```

Copy the pull-secret from the openshift-config namespace into the open-cluster-management-observability namespace. Run the following command:

```
oc create secret generic multiclusterhub-operator-pull-secret -n open-cluster-management-observability 
--from-literal=.dockerconfigjson="$DOCKER_CONFIG_JSON" --type=kubernetes.io/dockerconfigjson
```

With this, observability components in this namespace have credentials that are used for accessing needed images in the registry. Now it’s time to generate our S3-compatible ObjectStore. In this example, like stated before, we’ll be using AWS S3. In the RHACM Web Console, click on the “+” sign on the top bar, and add this to create the secret which Thanos will consume as an S3 resource:

```
apiVersion: v1
kind: Secret
metadata:
  name: thanos-object-storage
  namespace: open-cluster-management-observability
type: Opaque
stringData:
  thanos.yaml: |
    type: s3
    config:
      bucket: YOUR_S3_BUCKET
      endpoint: s3.amazonaws.com
      insecure: false
      access_key: YOUR_ACCESS_KEY
      secret_key: YOUR_SECRET_KEY
```

We’re now ready to finally enable the MultiCluster Obersavility add-on. On the Installed Operators tab, click on the RHACM operator, navigate to MultiClusterObservability and hit **Create instance**:

<img src="https://github.com/levenhagen/blog_rhacm_observability_migration_OCP3-4/blob/main/rhacm-operator-page.png" width="1000">

If everything went correctly, you will be able to see this Grafana option in the RHACM Overview page:

<img src="https://github.com/levenhagen/blog_rhacm_observability_migration_OCP3-4/blob/main/rhacm-overview-page.png" width="1000">

## Import Source and Target Clusters

… to be continued ...

## Building custom dashboards to visualize high-level indicators of migration health

… to be continued ...

## Further recommended practices and conclusion

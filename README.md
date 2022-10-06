# Leveraging the RHACM Observability to support OCP3 to OCP4.x migration
by Luiz Bernardo Levenhagen

Since its release back in 2015, OpenShift Container Platform 3 gained a lot of traction in the market, with more than 1,000 organizations across industries and around the world using it.
With OpenShift Container Platform 3, administrators individually deployed Red Hat Enterprise Linux (RHEL) hosts, and then installed OpenShift Container Platform on top of these hosts to form a Kubernetes cluster. Administrators were responsible for properly configuring these hosts and performing updates. The point: cluster management was a challenge and configuration drift a constant headache.

During Red Hat Summit 2019, OpenShift Container Platform 4 was introduced and presented a significant change in the way the OpenShift Container Platform clusters were deployed and managed. OpenShift Container Platform 4 included new technologies and functionality, such as Operators, machine sets, and the immutable operating system Red Hat Enterprise Linux CoreOS (RHCOS), which are core to the operation of the cluster. This technology shift enabled clusters to self-manage - even self-heal - some functions which previously were only performed by administrators. Key takeaways were platform stability and consistency, and simplified installation and scaling.

Despiste the ease of setup and day-2 configurations from OCP4, upgrading from 3.x to 4.x is not available and users must spin up a new installation of OpenShift Container Platform 4. Hence, migrating workloads from OpenShift 3 to OpenShift 4 requires some planning and possibly additional tooling, depending on the situation. Ideally, organizations could redeploy applications to the new environment making use of their CI/CD pipeline and subsequently copying any persistent volume data. If the latter is not an option, Red Hat also provides a solution called Migration Toolkit for Containers, which we’ll talk more about in a moment. 
Regardless, most organizations will follow a similar migration journey, as below:

<img src="https://github.com/levenhagen/blog_rhacm_observability_migration_OCP3-4/blob/main/migration-journey-OCP3-4.png" width="1000">

## Migration considerations
When it comes to migration concerns, cluster administrators will have to analyze storage, networking, authentication and a few other components, throughout all the steps of this journey defined above, in order to successfully migrate. In the next section, I will explain how Red Hat Advanced Cluster Management can help. We’ll focus on the **Setup and Certification** and make sure we have a nice environment setup to certify the migration process.

### Environment Specifications
As mentioned previously, we’re going to make use of an OpenShift Cluster with RHACM installed, and having the source and target clusters as managed clusters. The environment used in our example has the following specifications:
- Red Hat OpenShift Container Platform 4.11 (or later)
  - 3 Control Plane Nodes (vCPU 4, RAM 16GB)
  - 3 Compute Nodes (vCPU 4, RAM 16GB)
  - At least  one ```Storageclass``` defined
- Red Hat Advanced Cluster Management 2.5.1 (or later)
- S3-API compatible Object Storage

## Installing the Multicluster Observability add-on
When installing the RHACM operator, you will get some observability right away - you get the Search capability and it provides an Overview page that shows some guages and details about the managed clusters. This includes an add-on (search-collector) that runs on the managed cluster which collects kubernetes resources (CRDs, secrets, kinds, pods, storage, etc, just all the 'raw' resources that are in a cluster) and allows the user to search through these things across-all-the-clusters.  This is enabled by default, out of the box.

Multicluster Observability, which we mean in this case specifically the collection and visualization of OCP + Kubernetes platform metrics and alerts, is not turned on by default on the ACM hub. It requires the user to configure an objectStorage and have that ready for the hub Thanos to store all cluster platform metrics into.

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
oc create secret generic multiclusterhub-operator-pull-secret -n open-cluster-management-observability --from-literal=.dockerconfigjson="$DOCKER_CONFIG_JSON" --type=kubernetes.io/dockerconfigjson 
```

With this, observability components in this namespace have credentials that are used for accessing needed images in the registry. Now it’s time to configure a connection for our S3-compatible ObjectStore. In this example, like stated before, we’ll be using AWS S3. In the RHACM Web Console, click on the “+” sign on the top bar, and add this to create the secret which Thanos will consume as an S3 resource:

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

If everything goes smoothly, you will be able to see this *Grafana* link in the RHACM Overview page:

<img src="https://github.com/levenhagen/blog_rhacm_observability_migration_OCP3-4/blob/main/rhacm-overview-page.png" width="1000">

## Import Source and Target Clusters

One of the greatest values that RHACM brings into this migration process is the ability of managing both source and target clusters in various ways. The first step is really a central inventory of clusters and being able to display all of them in the Clusters view and making sure they are connected (with the ready✅ status) to the hub cluster. In our example below, the target cluster will be a OCP v4.10 cluster called **aws-management-1** and source cluster will be a OCP v3.11 cluster named **ocp3-11**.

<img src="https://github.com/levenhagen/blog_rhacm_observability_migration_OCP3-4/blob/main/ocp3-11-and-OCP4-clusters-view.png" width="1000">

**Note:** For importing a cluster that was not created by Red Hat OpenShift Container Platform, you need a **multiclusterhub.spec.imagePullSecret** defined. Check your MultiClusterHub CR from your RHACM installation to check if this field is set or follow [these steps](https://access.redhat.com/documentation/en-us/red_hat_advanced_cluster_management_for_kubernetes/2.6/html/install/installing#custom-image-pull-secret) for steps on how to define the pull-secret.

Assuming you already have your OpenShift clusters 3 up and running, in this same Clusters view, click on **Import cluster** to import the source cluster. You’ll be prompted to give it a name, a clusterSet (optional for RBAC scenarios) which you can leave it empty as it’ll not be useful for now, and the **Import mode**: choose to run commands manually via oc/CLI for your OpenShift 3.

<img src="https://github.com/levenhagen/blog_rhacm_observability_migration_OCP3-4/blob/main/rhacm-import-mode-options.png" width="1000">

Select what’s most convenient for you and in a few minutes you cluster will be fully imported in the **Clusters view**. As you can see, it's a very straight-forward process. You can follow these same steps for importing source and target clusters.
In the case that you don’t have a fully setup OCP4 cluster up and running, RHACM can also help spin up a new one to migrate your workloads from OCP3. Try hitting **Create cluster** on that same **Clusters view** and RHACM will display a bunch of options to guide you create your own OCP4 cluster interactively and very easily. Keep in mind that it’ll require credentials to the infrastructure provider for this automated installation. You can set up these credentials in the **Credentials view** on the left side bar. Also, see [here](https://access.redhat.com/documentation/en-us/red_hat_advanced_cluster_management_for_kubernetes/2.6/html/multicluster_engine/multicluster_engine_overview#creating-a-cluster) the list of supported infrastructure providers for this capability.


## Building custom dashboards to visualize high-level indicators of migration health

…

_This section will introduce how to create customized dashboards to check for overall clusters health and also check for essential components for a OCP3->4 migration.
It’ll also provide a json file that users can import via configmap to the observability component, and they’ll be able to have an example dashboard and go from there_
<<Grafana-Dashboard.json and ScreenShot>>

<img src="https://github.com/levenhagen/blog_rhacm_observability_migration_OCP3-4/blob/main/ocp311.gif" width="1000">
… 


## Further recommended practices and conclusion
Regardless if you’re using CI/CD to move your applications artifacts, and your workloads are just a click away from being placed in the target cluster or if you’ll need a bunch of the other tooling to successfully migrate, the goal here was to introduce RHACM and its flexible Observability component, where it can definitely help you certify that your environments are compatible with your migration strategy. 

The Grafana dashboard above will basically serve as an entrypoint for using customized dashboards for better visualization, but the sky is the limit for adjusting these visualizations to better serve your needs. Check [RHACM docs](https://access.redhat.com/documentation/en-us/red_hat_advanced_cluster_management_for_kubernetes/2.5/html/observability/observing-environments-intro) with the possibility of creating custom metrics, predefined rules and times of data collection for better resource consumption and configurable search. More than just providing monitoring for K8s resources, RHACM teams up with this Thanos + Prometheus + Grafana setup to give you a nice way to be alerted of misused and/or failed components and help you with root cause analysis, if needed. 

Check [MTC Docs](https://docs.openshift.com/container-platform/4.10/migration_toolkit_for_containers/about-mtc.html) and our [Migration webpage](https://www.redhat.com/en/technologies/cloud-computing/openshift/migrating) for more insights and requirements regarding networking, storage and other computing resources for a successful migration. Red Hat Training also provides a course to get your teams sharp on migration from OCP3 to OCP4 - [DO328](https://www.redhat.com/en/services/training/do326-red-hat-openshift-migration-lab). Reach out to the Sales team if interested.

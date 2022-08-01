# Leveraging the RHACM Observability to support OCP3 to OCP4.x migration
by Luiz Bernardo Levenhagen

Since its release back in 2015, OpenShift Container Platform 3 gained a lot of traction in the market, with more than more than 1,000 organizations across industries and around the world making some use of it.
With OpenShift Container Platform 3, administrators individually deployed Red Hat Enterprise Linux (RHEL) hosts, and then installed OpenShift Container Platform on top of these hosts to form a Kubernetes cluster. Administrators were responsible for properly configuring these hosts and performing updates.

During the Red Hat Summit in 2019, OpenShift Container Platform 4 was introduced and presented a significant change in the way that OpenShift Container Platform clusters were managed and deployed. OpenShift Container Platform 4 included new technologies and functionality, such as Operators, machine sets, and the immutable operating system Red Hat Enterprise Linux CoreOS (RHCOS), which are core to the operation of the cluster. This technology shift enabled clusters to self-manage some functions previously performed by administrators. Key takeaways were platform stability and consistency, and simplified installation and scaling.

Despistes the ease of setup and day-2 configurations from OCP4, upgrading from 3.x to 4.x is not available and users must spin up a new installation of OpenShift Container Platform 4. Hence, migrating workloads from OpenShift 3 to OpenShift 4 requires some planning and possibly additional tooling, depending on the case. Ideally, organizations could redeploy applications to the new environment making use of their CI/CD pipeline and subsequently copying any persistent volume data. If the latter is not an option, Red Hat also provides a solution called Migration Toolkit for Containers, which we’ll talk more about in a moment. 
Regardless, most organizations will follow a similar migration journey, as below:
<img src="https://github.com/levenhagen/blog_rhacm_observability_migration_OCP3-4/blob/main/migration-journey-OCP3-4.png" width="1000">

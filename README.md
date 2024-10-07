# Fathom AI

Hello Fathom AI Team!

Thank you so much for taking the time to interview this week.  I have completed the assigned work provided to me, and have made it available for viewing here.

Below is a summation of the work I have done, as well as some thoughts/questions around the implementation of the service moving forward.

## Helm Chart

The Helm Chart here is a simple rendition of a basic helm chart, which can be easily created using the `helm create <name>` command, available in Helm 3.

There are some changes made to the base chart, however:

#### 1. The use of a `StatefulSet` in lue of a `Deployment`-type object

Because the underlying service, known here as `coolkit`, utilizes a shared distributed in-memory cache, it makes sense to utilize a `StatefulSet`, rather than a `Deployment` object, in order to properly handle this use case.

The reason lies in the `service-headless.yaml` file, which can be activated using the `service.additionalHeadlessService.enabled` boolean in the `values.yaml` file.

`StatefulSet`s [allow you to create](https://kubernetes.io/docs/concepts/workloads/controllers/statefulset/#limitations) attached [headless services](https://kubernetes.io/docs/concepts/services-networking/service/#headless-services).  These Kubernetes services, rather than acting as load balancers, simply return a full list of `A` DNS records to contacting pods, of all pods under its `Selector` labels.

By utilizing this endpoint, the underlying `coolkit` pods can retrieve a full list of other `coolkit` pods in the cluster, and communicate with them directly.

For the purpose of this interview, I have simply added an "assumed" environment variable to the `StatefulSet`, that would allow for this to be passed down to the pods:

```yaml
- name: HEADLESS_SERVICE_FOR_POD_ENDPOINTS
  value: {{ include "coolkit.fullname" . }}-headless:{{ .Values.service.port }}
```

`Deployment` type objects do not allow for this functionality, easily.  Although, as an alternative, if you so choosed, you *could* utilize a client library within `coolkit` to contact the Kubernetes API (via the Watch API) to retrieve a list of pods associated with the created `matchLabels`.  This would most likely be unnecessary in this case, however.

Important to note here, however, that a general load-balancing service *is* created to serve pods that are trying to contact `coolkit` API. This is important, as the `headless` service does not provide any kind of load balancing, in and of itself.  So we have two created services: one that allows for `coolkit` pods to contact one another for the purposes of the shared in-memory cache, and another for *non*-`coolkit` pods in the cluster to contact its API, while being provided actual load balancing.

#### 2. Topology Aware Setup

You will notice the use of several Topology Aware settings in the Helm Chart.

Particulary, the use of `topologySpreadConstraints`, `podAntiAffinity`, and the creation of a Topology Aware mapping in the Service annotation (`service.kubernetes.io/topology-mode: "Auto"`).

Let's start with the **scheduling** side of this setup...

To start, because this service will have several pods associated with the deployment/statefulset, it is advantageous for us to spread these pods across as many nodes and availability zones as possible in Google Cloud.

To do this, we enable a Topology Spread Constraint on the `StatefulSet` to ensure that the pods of the StatefulSet are spread across as many Availability Zones as possible.  Note that we utilize a `ScheduleAnyway` policy with the TSC, as it ensures pods that are not able to meet the `maxSkew` of `1` *will* still schedule.  This is a *best attempt* strategy, based on the availability of space in our nodes, at the time of scaling.  This helps ensure that we will scale, even if availability doesn't allow us to meet this requirement.

The `podAntiAffinity`, then, takes this a step further: we also tell the scheduler to spread out our pods, again via a *best attempt* strategy, to as many *distinct* nodes as possible. This helps to ensure the high availability of our service. So, in the case we lose a node, or potentially are utilizing a large amount of Spot-type instance, we should have a very minimal disruption to the service.

Finally, let's discuss the **networking** side of this setup...

The final piece of this puzzle is the `service.kubernetes.io/topology-mode: "Auto"` annotation, which has been added to our *load balancing* service for `coolkit`.

One major cost contributor of networking costs I've seen at several companies, especially on AWS (I have limited experience with Google Cloud), is the cost of *cross-availability-zone* traffic between pods.

Utilizing [Topology Aware Routing](https://kubernetes.io/docs/concepts/services-networking/topology-aware-routing/), you can *significantly* reduce the cost of cross-AZ traffic in a cluster.  This setup actually utilizes a "hint"-based setup, whereby, when an application's pod hits the *loadbalancer* service of `coolkit`, the K8s DNS provider will actually provide a list of pods in that availability zone that the pod can contact. This will not only lower costs associated with cross-AZ traffic, but it should also reduce latency related to calls to the `coolkit` service, as we are keeping traffic to the service as efficient as possible.

Note this works for the `coolkit` service, as the Topology Aware Routing setup requires ***at least*** 9 endpoints (pods) in order for this to work properly.  Because the minimum count of pods of this service is *10*, we can ensure that the TAR setup will work correctly.

#### 3. ScaledObject

Because this service (`coolkit`) utilizes Keda for scaling, there is a `so.yaml` file representing a Keda `ScaledObject` entity.  This is created by default, with the `autoscaling.enabled` value set to `true`.

The end-user can also customize the settings of the ScaledObject.

For the purposes of this example, I have some "basic" values, as defined by the [Keda Documentation](https://keda.sh/docs/2.12/scalers/metrics-api/). The polling interval, and cooldown interval, are set based on what I felt were values that makes sense for most services.  These can be updated, based on the service's needs.

In addition, I have provided both a `scaleUp` and a `scaleDown` policy, which defines that the service will scale *up* more rapidly than it scales *down*.

Important to note that the `ScaledObject` is utilized *in lue of* a `HorizontalPodAutoscaler`.

#### 4. Canary Support

A simple, but useful, update to the chart is the creation of a `fathom.ai/track` annotation, that is added to our objects.

Utilizing these tags, we can utilize CD tools, such as Argo or Spinnaker, to track changes between our `track`s in Kubernetes, and the metrics each are providing us.

A [canary deployment](https://semaphoreci.com/blog/what-is-canary-deployment) setup allows us to have `stable`, `baseline`, and `canary` tracks, which we can perform activations on.

`stable` is, of course, our primary `production` deployment code. `baseline` (a copy of `stable`) and `canary`, on the other hand, can be deployed and analyzed to determine if new code we have deployed is acting *as intended*, prior to us moving that code to a `stable` track.

Utilizing Prometheus, Datadog, Google Cloud metrics, etc., we can perform canary analysis against these code changes, make determinations against those changes, and decide to either move code forward to production, or not.

#### 5. Additional Settings

A few other key things to note:

1. The Cloud DB that is utilized by `coolkit` is setup here utilizing simple environment variables, since I am not personally aware of Fathom's setup.  It assumes, in this case, that the password associated with the Google Cloud DB is made available via a Kubernetes secret (`coolkit-secrets`). However, it is important to note that Google Cloud actually provides an [authentication proxy](https://cloud.google.com/sql/docs/mysql/connect-kubernetes-engine) that utilizes *dynamic* credentials for SQL authentication.  This is the *preferred* method of authentication, and would be a simple `env` change to allow for (just remove our DB secret).
2. The `resources` assigned to the `coolkit` pod is based on best practices for Kubernetes resource utilization:  when possible, it is preferred to [*disable* CPU Limits](https://home.robusta.dev/blog/stop-using-cpu-limits#data-fancybox-2), and set memory requests and limits [to be equal](https://home.robusta.dev/blog/kubernetes-memory-limit).
3. A `fathom.ai/deployed-version` annotation has been added to the `StatefulSet` object, in order to track the `tag` of the image set during Helm deployment.  Important to note here that we **use the tag as the version of our app**, which is different than usually utilizing the `Chart.yaml` value of `appVersion`.  `appVersion` requires semantic versioning, and since we are utilizing a `sha` tag for container deployments, it makes sense to utilize that for tracking the version deployed on K8s.
4. The *load balancing* `service` has been updated to make available the observability port (`2345`) to provide access to the `/metrics` endpoint to services which wish to utilize it (such as, say, a Prometheus). This is largely dependent on setup.
5. That same `observability` port is used for Health Checks of the pods in the StatefulSet, utilizing the `/healthz` endpoint, provided by the service.

## Additional Questions

### Deployment, Maintenance, and Observability

#### Deployment

This helm chart is setup to provide a high-availability setup for the `coolkit` service.

It is intentionally a bit "conservative" when it comes to both the scaling up and scaling down of the service, as I am not privy to data around what may or may not occur as this service does scale up or down.

In particular, around `scaleUp`, we obviously want to error on the side of a steady scaling of the service, to ensure that the underlying Cloud SQL DB is not overwhelmed by the increase in pods.  Depending on the Cloud SQL DB, such as the use of Spanner, you may or may not be able to quickly scale up the database itself to handle any changes in load.

#### Maintenance and Observability

In terms of maintenance, this is largely dependent on the service, and the requirements of the team that created and is managing the `coolkit` service.

Some baseline requirements are obvious: we will need to utilize a tool, such as Prometheus, to track metrics associated with `coolkit`'s uptime and scaling behaviors.  This includes ensuring that `coolkit`'s response time, upstream latency, cpu and memory management, and other important metrics are tracked, and alerted on.

In the case of a substantial number of failing pods, we will of course want to have a proper rollback strategy for the service, in the case of a deployment failure/code issue, but also an emergency strategy to handle issues that may be associated with the `coolkit` app.

In addition, while I am not personally familiar with the shared in-memory cache `coolkit` has setup, additional metrics and alerting on that in-memory cache will be vital to ensuring proper cache response rates from the service.

Additional metrics and observability decisions should come from the metrics being provided to the SRE team from the `/metrics` endpoint of the service.

### Multi-Cluster Deployments

Let me just begin by stating that I am not personally familiar with ArgoCD.  Previously, at both GIPHY and at WeWork, we utilized [Spinnaker](https://spinnaker.io/) for our multi-cluster deployments.  I am somewhat aware of how ArgoCD *can* work for multi-cluster deployments (i.e. a single ArgoCD instance running in each cluster, which then provides deployments from that given cluster based on tagging/folder structure in Git), but for now, I want to give a personal overview of how **I** have accomplished this in the past, as a way of explaining how we could do this moving forward.

In Spinnaker, we have the concept of [pipelines](https://spinnaker.io/docs/concepts/pipelines/).  At GIPHY, I setup our Spinnaker service to be able to deploy to, communicate with, and operate each of our distinct Kubernetes clusters.  Currently, we utilize a "Golden-Cluster" strategy, meaning we have one large Kubernetes cluster spread across multiple-AZs in AWS.

We have distinct pipelines for both *dev* and *prod* clusters, so depending on the Kubernetes manifests being deployed, and from where, Spinnaker is smart about knowing *which* cluster it should be deploying to.

**However**, I have actually begun work on a new method of deploying to Kubernetes, which utilizes a stove-piped setup of Kubernetes clusters.  The idea here is pretty simple:  rather than having a *single* Kubernetes cluster for production, we are working towards a world where we have a Kubernetes cluster *in each availability zone* of our Cloud Provider in Production (Note the model below assumes the use of AWS, rather than GCP, but it applies in any cloud). 

![Stovepipe](./images/K8s%20Stove%20Top.png)

Why would we do this?  Well the benefits are pretty obvious:

1. It allows us to ensure that, in the case of a catastrophic failure of a single Kubernetes cluster, we have our code deployed and ready in at least one, if not more, Kubernetes clusters.  Utilizing a shared load-balancer between the clusters, we can simply "deactivate" the bad cluster, and route all traffic to our *active* clusters.
2. It simplifies `canary` deployments, as well.  Imagine having 3-distinct Production Kubernetes cluster.  Potentially, one is marked as the `canary` cluster, and the other two are `primary` clusters.  The `canary` cluster receives, say, 5% of traffic on any given day, while the `primary` clusters split the remaining `95%`.  We can simply deploy our new `canary` code to the `canary` cluster, and begin tracking analytics around the `5%` of traffic the service is receiving there.
3. The need for Topology Aware Routing *goes away*.  By this I mean, because each Kubernetes cluster operates in a distinct Availability Zone, and networking is kept *within* that availability zone.  So cross-AZ costs *go away*.

Because Spinnaker, or ArgoCD (potentially), allows us to target *multiple* clusters, we can very easily within our pipelines dictate where the code should be deployed, and *how* it should be deployed. We can easily have the pipelines dictate *how*, *when* and *where* the code should be deployed out, based on requirements.

For services that need to communicate across clusters, the use of [cilium cluster mesh](https://cilium.io/use-cases/cluster-mesh/) is one tool available for use, or shared load balancers between clusters could also be used (although this would require traffic to exit the clusters, and return).

## That's all!

Thank you so much for your time, and my apologies for this running a little bit late.

If you have any questions on my implementation, please feel free to reach out to me before our interview time.

All the best,
Bryant Rockoff




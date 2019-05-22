RabbitMQ is a fantastic piece of software when it comes to queuing and multiplexing events. This post will show one way to integrate a RabbitMQ cluster within Kubernetes.

I've struggled quite a bit with this, having never set up RabbitMQ before and, at the time, having almost no experience with Kubernetes as well. When I finally did get it to run however, it ran smoothly and hasn't been anything but stable since.

In the following example configuration, we'll be using the `rabbitmq_peer_discovery_k8s` rabbitmq plugin for clustering. There were several different methods of doing this in the past, but right now, this is the recommended and supported way to do it.

## RabbitMQ Configuration

First, we will go over the configuration for rabbitmq. In this case, we will simply create a custom docker image, which includes most of the configuration we need.

As a base-image, we use the official [RabbitMQ Docker Image](https://hub.docker.com/_/rabbitmq/) and copy in our own `rabbitmq.conf` and `enabled_plugins` files.

```bash
FROM rabbitmq:3.7.8

ADD rabbitmq.conf /etc/rabbitmq
ADD enabled_plugins /etc/rabbitmq
```

The `enabled_plugins` file, which adds the `rabbitmq_peer_discovery_k8s` plugin and the standard management plugin, looks like this:
```bash
[rabbitmq_management,rabbitmq_peer_discovery_k8s].
```

So far, so good. Now it gets a bit more interesting. In `rabbitmq.conf`, we actually configure the peer-discovery mechanism, which will enable our rabbitmq nodes to find each other inside the kubernetes cluster later on:

```bash
## Cluster formation. See http://www.rabbitmq.com/cluster-formation.html to learn more
cluster_formation.peer_discovery_backend  = rabbit_peer_discovery_k8s
cluster_formation.k8s.host = kubernetes.default.svc.cluster.local
cluster_formation.k8s.address_type = hostname
cluster_formation.k8s.service_name = rabbitmq-internal
cluster_formation.k8s.hostname_suffix = .rabbitmq-internal.example.svc.cluster.local
cluster_formation.node_cleanup.interval = 60
cluster_formation.node_cleanup.only_log_warning = true
cluster_partition_handling = autoheal
queue_master_locator=min-masters
```

First, we specify, that we want to use `rabbit_peer_discovery_k8s` as our discovery backend. Then, we define the k8s host (for API access), and that we use `hostname` as an addressing scheme. There is also an `ip` option, but `hostname` is recommended. However, `hostname` can only be used with Stateful Sets in kubernetes, which is what we plan to do anyway.

Then, we specify the name of the `rabbitmq` service in kubernetes and the hostname suffix, which defines that our nodes will be named `rabbit@rabbitmq-0.rabbitmq-internal.example.svc.cluster.local`, with the number counting up (e.g.: rabbitmq-1,2...).

There are some additional settings here, but what we're mostly interested in are the kubernetes configuration options.

Alright, the next step is the kubernetes configuration!

## Kubernetes Configuration

The rabbitmq setup on kubernetes is based on a `StatefulSet` configuration. We also set up `two` services, one for accessing the pods from the outside, and one for peer-discovery between the individual rabbitmq pods.

We'll start off with the less interesting parts - RBAC config and a general configMap, and then move on to the more interesting and important configurations.

First, a general configMap:

```bash
apiVersion: v1
kind: ConfigMap
metadata:
  name: rabbitmq-cfg
  namespace: example
data:
  RABBITMQ_VM_MEMORY_HIGH_WATERMARK: "0.6"
```

In such a configMap, arbitrary environment variables can be set for rabbitmq.

Next up is the RBAC (Role Based Access Control) config, which regulates kubernetes API access:

```bash
kind: Role
apiVersion: rbac.authorization.k8s.io/v1beta1
metadata:
  name: endpoint-reader
  namespace: example
rules:
- apiGroups: [""]
  resources: ["endpoints"]
  verbs: ["get"]
---
kind: RoleBinding
apiVersion: rbac.authorization.k8s.io/v1beta1
metadata:
  name: endpoint-reader
  namespace: example
subjects:
- kind: ServiceAccount
  name: rabbitmq
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: Role
  name: endpoint-reader
apiVersion: v1
---
kind: ServiceAccount
metadata:
  name: rabbitmq 
  namespace: example
```

We define a `rabbitmq` service account, an `endpoint-reader` role, which allows calling `get` on all `endpoints` and then hook them up with a role binding.

Alright, with that out of the way, let's define our services:

```bash
kind: Service
apiVersion: v1
metadata:
  namespace: example 
  name: rabbitmq-internal
  labels:
    app: rabbitmq
spec:
  clusterIP: None
  ports:
    - name: http
      protocol: TCP
      port: 15672
    - name: amqp
      protocol: TCP
      port: 5672
  selector:
    app: rabbitmq  
---
kind: Service
apiVersion: v1
metadata:
  namespace:  example
  name: rabbitmq
  labels:
    app: rabbitmq
    type: LoadBalancer
spec:
  selector:
    app: rabbitmq
  ports:
   - name: rabbitmq-mgmt-port
     protocol: TCP
     port: 15672
     targetPort: 15672
   - name: rabbitmq-amqp-port
     protocol: TCP
     port: 5672
     targetPort: 5672
```

As mentioned above, we need two services in this case. One, called `rabbitmq`, for accessing the rabbitmq pods from outside/other parts of our kubernetes cluster and another one called `rabbitmq-internal`, which is used by the rabbitmq pods to do peer-discovery and to create the rabbitmq-cluster between them.

Now we'll take a look at the core part of the kubernetes configuration - the `StatefulSet`. It is rather long, but don't be alarmed - we'll go over all the relevant parts one by one:

```bash
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: rabbitmq
  namespace: example
spec:
  serviceName: rabbitmq-internal
  revisionHistoryLimit: 3
  updateStrategy:
    type: RollingUpdate
  replicas: 3
  selector: 
    matchLabels:
          app: rabbitmq
  template:
    metadata:
      name: rabbitmq
      labels:
        app: rabbitmq
    spec:
      serviceAccountName: rabbitmq
      terminationGracePeriodSeconds: 10
      containers:        
      - name: rabbitmq
        image: path.to.image/rabbitmq:1234
        lifecycle:
          postStart:
            exec:
              command:
                - /bin/sh
                - -c
                - >
                  until rabbitmqctl --erlang-cookie ${RABBITMQ_ERLANG_COOKIE} node_health_check; do sleep 1; done;
                  rabbitmqctl --erlang-cookie ${RABBITMQ_ERLANG_COOKIE} set_policy ha-all "" '{"ha-mode":"all", "ha-sync-mode": "automatic"}'

        ports:
        - containerPort: 4369
        - containerPort: 5672
        - containerPort: 25672
        - containerPort: 15672
        resources:
          requests:
            memory: "800Mi"
            cpu: "0.4"
          limits:
            memory: "900Mi"
            cpu: "0.6"
        livenessProbe:
          exec:
            command:
              - /bin/sh
              - -c
              - >
                curl -H "Authorization: Basic ${RABBITMQ_BASIC_AUTH}" http://localhost:15672/api/aliveness-test/%2F
          initialDelaySeconds: 15
          periodSeconds: 5
        readinessProbe:
          exec:
            command:
              - /bin/sh
              - -c
              - >
                curl -H "Authorization: Basic ${RABBITMQ_BASIC_AUTH}" http://localhost:15672/api/aliveness-test/%2F
          initialDelaySeconds: 15
          periodSeconds: 5
        envFrom:
         - configMapRef:
             name: rabbitmq-cfg
        env:
          - name: HOSTNAME
            valueFrom:
              fieldRef:
                fieldPath: metadata.name
          - name: NAMESPACE
            valueFrom:
              fieldRef:
                fieldPath: metadata.namespace
          - name: RABBITMQ_USE_LONGNAME
            value: "true"
          - name: RABBITMQ_BASIC_AUTH
			value: 1234
          - name: RABBITMQ_NODENAME
            value: "rabbit@$(HOSTNAME).rabbitmq-internal.$(NAMESPACE).svc.cluster.local"
          - name: K8S_SERVICE_NAME
            value: "rabbitmq-internal"
          - name: RABBITMQ_DEFAULT_USER
			value: usr
          - name: RABBITMQ_DEFAULT_PASS
			value: secret_pass
          - name: RABBITMQ_ERLANG_COOKIE
			value: secret_cookie
          - name: NODE_NAME
            valueFrom:
              fieldRef:
                fieldPath: metadata.name
```

We start out rather basic - defining a `rabbitmq` StatefulSet, configuring update-strategy, service and replicas. Nothing out of the ordinary there, but then we get to the pod template.

The first interesting part is the `postStart` lifecycle hook. This hook executes a script, which first waits until the node is healthy and then sets the `ha-mode`. This is the high-availability mode configuration of rabbitmq. We set it to `all`, which means, that the whole queue will be mirrored to other nodes in the cluster, so we can't lose data, even if one node dies.

This means more synchronization work, which has a performance tax, but there are also other settings for queue mirroring and you should take the one suiting your needs the best.

Next up, we define ports and resource limits, which is not that interesting, but then we define the `livenessProbe` and the `readinessProbe`. There several ways to check, if a rabbitmq instance is healthy, or ready. In this case, we use a very lightweight approach of simply sending something on the `aliveness-test` queue, using a BasicAuth token (which we'll define further down) to do so.

The advantage of this approach is that the impact on the rabbitmq instance is very little, whereas a full status query would take some time to fulfill. The disadvantage is of course, that it's only a very basic test and there could be hidden issues, while sending to our test-queue would still work fine. Everything's a trade-off - this is a simple solution, but there are other, more sophisticated ways to do this I'm sure, which might fit your needs better.

At the end of the `StatefulSet` configuration, we configure it to use our above mentioned configMap and some environment variables. This is important, because some of these are relevant for the peer discovery mechanism.

Let's go through them one by one:

* `HOSTNAME` - we take this from the metadata, it will simply be `rabbitmq-0...n`
* `NAMESPACE` - we also take this from the metadata, we will need it further down for creating the node name
* `RABBITMQ_USE_LONGNAME` - use fully qualified names to identify rmq nodes
* `RABBITMQ_BASIC_AUTH` - the basic auth string used for the livenessProbe
* `RABBITMQ_NODENAME` - this is the node name of the pod based on the peer-discovery config - e.g. `rabbit@rabbitmq-0.rabbitmq-internal.example.svc.cluster.local`
* `K8S_SERVICE_NAME` - the k8s service name
* `RABBITMQ_DEFAULT_USER` - login user
* `RABBITMQ_DEFAULT_PASS` - login pw
* `RABBITMQ_ERLANG_COOKIE` - the erlang cookie
* `NODE_NAME` - the short node name, also generated from the metadata, same as the hostname

Alright, that's the whole config. If we were to run this on a test kubernetes cluster, or locally in [minikube](https://github.com/kubernetes/minikube), we should get multiple rabbitmq instances spawning after one another and finding each other in the cluster. Yay!

## Conclusion

Setting up the RabbitMQ cluster inside kubernetes took me quite a while when I first did it, as it's a bit hard to find up-to-date documentation as well as concrete examples, so I hope this post has helped as a point to start from.

I realize, that this post covered a lot of stuff and that it doesn't go into any details and frankly, I wouldn't say I'm an expert on RabbitMQ or Kubernetes, but for simply setting this up, that's also not necessary. However, if you want to run something like this in production, it would definitely make sense to dive a bit deeper.

#### Resources

* [RabbitMQ](https://www.rabbitmq.com/) 
* [RabbitMQ Kubernetes Cluster Config](https://www.rabbitmq.com/cluster-formation.html#peer-discovery-k8s) 
* [Kubernetes](https://kubernetes.io/)
* [Minikube](https://github.com/kubernetes/minikube)

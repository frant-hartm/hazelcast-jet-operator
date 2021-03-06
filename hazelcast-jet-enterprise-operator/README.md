# Hazelcast Jet Enterprise Operator

Hazelcast Jet Enterprise is packaged with [Operator
Framework](https://github.com/operator-framework), which simplifies
deployment on Kubernetes. This is a step-by-step guide how
to deploy Hazelcast Jet Enterprise cluster together with Management
Center on your Kubernetes cluster.

## Prerequisites

- Kubernetes cluster (with admin rights) and the `kubectl` command
  configured (you may use
  [Minikube](https://kubernetes.io/docs/getting-started-guides/minikube/))

## Kubernetes Deployment steps

Below are the steps to start a Hazelcast Jet cluster using Operator
Framework. Note that the first 3 steps are usually performed only once
for the Kubernetes cluster (by the cluster admin). The step 4 is
performed each time you want to create a new Hazelcast Jet cluster.

Note: You need to clone this repository before following the next steps.

    git clone https://github.com/hazelcast/hazelcast-jet-operator.git
    cd hazelcast-jet-operator/hazelcast-jet-enterprise-operator

### Step 1: Create RBAC

Set the namespace that you are going to run the Operator(`default` in this case):

    sed -i "s|REPLACE_NAMESPACE|default|g" operator-rbac.yaml

If you are on MacOSX run this:

    sed -i "" "s|REPLACE_NAMESPACE|default|g" operator-rbac.yaml

Then run following to configure the Operator permissions.

    kubectl apply -f operator-rbac.yaml

### Step 2: Create CRD (Custom Resource Definition)

To create the Hazelcast Jet resource definition, run the following command.

    kubectl apply -f hazelcastjetenterprisecluster.crd.yaml

### Step 3: Deploy Hazelcast Jet Enterprise Operator

Deploy Hazelcast Jet Operator with the following command.

    kubectl apply -f operator.yaml

### Step 4: Create Secret with Hazelcast License Key

Create a secret with your Hazelcast Jet Enterprise License Key. If you
don't have one, get a trial key from this [link](https://hazelcast.com/hazelcast-enterprise-download/trial/)
.

    $ kubectl create secret generic jet-license-key-secret --from-literal=key=your-license-key
    secret/jet-license-key-secret created

### Step 5: Start Hazelcast Jet Enterprise

Start Hazelcast Jet cluster with the following command.

    kubectl apply -f hazelcast-jet-enterprise.yaml

Your Hazelcast Jet Enterprise cluster (together with Hazelcast Jet
Management Center) should be created.

    $ kubectl get all
    NAME                                                                  READY   STATUS    RESTARTS   AGE
    pod/hazelcast-jet-enterprise-operator-df6bc9876-bmmfm                 1/1     Running   0          9m19s
    pod/jet-enterprise-cluster-hazelcast-jet-enterpri-management-cmq5fd   1/1     Running   0          8m56s
    pod/jet-enterprise-cluster-hazelcast-jet-enterprise-0                 1/1     Running   0          8m56s
    pod/jet-enterprise-cluster-hazelcast-jet-enterprise-1                 1/1     Running   0          8m11s

    NAME                                                                      TYPE           CLUSTER-IP      EXTERNAL-IP                                                              PORT(S)                        AGE
    service/hazelcast-jet-enterprise-operator-metrics                         ClusterIP      172.30.53.146   <none>                                                                   8686/TCP,8383/TCP              8m59s
    service/jet-enterprise-cluster-hazelcast-jet-enterpri-management-center   LoadBalancer   172.30.6.157    a219ca55647854404b7fb0e8b04d37c6-587626856.eu-west-3.elb.amazonaws.com   8081:31465/TCP,443:31788/TCP   8m56s
    service/jet-enterprise-cluster-hazelcast-jet-enterprise                   ClusterIP      None            <none>                                                                   5701/TCP                       8m56s

    NAME                                                                              READY   UP-TO-DATE   AVAILABLE   AGE
    deployment.apps/hazelcast-jet-enterprise-operator                                 1/1     1            1           9m19s
    deployment.apps/jet-enterprise-cluster-hazelcast-jet-enterpri-management-center   1/1     1            1           8m56s

    NAME                                                                                        DESIRED   CURRENT   READY   AGE
    replicaset.apps/hazelcast-jet-enterprise-operator-df6bc9876                                 1         1         1       9m19s
    replicaset.apps/jet-enterprise-cluster-hazelcast-jet-enterpri-management-center-b965b7c7c   1         1         1       8m56s

    NAME                                                               READY   AGE
    statefulset.apps/jet-enterprise-cluster-hazelcast-jet-enterprise   2/2     8m56s

**Note**: In `hazelcast-jet-enterprise.yaml` you can specify all parameters available
in the [Hazelcast Jet Enterprise Helm Chart](https://github.com/hazelcast/charts/tree/master/stable/hazelcast-jet-enterprise).
The `hazelcast-jet-full.yaml` file could be used as a reference to see all
possible configuration options with their default values.

### Step 6: Connect to Hazelcast Jet Management Center

To connect to Hazelcast Jet Management Center, you can use `EXTERNAL-IP`
and open your browser at: `http://<EXTERNAL-IP>:8081` or `http://<EXTERNAL-IP>:443`
.

If you are running on a Minikube environment which does not have
Load Balancer support out of the box and it will show the `EXTERNAL-IP`
as `pending`.

To solve this problem, on a separate terminal window, run the following
command:

    minikube tunnel

It will ask for a password and expose service to the host operating
system.

When asked for service details with the following:
    kubectl get svc

It should show the `EXTERNAL-IP` and Hazelcast Jet Management Center
should be accessible at  `http://<EXTERNAL-IP>:8081`.

### Step 7: Submitting a Job to the Jet Cluster

The Jet CLI shipped with the ZIP distribution could be used to interact 
with the cluster. Since the cluster is inside the Kubernetes environment,
we need to make port forwarding to access the cluster.

To forward port 5701 of the pod to local 5701 port run the following
command:

    kubectl port-forward pods/jet-enterprise-cluster-hazelcast-jet-enterprise-0  5701:5701

After that, we can use `localhost:5701` address to interact with the
cluster. The CLI, by default, tries to connect `localhost:5701`, so
there is no need to explicitly configure it.

Following will submit the example job from the distribution to the Jet
cluster inside Kubernetes:

    cd hazelcast-jet-<version>
    bin/jet submit examples/hello-world.jar

After job submission you can check out its details from Management
Center.

## Configuration

You may want to modify the behavior of the Hazelcast Jet Operator.

### Changing Hazelcast Jet and Hazelcast Jet Management Center version

If you want to modify the Hazelcast or Management Center version, update
`RELATED_IMAGE_HAZELCAST_JET` and `RELATED_IMAGE_MANAGEMENT_CENTER`
environment variables in `operator.yaml`.

### Configuring Hazelcast Cluster

You can check all configuration options in `hazelcast-jet-enterprise-full.yaml`.
Description of all parameters can be found
[here](https://github.com/hazelcast/charts/tree/master/stable/hazelcast-jet-enterprise#configuration).

### Adding Custom JAR file to the Classpath

You can mount any volume which contains your JAR files
to the pods created by operator using `customVolume` configuration.

When the `customVolume` set, the operator will mount provided
volume to the pod on `/data/custom` path.
This path also appended to the classpath of running Java process.

For example, if you have existing host path Persistent Volume and
Persistent Volume Claims like below;

```yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: jet-hostpathpv-volume
  labels:
    type: local
spec:
  storageClassName: manual
  capacity:
    storage: 1Gi
  accessModes:
    - ReadWriteOnce
  hostPath:
    path: "/path/to/my/jars"
---
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: jet-pv-claim
spec:
  storageClassName: manual
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 1Gi
```

You can configure your operator to use it like below in your `hazelcast-jet.yaml`
file.

```yaml
apiVersion: hazelcast.com/v1alpha1
kind: HazelcastJet
metadata:
  name: jet-cluster
spec:
  cluster:
    memberCount: 2
  customVolume:
    persistentVolumeClaim:
      claimName: jet-pv-claim
```

See [Volumes](https://kubernetes.io/docs/concepts/storage/) section on the
Kubernetes documentation for other available options.

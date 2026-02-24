# Extra docs

These docs help with understanding Kubernetes and operating the cluster

## Basic knowledge

* A Kubernetes cluster has a control plane node and worker nodes.
* The worker nodes have pods running in them
  > Pods are the smallest deployable units of computing that you can create and manage in Kubernetes.
  > —Pods, [kubernetes.io](https://kubernetes.io/docs/concepts/workloads/pods/)
  * A pod is a group of one or more containers. The containers share storage and network resources, and specs to run the containers.

## Get a working image

We shall use the Tansy project, which is a FastAPI application that does Monte Carlo Simulations. To confirm the project can run locally on your machine:

```zsh
docker run --rm -p 5100:5100 ghcr.io/herereadthis/tansy:latest
curl localhost:5100/health
```

Once the Kubernetes cluster is operational, run the following commands on the control plane node:

```zsh
kubectl create deployment tansy --image=ghcr.io/herereadthis/tansy:latest --port=5100
kubectl scale deployment tansy --replicas=2 # to run on 2 (multiple) nodes
kubectl expose deployment tansy --port=5100 --target-port=5100 --type=ClusterIP
kubectl get pods
kubectl run curl --image=curlimages/curl --rm -it --restart=Never -- curl http://tansy:5100/health
```

* `kubectl create deployment` docs: [kubernetes.io](https://kubernetes.io/docs/reference/kubectl/generated/kubectl_create/kubectl_create_deployment/)
* `kubectl expose deployment` docs: [kubernetes.io](https://kubernetes.io/docs/reference/kubectl/generated/kubectl_expose/)

### Understanding intent

Why go through the trouble of setting up a cluster, and running containers on the cluster, when you can just run the container on some machine?

* If the container crashes, Kubernetes can restart it. Kubernetes will schedule which node runs the pod.
* If a node goes down, Kubernetes can schedule the pod on another node.
  * If there are no replicas of the service, then the control plane will schedule another node to run the pod and download the image.
  * If there are replicas, then the failover is instant. Requests will just go to the next working node.
* You can run the container across many nodes, for scaling, via `--replicas` flag.
* You define what your desired state is (declarative) and Kubernetes makes it happen.
  * The advantage is your desired state is the source of truth.
  * In an imperative system, the state of the environment may differ from the instructions you send.
* When you create the deployment, the control plane will schedule a node to run the pod and pull the image.
* If you go to the node, you can see which images are running
  ```zsh
  sudo k3s crictl images
  ```

## Making requests

(For now, disregard ingress.)

### Get cluster configuration

For local development, use port forwarding. To get it working, you have to get the configuration of the kubernetes cluster, which includes the cluster's IP address and credentials. Afterwards, you can run `kubectl` on your machine. Run the following commands on a machine that is not in the cluster, but is on the same local network:

```zsh
# Run these commands on your machine, not the cluster
scp admin@<SERVER_IP_ADDRESS>:/etc/rancher/k3s/k3s.yaml ~/.kube/config-apodemus
sed -i '' 's/127.0.0.1/<SERVER_IP_ADDRESS>/g' ~/.kube/config-apodemus
export KUBECONFIG=~/.kube/config-apodemus
# Confirm it works
kubectl get nodes

# Enable port forwarding. Run it on your machine, not the cluster
kubectl port-forward service/tansy 5100:5100
# Confirm it works
curl localhost:5100/health
```

* Docs: "Accessing the Cluster from Outside with kubectl", [docs.k3s.io/cluster-access](https://docs.k3s.io/cluster-access#accessing-the-cluster-from-outside-with-kubectl)
  > Copy /etc/rancher/k3s/k3s.yaml on your machine located outside the cluster as ~/.kube/config. Then replace the value of the server field with the IP or name of your K3s server. kubectl can now manage your K3s cluster.

Once you have the configuration, you can run the previous `kubectl create deployment`, etc. commands on your machine.

`kubectl port-forward` is safe to run on home network; it only binds to localhost.  If you have multiple replicas, `kube-proxy` will split/balance the requests between the replicas. `kube-proxy` runs on every node and handles the network rules that route traffic from your machine, to the clusterIP, to the correct pod.

<!--
* There is a network component called `kube-proxy` that runs on every node.
  * It watches the Kubernetes API for changes, and updates the node's network rules to route traffic to the correct pod.
-->

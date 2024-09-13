# Kind with Metrics
Note
Increase Podman Desktop ram to 4G

Deploy:
`kind create cluster --config kind-mk-config.yaml --name kind-metrics`

Install metrics (required) and dashboard (optional)

```
# Add kubernetes-dashboard repository
helm repo add kubernetes-dashboard https://kubernetes.github.io/dashboard/
# Deploy a Helm Release named "kubernetes-dashboard" using the kubernetes-dashboard chart
helm upgrade --install kubernetes-dashboard kubernetes-dashboard/kubernetes-dashboard --create-namespace --namespace kubernetes-dashboard
```

Set up [Bearer auth](https://github.com/kubernetes/dashboard/blob/master/docs/user/access-control/creating-sample-user.md):

```
k apply -f sa_admin.yaml
k -n kubernetes-dashboard create token admin-user
```

Copy Bearer token


Launch metrics dashboard:
`kubectl -n kubernetes-dashboard port-forward svc/kubernetes-dashboard-kong-proxy 8443:443`

Use browser `https://localhost:8443`

# Deployment

A simple deployment will be made
```
kubectl apply -f deploy.yaml
```
This deployment will have resource requests and limits set.

```
resources:
          limits:
            memory: "128Mi"
            cpu: "500m"
          requests:
            memory: "16Mi"
            cpu: "50m"
```

Now, deploy vpa into the cluster. This repo has a clone of the official VPA repository and you'll want to get the lastest version for testing.

```
git clone https://github.com/kubernetes/autoscaler.git
```

Once cloned, move to this repository and overwrite the *autoscaler* directory. Then navigate to autoscaler/vertical-pod-autoscaler for installation instructions. [shortcut](https://github.com/kubernetes/autoscaler/tree/master/vertical-pod-autoscaler#install-command)

# How vpa works 
Link to [vpa](https://github.com/kubernetes/autoscaler/tree/master/vertical-pod-autoscaler) repo

![vpa](img/vpa-allocate-resources.png)

Components:
- VPA Recommender - monitors metrics to generate recommendations
- VPA Updater - evicts pods
- VPA Admission Controller - will intercept deployment to mutate resource requests and/or limits

Modes:
- Auto - similar to Recreate; continued testing with in-place (no restart) needs to happen as currently pod is still evicted. [issue](https://github.com/kubernetes/autoscaler/issues/5885)
- Recreate - assign resource requests at creation and as well as updates to existing pods by eviction
- Initial - assigns resource requests on Pod creattion and never changes them later
- Off - calculate recommendations only; do not change

> [!Note]
> Once vpa.yaml is deployed the mode is set to Off so you can see recommendations.

Check recommendations:
```
kubectl describe vpa
```
Recommendations:

- Lower Bound - minimum for container
- Target - setting to be used for container resource request
- Uncapped Target - estimation used if no minAllowed/maxAllowed restrictions are in place. We HAVE set those in the vpa.yaml file
- Upper Bound - max recommended resource estimation for container


You'll need two shells running to watch the cluster and deploy a modified vpa.yaml file. In one shell execute:
```
kubectl get po --watch
```

Update the ``vpa.yaml`` line 21, to ``"Auto"`` and redeploy
```
kubectl apply -f vpa.yaml
```
Now watch as vpa will get recommendations, use the updater to evict the pods and admission controler to alter the resource requests.

# How to limit blast radius
Pod Disruption Budget - PDB see [pdb.yaml](./pdb.yaml)
Limits in vpa:
- set maxAllowed/minAllowed
- set controlled resources
- set controlled values
- set container


## Individual Container

>Note
>[CRD](https://github.com/kubernetes/autoscaler/blob/master/vertical-pod-autoscaler/deploy/vpa-v1-crd.yaml) spec for container specific controls

For container control, you specify a single contaier to either:
- turn off vpa; must state ``"off"``
- auto vpa; ``"auto"``

## Other testing

Enable feature gate "InPlacePodVerticalScaling", this is outside of VPA
See also resize cpu and memory [here](kubernetes.io/docs/tasks/configure-pod-container/resize-container-resources/)
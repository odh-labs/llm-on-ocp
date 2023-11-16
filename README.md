# Large Language Model (LLM) on OpenShift

The intent of this repository is to automate the deployment of a large language model and its supporting components on OpenShift.

## Prerequisites

This process assumes you already have an OpenShift 4.12+ running with cluster admin permissions.

You will also need the OpenShift client `oc` and `helm`.

## Step 1 - Install OpenShift GitOps

The goal of this setup process is to be as declarative as possible. With this in mind our first step on the new cluster will be to install the [OpenShift Gitops](https://www.redhat.com/en/technologies/cloud-computing/openshift/gitops) operator and create an instance of [ArgoCD](https://argoproj.github.io/cd/) via the operator, so that all remaining steps can be performed in a GitOps manner.

### Login to the cluster

Before we can run any of the following commands, we need to ensure we are logged in. Run the following to do so, updating the placeholder values:

```bash
oc login --token=<token> \
         --server=<cluster> 
         --insecure-skip-tls-verify=true
```

### Install openshift gitops operator

We can programatically install the openshift gitops operator on the cluster in a declaritive way by creating a `Subscription` kubernetes custom resource to subscribe a given namespace to the Operator.

```bash
cat << EOF | oc apply --filename -
apiVersion: operators.coreos.com/v1alpha1
kind: Subscription
metadata:
  name: openshift-gitops-operator
  namespace: openshift-operators
spec:
  channel: latest
  installPlanApproval: Automatic
  name: openshift-gitops-operator
  source: redhat-operators
  sourceNamespace: openshift-marketplace
  config:
    env:
    - name: ARGOCD_CLUSTER_CONFIG_NAMESPACES
      value: openshift-gitops
EOF
```

### Update openshift gitops instance

Once the operator is installed we can apply our `ArgoCD` custom resource definition. This will be picked up by the operator and an updated argocd instance will be deployed based on the specification we provided.

```bash
cat << EOF | oc apply --filename -
apiVersion: argoproj.io/v1beta1
kind: ArgoCD
metadata:
  name: openshift-gitops
  namespace: openshift-gitops
spec:
  kustomizeBuildOptions: --enable-helm --enable-alpha-plugins
  rbac:
    defaultPolicy: role:admin
    scopes: '[groups]'
  resourceExclusions: |
    - kinds:
        - TaskRun
  server:
    insecure: true
    route:
      enabled: true
      tls:
        insecureEdgeTerminationPolicy: Redirect
        termination: edge
  sso:
    dex:
      openShiftOAuth: true
    provider: dex
EOF
```

Once the argocd instance has started we can access the web interface via the `Route` automatically created by the Operator.

```bash
echo "https://$(oc --namespace openshift-gitops get route openshift-gitops-server --output jsonpath='{.spec.host}')"
```

### Create components with gitops

From here, with openshift gitops running in our cluster, all we need to do is apply the argocd `ApplicationSet` custom resource shown below, which points to a git repository containing our remaining manifests.

This `ApplicationSet` resource will be picked up by ArgoCD and periodically synchronised to our cluster to create an `Application` for each of our required components.

```bash
cat << EOF | oc apply --filename -
apiVersion: argoproj.io/v1alpha1
kind: ApplicationSet
metadata:
  name: llm-on-ocp
  namespace: openshift-gitops
spec:
  goTemplate: true
  goTemplateOptions: ["missingkey=error"]
  generators:
    - git:
        repoURL: https://github.com/odh-labs/llm-on-ocp.git
        revision: main
        directories:
          - path: gitops*
  template:
    metadata:
      name: '{{.path.basename}}'
    spec:
      project: "default"
      source:
        repoURL: https://github.com/odh-labs/llm-on-ocp.git
        targetRevision: main
        path: '{{.path.path}}'
      destination:
        server: https://kubernetes.default.svc
        namespace: llm
      syncPolicy:
        automated:
          prune: true
        syncOptions:
          - CreateNamespace=true
EOF
```




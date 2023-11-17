* Large Language Model (LLM) on OpenShift

The intent of this repository is to automate the deployment of a large language model and its supporting components on OpenShift.

** Prerequisites

This process assumes you already have an OpenShift 4.12+ running with cluster admin permissions.

You will also need the OpenShift client ~oc~.


** Step 1 - Install openshift gitops

The goal of this setup process is to be as declarative as possible. With this in mind our first step on the new cluster will be to install the [[https://www.redhat.com/en/technologies/cloud-computing/openshift/gitops][OpenShift GitOps]] operator and create an instance of [[https://argoproj.github.io/cd][ArgoCD]] via the operator, so that all remaining steps can be performed in a GitOps manner.


*** 1.1 Login to the cluster

Before we can run any of the following commands, we need to ensure we are logged in. Run the following to do so, updating the placeholder values:

#+begin_src bash :results silent
oc login --token=<token> \
         --server=<cluster> 
         --insecure-skip-tls-verify=true
#+end_src


*** 1.2  Install openshift gitops operator

We can programatically install the openshift gitops operator on the cluster in a declaritive way by creating a ~Subscription~ kubernetes custom resource to subscribe a given namespace to the Operator.

#+begin_src bash :results silent
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
#+end_src


*** 1.3 Update openshift gitops instance rbac

Once the operator is installed we can apply our ~ArgoCD~ custom resource definition. This will be picked up by the operator and an updated argocd instance will be deployed based on the specification we provided.

The updates below are primarily to rbac, so that we can login via OpenShift SSO and still see applications targeting our local cluster.

#+begin_src bash :results silent
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
  applicationSet: {}
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
#+end_src

Once the argocd instance has started we can access the web interface via the ~Route~ automatically created by the Operator and click *"Login with OpenShift"*.

#+begin_src bash :results silent
echo "https://$(oc --namespace openshift-gitops get route openshift-gitops-server --output jsonpath='{.spec.host}')"
#+end_src


** Step 2 - Create gitops applicationset

From here, with openshift gitops running in our cluster, all we need to do is apply the argocd ~ApplicationSet~ custom resource shown below, which points to our git repository containing our remaining manifests.

This ~ApplicationSet~ resource will be picked up by ArgoCD and periodically synchronised to our cluster to create an ~Application~ for each of our required components.

#+begin_src bash :results silent
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
#+end_src


** Step 3 - Open our juypter notebook route

Once our gitops ~Applications~ have completed syncing (this may take 5 minutes) we can run snippet below to retrieve our juypter notebook ~Route~ and open this in our browser of choice.

Note the addition of the ~/notebook/rhods-notebooks/jupyter-nb-admin/lab~ trailing url fragment.

#+begin_src bash :results silent
echo "https://$(oc get route --namespace rhods-notebooks jupyter-nb-admin --output jsonpath={.spec.host})/notebook/rhods-notebooks/jupyter-nb-admin/lab"
#+end_src

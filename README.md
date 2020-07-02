# Kubernetes Extensions

- Login to IBM Cloud
- Login to OpenShift
- Create a Custom Resource
- Create an Operator
- Create a Custom Resource and Operator using the Operator SDK
- Create an Application Custom Resource

## Login

```
export CLUSTERNAME=remkohdev-roks-labs-3n-cluster
ibmcloud login 
```

Go to the OpenShift web console
Copy Login command
```
oc login --token=_12AbcD345kIPDIRg2jYpCuZ-g5SM5Im9irY2tol4Q8 --server=https://c100-e.us-south.containers.cloud.ibm.com:30712
```

## Custom Resource (CR)

https://kubernetes.io/docs/tasks/extend-kubernetes/custom-resources/custom-resource-definitions/

Custom Resource Definitions (CRD) were added in Kubernetes v1.7 in June 2017. A CRD defines Custom Resources (CR). A CR is an extension of the Kubernetes API that allows you to store your own API Objects and lets the API Server handle the lifecycle of a CR. On their own, CRs simply let you store and retrieve structured data.

For instance, our Guestbook application consists of an object `Guestbook` with attributes `GuestbookTitle` and `GuestbookSubtitle`, and a Guestbook handles objectes of type `GuestbookMessage` with attributes `Message`, `Sender`. 

You have to ask yourself if it makes sense if your objects are added as a Custom Resource to Kubernetes or not. If your API is a [Declarative API](https://kubernetes.io/docs/concepts/extend-kubernetes/api-extension/custom-resources/#declarative-apis) you can consider adding a CR. 

- Your API has a small number of small objects (resources).
- The objects define configuration of applications or infrastructure.
- The objects are updated relatively infrequently.
- Users often need to read and write the objects.
- main operations on the objects are CRUD (create, read, update and delete).
- Transactions between objects are not required.

It doesn't immediately make sense to store messages by Guestbook users in Kubernetes, but it might make sense to store meta-data about a Guestbook deployment, for instance the title and subtitle of a Guestbook deployment, assigned resources or replicas.

Another benefit of adding a Custom Resource is to view your types in the Kubernetes Dashboard.

If you want to deploy a Guestbook instance as a Kubernetes API object and let the Kubernetes API Server handle the lifecycle events of the Guestbook deployment, you can create a Custom Resource Definition (CRD) for the Guestbook object as follows. That way you can deploy multiple Guestbooks with different titles and let each be managed by Kubernetes.

```
cat <<EOF >>guestbook-crd.yaml
apiVersion: apiextensions.k8s.io/v1
kind: CustomResourceDefinition
metadata:
  name: guestbooks.apps.ibm.com
spec:
  group: apps.ibm.com
  versions:
    - name: v1
      served: true
      storage: true
      schema:
        openAPIV3Schema:
          type: object
          properties:
            spec:
              type: object
              properties:
                guestbookTitle:
                  type: string
                guestbookSubtitle:
                  type: string
  scope: Namespaced
  names:
    plural: guestbooks
    singular: guestbook
    kind: Guestbook
    shortNames:
    - gb
EOF
```

- You can see that the `apiVersion` is part of the `apiextensions.k8s.io/v1` API Group in Kubernetes, which is the API that enables extensions, and the `kind` is set to `CustomResourceDefinition`.
- The `served` flag can disable and enable a version.
- Only 1 version can be flagged as the storage version.
- The `spec.names.kind` is used by your resource manifests and should be CamelCased.

Create the Custom Resource for the Guestbook witht he command,

```
oc create -f guestbook-crd.yaml
```

When run in the terminal,
```
$ oc create -f guestbook-crd.yaml
customresourcedefinition.apiextensions.k8s.io/guestbooks.apps.ibm.com created
```

You have now added a CR to the Kubernetes API, but you have not yet created a deployment of type Guestbook yet.

Create a resource specification of type Guestbook named `my-guestbook`,

```
cat <<EOF >>my-guestbook.yaml
apiVersion: "apps.ibm.com/v1"
kind: Guestbook
metadata:
  name: my-guestbook
spec:
  guestbookTitle: "The Chemical Wedding of Remko"
  guestbookSubtitle: "First Day of Many"
EOF
```

And to create the `my-guestbook` resource, run the command

```
oc create -f my-guestbook.yaml
```

When run in the terminal,
```
$ oc create -f my-guestbook.yaml
guestbook.apps.ibm.com/my-guestbook created
```

If you list all Kubernetes resources, only the default Kubernetes service is listed. To list your Custom Resources, add the extended type to your command.

```
$ oc get all
NAME    TYPE    CLUSTER-IP    EXTERNAL-IP    PORT(S)    AGE
service/kubernetes    ClusterIP    172.21.0.1    <none>    443/TCP    5d14h
service/openshift    ExternalName    <none>    kubernetes.default.svc.cluster.local    <none>    5d14h
service/openshift-apiserver    ClusterIP    172.21.6.8    <none>    =443/TCP    5d14h

$ oc get guestbook 
NAME    AGE
my-guestbook    8m32s
```

To read the details for the `my-guestbook` of type `Guestbook`, describe the instance,

```
$ oc describe guestbook my-guestbook

Name:         my-guestbook
Namespace:    default
Labels:       <none>
Annotations:  <none>
API Version:  apps.ibm.com/v1
Kind:         Guestbook
Metadata:
  Creation Timestamp:  2020-06-30T20:31:36Z
  Generation:          1
  Resource Version:    1081471
  Self Link:           /apis/apps.ibm.com/v1/namespaces/default/guestbooks/my-guestbook
  UID:                 dcbdcafc-999d-4051-9244-0315093357e7
Spec:
  Guestbook Subtitle:  First Day of Many
  Guestbook Title:     The Chemical Wedding of Remko
Events:                <none>
```

Or retrieve the resource information by specifying the type,

```
$ oc get Guestbook -o yaml
apiVersion: v1
items:
- apiVersion: apps.ibm.com/v1
  kind: Guestbook
  metadata:
    creationTimestamp: "2020-07-02T04:41:57Z"
    generation: 1
    name: my-guestbook
    namespace: default
    resourceVersion: "1903244"
    selfLink: /apis/apps.ibm.com/v1/namespaces/default/guestbooks/my-guestbook
    uid: 3f774899-3070-4e00-b74c-a6a14654faeb
  spec:
    guestbookSubtitle: First Day of Many
    guestbookTitle: The Chemical Wedding of Remko
kind: List
metadata:
  resourceVersion: ""
  selfLink: ""
```

In the OpenShift web console, you can browse to Administration > Custom Resource Definitions and find the Guestbook CRD at `/k8s/cluster/customresourcedefinitions/guestbooks.apps.ibm.com`.

![Administration > Custom Resource Definitions](images/openshift-admin-crd.png)

You have now created a new type or Custom Resource (CR) and created an instance of your new type. But just having a new type and a new instance of the type, does not add as much control over the instances yet, we can basically only create and delete a static type with some descriptive meta-data. With a custom controller or `Operator` you can over-write the methods that are triggered at certain lifecycle events.


## Operators

https://kubernetes.io/docs/concepts/extend-kubernetes/operator/

Operators are clients of the Kubernetes API that act as controllers for a Custom Resource. 

To write applications that use the Kubernetes REST API, you can use one of the following supported client libraries:
- [Go](github.com/kubernetes/client-go/),
- [Python](github.com/kubernetes-client/python/),
- [Java](github.com/kubernetes-client/java),
- [CSharp dotnet](github.com/kubernetes-client/csharp),
- [JavaScript](github.com/kubernetes-client/javascript),
- [Haskell](github.com/kubernetes-client/haskell).

In addition, there are many community-maintained [client libraries](https://kubernetes.io/docs/reference/using-api/client-libraries/).


### Ready made operators

At the [OperatorHub.io](https://operatorhub.io/), you find ready to use operators written by the community.

![OperatorHub.io](./images/operatorhub.png)


### Write your own operator

To write your own operator you can use existing tools:
- [KUDO](https://kudo.dev/) (Kubernetes Universal Declarative Operator),
- [kubebuilder](https://book.kubebuilder.io/),
- [Metacontroller](https://metacontroller.app/) using custom WebHooks,
- the [Operator Framework](https://github.com/operator-framework/getting-started).


The Operator SDK provides the following workflow to develop a new Operator:

The following workflow is for a new Go operator:

1. Create a new operator project using the SDK Command Line Interface(CLI)
2. Define new resource APIs by adding Custom Resource Definitions(CRD)
3. Define Controllers to watch and reconcile resources
4. Write the reconciling logic for your Controller using the SDK and controller-runtime APIs
5. Use the SDK CLI to build and generate the operator deployment manifests


Install sdk-operator

https://github.com/remkohdev/ngx-app

Create a new Operator project,

```
$ export DOCKER_USERNAME=<your-docker-username>
$ export DOCKER_PASSWORD=<your-docker-password>

$ export OPERATOR_NAME=my-operator
$ operator-sdk new ngx-app-operator --repo github.com/$DOCKER_USERNAME/ngx-app
$ cd ngx-app-operator
```

Add a new API definition for the Custom Resource under `pkg/apis` and generate the Custom Resource Definition (CRD) and Custom Resource (CR) files under `deploy/crds`.

```
$ operator-sdk add api --api-version=ngx-app.remkoh.dev/v1alpha1 --kind=NgxAppService
```

Add a new controller under `pkg/controller/<kind>`. 

```
$ operator-sdk add controller --api-version=ngx-app.remkoh.dev/v1alpha1 --kind=NgxAppService
```

```
$ operator-sdk build docker.io/$DOCKER_USERNAME/ngx-app-operator
$ docker login docker.io -u $DOCKER_USERNAME -p $DOCKER_PASSWORD

$ docker push docker.io/$DOCKER_USERNAME/ngx-app-operator

$ sed -i "s|REPLACE_IMAGE|docker.io/$DOCKER_USERNAME/ngx-app-operator|g" deploy/operator.yaml
```

Make sure you are connected to the OpenShift cluster (see above),

```
$ oc create sa ngx-app-operator
$ oc create -f deploy/role.yaml
$ oc create -f deploy/role_binding.yaml
$ oc create -f deploy/crds/ngx-app.remkoh.dev_ngxappservices_crd.yaml
$ oc create -f deploy/operator.yaml
$ oc create -f deploy/crds/ngx-app.remkoh.dev_v1alpha1_ngxappservice_cr.yaml
$ oc get pod -l app=example-ngxappservice
$ oc describe ngxappservice example-ngxappservice
```

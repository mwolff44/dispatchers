# dispatchers
dispatcher management for kamailio running inside kubernetes

This tool keeps a `dispatchers.list` file in sync with the Endpoints of any
number of Kubernetes Services.  Each Service is mapped to a single dispatcher
set ID which may be used in kamailio route scripts.

When the `dispatchers.list` file is updated, the tool connects to kamailio over
its binrpc service and tells it to reload the file.

## Options

Command-line options are available to customize and configure the operation of
`dispatchers`:

  * `-kubecfg <string>` - allows specification of a kubecfg, if not running
    inside kubernetes
  * `-o <string>` - specifies the output filename for the dispatcher list.  It
    defaults to `/data/kamailio/dispatcher.list`.
  * `-p <string>` - specifies the port on which kamailio is running its binrpc
    service.  It defaults to `9998`.
  * `-set [namespace:]<service-name>=<index>[:port]`- Specifies a dispatcher set.  This
    may be passed multiple times for multiple dispatcher sets.  Namespace and
    port are optional.  If not specified, namespace is `default` and port is
    `5060`.


## RBAC

When role-based access control (RBAC) is enabled in kubernetes, `dispatchers`
will need to run under a service account with access to the `endpoints` resource
for the namespace(s) in which your dispatcher services exist.

Example RBAC Role for services in the `sip` namespace:

```yaml

kind: ServiceAccount
apiVersion: v1
metadata:
  name: dispatchers
  namespace: sip

--

kind: ClusterRole
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  name: endpoints-reader
rules:
  - apiGroups: [""]
    resources: ["endpoints"]
    verbs: ["get", "watch", "list"]

--

kind: RoleBinding
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  namespace: sip
  name: dispatchers
subjects:
  - kind: ServiceAccount
    name: dispatchers
    apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: ClusterRole
  name: endpoints-reader
```

One `RoleBinding` should be added for each namespace `dispatchers` should have
access to, changing `metadata.namespace` as appropriate.

Once added, make sure that the Pod in which the `dispatchers` container is
running is assigned to the ServiceAccount you created using the
`spec.serviceAccountName` parameter.  For instance:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: my-pod
spec:
  serviceAccountName: dispatchers
  ...
```

You can also bind the namespace-default ServiceAccount to make things easier, if
you have a simple setup:  `system:serviceaccount:<namespace>:default`.


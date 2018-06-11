# Expose VirtualMachines as a Services

Once the VirtualMachine is started, in order to connect to a VirtualMachine,
you can create a `Service` object for a VirtualMachine. Currently, three types
of service are supported: `ClusterIP`, `NodePort` and `LoadBalancer`. The
default type is `ClusterIP`.

> **Note**: Labels on a VirtualMachine are passed through to the pod, so simply
> add your labels for service creation to the VirtualMachine. From there on it works like
> exposing any other k8s resource, by referencing these labels in a service.

## Expose VirtualMachine as a ClusterIP Service

Give a VirtualMachine with the label `special: key`:

```yaml
apiVersion: kubevirt.io/v1alpha1
kind: VirtualMachine
metadata:
  name: vm-ephemeral
  labels:
    special: key
spec:
  domain:
    devices:
      disks:
      - disk:
          bus: virtio
        name: registrydisk
        volumeName: registryvolume
    resources:
      requests:
        memory: 64M
  volumes:
  - name: registryvolume
    registryDisk:
      image: kubevirt/cirros-registry-disk-demo:latest
```

we can expose its SSH port (22) by creating a `ClusterIP` service:

```yaml
apiVersion: v1
kind: Service
metadata:
  name: vmservice
spec:
  ports:
  - port: 27017
    protocol: TCP
    targetPort: 22
  selector:
    special: key
  type: ClusterIP
```

You just need to create this `ClusterIP` service by using `kubectl`:

```bash
$ kubectl create -f vmservice.yaml
```

Alternatively, the VirtualMachine could be exposed using the `virtctl` command:


```bash
$ virtctl expose virtualmachine vm-ephemeral --name vmservice --port 27017 --target-port 22
```

Notes:
* If `--target-port` is not set, it will be take the same value as `--port`
* The cluster IP is usually allocated automatically, but it may also be forced into a value using the `--cluster-ip` flag (assuming value is in the valid range and not taken)

Query the service object:

```bash
$ kubectl get service
NAME        TYPE        CLUSTER-IP     EXTERNAL-IP   PORT(S)     AGE
vmservice   ClusterIP   172.30.3.149   <none>        27017/TCP   2m
```

You can connect to the VirtualMachine by service IP and service port inside the cluster network:

```bash
$ ssh cirros@172.30.3.149 -p 27017
```

## Expose VirtualMachine as a NodePort Service

Expose the SSH port (22) of a VirtualMachine running on KubeVirt by creating a
`NodePort` service:

```yaml
apiVersion: v1
kind: Service
metadata:
  name: nodeport
spec:
  externalTrafficPolicy: Cluster
  ports:
  - name: nodeport
    nodePort: 30000
    port: 27017
    protocol: TCP
    targetPort: 22
  selector:
    special: key
  type: NodePort
```

You just need to create this `NodePort` service by using `kubectl`:

```bash
$ kubectl -f nodeport.yaml
```

Alternatively, the VirtualMachine could be exposed using the `virtctl` command:

```bash
$ virtctl expose virtualmachine vm-ephemeral --name nodeport --type NodePort --port 27017 --target-port 22 --node-port 30000
```

Notes:
* If `--node-port` is not set, its value will be allocated dynamically (in the range above 30000)
* If the `--node-port` value is set, it must be unique across all services

The service can be listed by querying for the service objects:

```bash
$ kubectl get service
NAME           TYPE       CLUSTER-IP      EXTERNAL-IP   PORT(S)           AGE
nodeport       NodePort   172.30.232.73   <none>        27017:30000/TCP   5m
```

Connect to the VirtualMachine by using a node IP and node port outside the
cluster network:

```bash
$ ssh cirros@$NODE_IP -p 30000
```

## Expose VirtualMachine as a LoadBalancer Service

Expose the RDP port (3389) of a VirtualMachine running on KubeVirt by creating
`LoadBalancer` service. Here is an example:

```yaml
apiVersion: v1
kind: Service
metadata:
  name: lbsvc
spec:
  externalTrafficPolicy: Cluster
  ports:
  - port: 27017
    protocol: TCP
    targetPort: 3389
  selector:
    speical: key
  type: LoadBalancer
```

You could create this `LoadBalancer` service by using `kubectl`:

```bash
$ kubectl -f lbsvc.yaml
```

Alternatively, the VirtualMachine could be exposed using the `virtctl` command:

```bash
$ virtctl expose virtualmachine vm-ephemeral --name lbsvc --type LoadBalancer --port 27017 --target-port 3389
```

Note that the external IP of the service could be forced to a value using the `--external-ip` flag (no validation is performed on this value).

The service can be listed by querying for the service objects:

```bash
$ kubectl get svc
NAME      TYPE           CLUSTER-IP       EXTERNAL-IP                   PORT(S)           AGE
lbsvc     LoadBalancer   172.30.27.5      172.29.10.235,172.29.10.235   27017:31829/TCP   5s
```

Use `vinagre` client to connect your VirtualMachine by using the public IP and
port. 

Note that here the external port here (31829) was dynamically allocated.

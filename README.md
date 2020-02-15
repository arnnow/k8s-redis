# Setup redis & redis-sentinel using kustomize
based on [redis](https://github.com/kubernetes/examples/tree/master/staging/storage/redis) but adapted for services/deployment & kustomize  
Using Redis V5 - Dockerfile adapted from [this](https://hub.docker.com/_/redis/)  

### Building you image for minikube
#### Creating the image
We will here create a docker image via a dockerfile and use it directly in minikube without pushing it onto a repository  
the dockerfile is here
```bash
# Set docker env
{22:17}~/k8s-redis:master ✗ ➭ eval $(minikube docker-env)

{22:17}~/k8s-redis/image:master ✗ ➭ docker build -t redis:v5 .
Sending build context to Docker daemon  3.072kB
Step 1/12 : FROM alpine:3.11
...
Successfully built bb7702cc6a2b
Successfully tagged redis:v5
```
Your image should be present in minikube
```bash
{22:23}~/k8s-redis:master ✗ ➭ minikube ssh
                         _             _
            _         _ ( )           ( )
  ___ ___  (_)  ___  (_)| |/')  _   _ | |_      __
/' _ ` _ `\| |/' _ `\| || , <  ( ) ( )| '_`\  /'__`\
| ( ) ( ) || || ( ) || || |\`\ | (_) || |_) )(  ___/
(_) (_) (_)(_)(_) (_)(_)(_) (_)`\___/'(_,__/'`\____)

$ docker image ls | grep redis
redis                                     v5                  bb7702cc6a2b        28 minutes ago      34MB
```


### Turning up an initial master/sentinel pod.

We will use the shared network namespace to bootstrap our Redis cluster.  In particular, the very first sentinel needs to know how to find the master (subsequent sentinels just ask the first sentinel).  Because all containers in a Pod share a network namespace, the sentinel can simply look at ```$(hostname -i):6379```.

Here is the config for the initial master and sentinel pod: [redis-bootstrap_pod.yaml](boostrap/redis-bootstrap_pod.yaml)


Create this master as follows:
```sh
/k8s-redis ➭ kubectl apply -f boostrap/redis-bootstrap_pod.yaml
pod/redis-bootstrap created

```

You should get a running redis-bootstrap pod.
```sh
/k8s-redis ➭ kubectl get pods,rc,svc,deployments,cm,pv,pvc -o wide
NAME                  READY   STATUS    RESTARTS   AGE   IP           NODE       NOMINATED NODE   READINESS GATES
pod/redis-bootstrap   2/2     Running   0          49s   172.17.0.5   minikube   <none>           <none>

```
### Launching the redis stack

We have a pod running with :
* redis as master
* redis-sentinel  

Labels have been set onto the pod
```
metadata:
  name: redis-bootstrap
  labels:
    app: redis
    tier: redis-sentinel
```
Our redis-sentinel services will adopt this pod as its selector is set to :
```
spec:
  selector:
    app: redis
    tier: redis-sentinel
```
So when our redis-sentinel pod will start and query try to connect to sentinel in order to get the master IP, they will reach the running
redis-sentinel.
in our [image/run.sh](image/run.sh)
```
master=$(redis-cli -h ${REDIS_SENTINEL_SERVICE_HOST} -p ${REDIS_SENTINEL_SERVICE_PORT} --csv SENTINEL get-master-addr-by-name mymaster | tr
',' ' ' | cut -d' ' -f1)
```
```REDIS_SENTINEL_SERVICE_HOST``` & ```REDIS_SENTINEL_SERVICE_PORT``` are set by the service redis-sentinel.

Let's apply our config and check :

```sh
k8s-redis/env/dev ➭ kubectl apply -k ./
service/redis-sentinel created
service/redis created
deployment.apps/redis-sentinel created
deployment.apps/redis created

k8s-redis/env/dev ➭ kubectl describe service/redis-sentinel | grep Endpoints
Endpoints:         172.17.0.12:26379,172.17.0.5:26379,172.17.0.6:26379 + 1 more...
```
As we can see the ```pod/redis-bootstrap``` is part of the service now. Query to this "Sentinel" service will reach it. Thus enabling the
other services to be configured.

Here are the logs of one of the new sentinel pods
```sh
# Keyspace
11:X 15 Feb 2020 22:14:32.903 # oO0OoO0OoO0Oo Redis is starting oO0OoO0OoO0Oo
11:X 15 Feb 2020 22:14:32.903 # Redis version=5.0.7, bits=64, commit=00000000, modified=0, pid=11, just started
11:X 15 Feb 2020 22:14:32.903 # Configuration loaded
11:X 15 Feb 2020 22:14:32.905 * Running mode=sentinel, port=26379.
11:X 15 Feb 2020 22:14:32.905 # WARNING: The TCP backlog setting of 511 cannot be enforced because /proc/sys/net/core/somaxconn is set to
the lower value of 128.
11:X 15 Feb 2020 22:14:32.909 # Sentinel ID is ce945ec2abb0bd561f1f365f4303e497c75c082d
11:X 15 Feb 2020 22:14:32.909 # +monitor master mymaster 172.17.0.5 6379 quorum 2
11:X 15 Feb 2020 22:14:33.228 * +sentinel sentinel b12916626795c6f4743c90374229bc58ad085827 172.17.0.5 26379 @ mymaster 172.17.0.5 6379
11:X 15 Feb 2020 22:14:34.960 * +sentinel sentinel fda58015aaa8e32131e55486e00775b64164566a 172.17.0.6 26379 @ mymaster 172.17.0.5 6379
11:X 15 Feb 2020 22:14:35.197 * +sentinel sentinel ec4641d22b3fedb64c2bcf98edaea5357fe1c2e6 172.17.0.7 26379 @ mymaster 172.17.0.5 6379
11:X 15 Feb 2020 22:14:42.914 * +slave slave 172.17.0.8:6379 172.17.0.8 6379 @ mymaster 172.17.0.5 6379
11:X 15 Feb 2020 22:14:42.916 * +slave slave 172.17.0.9:6379 172.17.0.9 6379 @ mymaster 172.17.0.5 6379
```

### Deleting the initial master/sentinel pod
we can now delete the initial ```pod/redis-bootstrap```. Our redis should now be electing a new master from the one deployed by kustomize
```
11:X 15 Feb 2020 22:19:31.214 # +odown master mymaster 172.17.0.5 6379 #quorum 3/2
11:X 15 Feb 2020 22:19:31.214 # +new-epoch 1
11:X 15 Feb 2020 22:19:31.214 # +try-failover master mymaster 172.17.0.5 6379
11:X 15 Feb 2020 22:19:31.217 # +vote-for-leader ce945ec2abb0bd561f1f365f4303e497c75c082d 1
11:X 15 Feb 2020 22:19:31.224 # ec4641d22b3fedb64c2bcf98edaea5357fe1c2e6 voted for ce945ec2abb0bd561f1f365f4303e497c75c082d 1
11:X 15 Feb 2020 22:19:31.224 # fda58015aaa8e32131e55486e00775b64164566a voted for ce945ec2abb0bd561f1f365f4303e497c75c082d 1
11:X 15 Feb 2020 22:19:31.304 # +elected-leader master mymaster 172.17.0.5 6379
11:X 15 Feb 2020 22:19:31.305 # +failover-state-select-slave master mymaster 172.17.0.5 6379
11:X 15 Feb 2020 22:19:31.364 # +selected-slave slave 172.17.0.8:6379 172.17.0.8 6379 @ mymaster 172.17.0.5 6379
11:X 15 Feb 2020 22:19:31.364 * +failover-state-send-slaveof-noone slave 172.17.0.8:6379 172.17.0.8 6379 @ mymaster 172.17.0.5 6379
11:X 15 Feb 2020 22:19:31.465 * +failover-state-wait-promotion slave 172.17.0.8:6379 172.17.0.8 6379 @ mymaster 172.17.0.5 6379
11:X 15 Feb 2020 22:19:32.228 # +promoted-slave slave 172.17.0.8:6379 172.17.0.8 6379 @ mymaster 172.17.0.5 6379
11:X 15 Feb 2020 22:19:32.228 # +failover-state-reconf-slaves master mymaster 172.17.0.5 6379
11:X 15 Feb 2020 22:19:32.301 * +slave-reconf-sent slave 172.17.0.9:6379 172.17.0.9 6379 @ mymaster 172.17.0.5 6379
11:X 15 Feb 2020 22:19:33.228 * +slave-reconf-inprog slave 172.17.0.9:6379 172.17.0.9 6379 @ mymaster 172.17.0.5 6379
11:X 15 Feb 2020 22:19:33.228 * +slave-reconf-done slave 172.17.0.9:6379 172.17.0.9 6379 @ mymaster 172.17.0.5 6379
11:X 15 Feb 2020 22:19:33.303 # -odown master mymaster 172.17.0.5 6379
11:X 15 Feb 2020 22:19:33.304 # +failover-end master mymaster 172.17.0.5 6379
11:X 15 Feb 2020 22:19:33.304 # +switch-master mymaster 172.17.0.5 6379 172.17.0.8 6379
11:X 15 Feb 2020 22:19:33.311 * +slave slave 172.17.0.9:6379 172.17.0.9 6379 @ mymaster 172.17.0.8 6379
11:X 15 Feb 2020 22:19:33.313 * +slave slave 172.17.0.5:6379 172.17.0.5 6379 @ mymaster 172.17.0.8 6379
11:X 15 Feb 2020 22:19:38.329 # +sdown slave 172.17.0.5:6379 172.17.0.5 6379 @ mymaster 172.17.0.8 6379
```
Our new master is 172.17.0.8:6379
And the replication from other redis pod is ok
```
13:S 15 Feb 2020 22:19:32.302 * REPLICAOF 172.17.0.8:6379 enabled (user request from 'id=8 addr=172.17.0.12:38617 fd=12
name=sentinel-ce945ec2-cmd age=290 idle=0 flags=x db=0 sub=0 psub=0 multi=3 qbuf=286 qbuf-free=32482 obl=36 oll=0
omem=0 events=r cmd=exec')
13:S 15 Feb 2020 22:19:32.306 # CONFIG REWRITE executed with success.
13:S 15 Feb 2020 22:19:33.139 * Connecting to MASTER 172.17.0.8:6379
13:S 15 Feb 2020 22:19:33.140 * MASTER <-> REPLICA sync started
13:S 15 Feb 2020 22:19:33.140 * Non blocking connect for SYNC fired the event.
13:S 15 Feb 2020 22:19:33.143 * Master replied to PING, replication can continue...
13:S 15 Feb 2020 22:19:33.144 * Trying a partial resynchronization (request f150c7d7fd305bc299fbc995d2a2f74aa4fcfee9:77793).
13:S 15 Feb 2020 22:19:33.146 * Successful partial resynchronization with master.
13:S 15 Feb 2020 22:19:33.146 # Master replication ID changed to b79d84f890c6871be9f07414b7d7e4b3de3b7065
13:S 15 Feb 2020 22:19:33.146 * MASTER <-> REPLICA sync: Master accepted a Partial Resynchronization.
```

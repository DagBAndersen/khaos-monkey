![khaos-monkey](./images/khaos-monkey-02.png)

# A simple Chaos Monkey for Kubernetes
Built on top of [kube-rs](https://github.com/kube-rs/kube-rs)

* [Why would you need this monkey?](#why-would-you-need-this-monkey)
* [How is this monkey different from similar tools?](#how-is-this-monkey-different-from-similar-tools)
* [3 Modes](#3-modes)
* [Pod Targeting](#pod-targeting)
* [Randomness](#randomness)
* [Running/Testing on local machine](#runningtesting-on-local-machine)
* [Installation](#installation)

# Why would you need this monkey?

Read about Chaos Engineering

# How is this monkey different from similar tools?

I created this tools because the tools i found didnt fit my use-case very well. Other tools like [kube-monkey](https://github.com/asobti/kube-monkey) forces you to add labels to every single resource you want the chaos monkey to target. This tool takes another approach and focuses on having equal rules for all pods in a given namespace.

# 3 Modes

### Percentage of pods killed
The monkey will kill a given percentage of targeted pods. The number is rounded down.

> *Example*: if you run the monkey with  `./khoas-monkey percentage 55` and your `ReplicaSet` has 4 pods the monkey will kill 2 random pods on every attack.

### Fixed number of pods killed
If set to `fixed` they will kill a fixed number of pods each type.

> *Example*: if you run the monkey with  `./khoas-monkey fixed 3` and your `ReplicaSet` has 5 pods the monkey will kill 3 random pods on every attack.

### Fixed number of pods left
If set to `fixed_left` they will kill all pod types until there is a fixed number of pods left.

> *Example*: if you run the monkey with `./khoas-monkey fixed-left 3` and your `ReplicaSet` has 5 pods the monkey will kill pods until there is 3 left. In this case it would kill 2 pods.

# Pod Targeting

## Based on namespaces

The monkey can either target individual pods or whole namespaces. If the option `--target-namespaces` is set to `"namespaceA, namespaceB"` the monkey will target all pods in those two namespaces. This means that the monkey may kill any pod (unless they [opt-out](#Opt-out)) in those namespaces.

> *Example*: If you run the monkey with `--target-namespaces="namespaceA,namespaceB"` it will target all pods in `namespaceA` and `namespaceB`.

## Opt-in

This feature means you can make the monkey target individual pod in any namespace by adding the label `khaos-enabled: "true"` to to the pod. If this label exist on a pod it doesn't matter if it inside in the namespaces specified by `--target-namespaces` or not. 

*Opt-in deployment example*:
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: whatever-deployment
spec:
  template:
    spec:
      metadata:
        labels:
          khaos-enabled: "true"
      containers:
      - name: whatever-container
        image: whatever-image
```

## Opt-out

This feature means you can make the monkey to target on all pods in specific namespaces and choose which pods you want to be excluded in those namespaces. All pods with the label `khaos-enabled: false` will opt-out and will be excluded in the pod targeting by the monkey.

> *Example*: The monkey is targeting `namespaceA` and `podA` exist inside that namespace. If the pod has the label `khaos-enabled: false` it will be ignored by the monkey and not killed - if it does not have that label it will be targeted by the monkey (and eventually be killed). 

*Opt-out Example*:
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: whatever-deployment
spec:
  template:
    spec:
    metadata:
        labels:
          khaos-enabled: "false"
      containers:
      - name: whatever-container
        image: whatever-image
```

## Target Grouping

By default all pods in a `ReplicaSet` is grouped. So if a `Deployment` has `4` replicas the monkey may kill *x* pods of those replicas. 
You can make a custom group by adding a label to the pods - e.g. `khaos-group=my-group`. This would make the monkey treat your custom group the same way it treats a `ReplicaSet`. 

> *Example*: Lets say that deployment `depA` has 2 pods/replicas and deployment `depB` has 1 pod/replicas and all 3 pods/replicas has the label `khaos-group=my-group`. The monkey is set to `./khaos-monkey fixed 2`. In this case the monkey will kill either 2 pods of `depA`' pods or 1 pod from each deployment, since they are treated as being in the same group.

*deployment example*:
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: whatever-deployment
spec:
  replicas: 4
  template:
    spec:
      metadata:
        labels:
          khaos-enabled: "true"
          khaos-group: my-group
      containers:
      - name: whatever-container
        image: whatever-image
```

## Blacklisting namespaces

You can specify namespaces where it is not possible to [opt-in](#opt-in).

> *Example*: If you want the monkey with `--blacklisted-namespace="whatever"` it is not possible for pods to [opt-in](#opt-in) in namespace `"whatever"`.

# Randomness

You can randomize how often the attack happens and how many pods are killed each attack.

You can set `random-kill-count` to `true` if you want the monkey to kill a random amount of pods between 0 and the specified value for that mode.
> *Example*: If the monkey run with `./khaos-monkey --random-kill-count=true percentage 50` then the monkey will kill between `0` and `50` percent of the pods in each `ReplicaSet`.

You can set `random-extra-time-between-chaos` to `5m` if you want you want to add additional random time between each attack.
> *Example*: If the monkey run with `--min-time-between-chaos=1m --random-extra-time-between-chaos=1m` the attacks will happen with a random time interval between 1 and 2 minutes.

# Running/Testing on local machine
You can test the monkey on your local machine before putting it on kubernetes. If you have your kube-config installed in `~/.kube/config` and have `cargo` installed then you can just pull the repo and run `cargo run -- --target-namespaces="my-namespace" fixed 1` in the repo root. If your config and permissions are correct the monkey will starting killing pods in namespace, "my-namespace", on the current `kubectl` context.

# Installation

> Note: The default settings not target any namespaces, so it won't start killing pods until you specify namespaces to target or you make pods opt-in.

### Create the namespace:
```bash
$ kubectl create namespace khaos-monkey
```
### Create the right permission with rbac:

Either by referring to the repo file:
```bash
$ kubectl apply -f https://raw.githubusercontent.com/DagBAndersen/khaos-monkey/main/kube-manifests/rbac.yml
```
or copy-paste and run this in your terminal:
```yaml
cat <<EOF | kubectl apply -f -
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: khaos-monkey-cluster-role
rules:
- apiGroups: ["*"]
  resources: ["pods", "namespaces"]
  verbs: ["list", "delete"]
---
kind: ClusterRoleBinding
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  name: khaos-monkey-cluster-role-binding
subjects:
- kind: ServiceAccount
  name: default
  namespace: khaos-monkey
roleRef:
  kind: ClusterRole
  name: khaos-monkey-cluster-role
  apiGroup: ""
EOF
```

### Deploy the monkey

> Feel free to tune the numbers yourself. Remember that the monkey may kill it self if it exist inside a targeted namespace and does not not [opt-out](#Opt-out). It is possible to run multiple instances of the monkey with different settings. 

Either by referering to the repo file:
```bash
$ kubectl apply -f https://raw.githubusercontent.com/DagBAndersen/khaos-monkey/main/kube-manifests/monkey-deployment.yml
```
or copy-paste and run this in your terminal:

```yaml
cat <<EOF | kubectl apply -f -
apiVersion: apps/v1
kind: Deployment
metadata:
  namespace: khaos-monkey
  name: khaos-monkey
spec:
  replicas: 1
  selector:
    matchLabels:
      app: khaos-monkey
  strategy:
    type: Recreate
  template:
    metadata:
      labels:
        khaos-enabled: "false"
        app: khaos-monkey
    spec:
      containers:
      - name: khaos-monkey
        image: dagandersen/khaos-monkey:v0.1.0
        args: ["fixed", "1" ]
EOF
```

### Verify that the Khaos Monkey work
Run the following command to verify that the monkey works as expected. 
```bash
$ kubectl wait -A --for=condition=ready pod -l "app=khaos-monkey" && kubectl logs -l app=khaos-monkey -n khaos-monkey --follow=true --tail=100
```

The command will print something like this

```
target_namespaces from args/env: {}
blacklisted_namespaces from args/env: {"kube-system", "kube-node-lease", "kube-public"}
Namespaces found in cluster: {"kube-system", "default", "kube-node-lease", "khaos-monkey", "kube-public", "local-path-storage"}

Monkey will target: {}

###################
### Chaos Beginning

Killed no pods

### Chaos over
Time until next Chaos: 1m 42s
###################

...
```

# All CLI Options

```
khaos-monkey 0.1.0

USAGE:
  khaos-monkey [OPTIONS] <SUBCOMMAND>

FLAGS:
  -h, --help       Prints help information
  -V, --version    Prints version information

OPTIONS:
  --attacks-per-interval <attacks-per-interval>
      Number of pod-types that can be deleted at a time. No limit if value is -1. Example: if set to "2" it may
      attack two ReplicaSets
      [env: ATTACKS_PER_INTERVAL=]  [default: 1]
  
  --blacklisted-namespaces <blacklisted-namespaces>
      namespaces you want the monkey to ignore. Pods running in these namespaces can't be target
      [env: BLACKLISTED_NAMESPACES=]  [default: kube-system, kube-public, kube-node-lease]
  
  --min-time-between-chaos <min-time-between-chaos>
      Minimum time between chaos attacks
      [env: MIN_TIME_BETWEEN_CHAOS=]  [default: 1m]

  --random-extra-time-between-chaos <random-extra-time-between-chaos>
      This specifies a random time interval that will be added to `min-time-between-chaos` each attack. Example:
      If both options are sat to `1m` the attacks will happen with a random time interval between 1 and 2 minutes
      [env: RANDOM_EXTRA_TIME_BETWEEN_CHAOS=]  [default: 1m]

  --random-kill-count <random-kill-count>
      If "true" a number between 0 and 1 is multiplied with number of pods to kill
      [env: RANDOM_KILL_COUNT=]  [default: false]

  --target-namespaces <target-namespaces>
      namespaces you want the monkey to target. Example: "namespace1, namespace2". The monkey will target all pods
      in these namespace unless they opt-out
      [env: TARGET_NAMESPACES=]  [default: default]

SUBCOMMANDS:
  fixed         Kill a fixed number of each pod group
  fixed-left    Kill pods until a fixed number of each pod group is alive
  percentage    Kill a percentage of each pod group
```

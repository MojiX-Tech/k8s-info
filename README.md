# k8s-info

A status report for any Kubernetes cluster, in one file.

`k8s-info setup` looks at the cluster, suggests what the report shall contain and asks the
user to confirm. After that `k8s-info` prints the report - whole or one section at a time.

The tool only reads. It never changes anything in the cluster.

## What it needs

| Needed | Why |
|---|---|
| bash 3.2 or newer | runs on Linux and macOS as they come |
| `kubectl` (or `oc`, `microk8s`, `k3s`) | found by the setup, in this order |
| `openssl` (optional) | expiry dates of the TLS certificates |
| `etcdctl` (optional) | only when etcd shall be read on a control plane directly |

No other tools (no `jq`, no Python).

## Start

```bash
curl -fsSLO https://raw.githubusercontent.com/MojiX-Tech/k8s-info/main/k8s-info
chmod +x k8s-info
./k8s-info setup
```

Read the file before you run it - it is one plain shell script.

The setup has seven steps:

1. **Connection** - asks how the cluster is reached: one of the contexts `kubectl` knows,
   a kubeconfig file it finds on a control plane, or a kubeconfig file whose path you type.
   A file with several clusters leads to one more question. Then the cluster gets a short
   name - the **environment**, for example `production` or `staging`.
2. **What is in the cluster** - metrics API, Longhorn, ingress controller, Gateway API,
   cert-manager, etcd. Read only.
3. **Title** of the report.
4. **Namespaces** - a list with the number of pods; every namespace you keep becomes its own
   section. You may give each one your own title.
5. **Cluster sections** - a list; only what was found in the cluster is offered.
6. **Summary** - nothing is written before you say yes.
7. **The command** - copies the file to `/usr/local/bin`, `~/.local/bin` or `~/bin`.
   An existing command of the same name is never replaced without a question.

After step 6 the setup asks whether there is another cluster. Yes starts again at step 1 for
the next one. With more than one environment it asks which one is used when none is named.

At the end the setup prints the help: how to call the command, the environments and the
sections.

In the lists you type numbers to switch lines on or off (`2 5` or `3-6`), `a` for all,
`n` for none, Enter to go on.

## Production, staging and other clusters

The tool does not guess which cluster is production. You tell it in the setup: every cluster
is set up once, with its own name, its own connection and its own sections.

```bash
k8s-info                   # the environment that was chosen as the usual one
k8s-info staging           # the whole report of staging
k8s-info staging nodes     # one section of staging
k8s-info staging --help    # the sections of staging
```

With more than one environment the box at the top of the report names it.

## Run the setup again

```bash
k8s-info setup
```

When something is set up already, the setup shows the environments and asks what to do:

| Choice | What happens |
|---|---|
| Change *environment* | the same questions; your choices of the last setup are the suggestion, titles and hand-made changes of the file stay, the order of the report stays, the old file is kept as `.bak` |
| Add another cluster | a new environment |
| Remove an environment | deletes its configuration file - the cluster is not touched |
| Only choose the usual environment | which one is used when none is named |
| Leave | nothing changes |

Without questions:

```bash
k8s-info setup staging                                    # change staging, or create it
k8s-info setup staging --kubeconfig ~/customer/stg.yaml   # with this kubeconfig file
k8s-info setup production --context prod-admin            # with this kubectl context
k8s-info setup --yes                                      # take every suggestion
```

`--yes` does not install the command.

## Use

```bash
k8s-info                  # the whole report
k8s-info nodes            # one section
k8s-info shop etcd        # several sections
k8s-info --help           # usage, environments, sections
k8s-info config           # show the configuration file
```

Colours are on in a terminal and off when the output goes to a file or a pipe
(`--color` / `--no-color` force it).

## Sections

| Section | Content |
|---|---|
| `nodes` | status, CPU and memory consumption, capacity, OS, kernel, container runtime |
| *namespace* | resources (`kubectl get all`, plus claims when the namespace has some), containers of every pod, consumption |
| `services` | Services of all namespaces |
| `configmaps` | ConfigMaps of the chosen namespaces |
| `secrets` | names and types of the Secrets in the chosen namespaces - never the values |
| `longhorn` | nodes, volumes with their claims, replicas, backups, snapshots, recurring jobs |
| `storage` | CSI drivers, storage classes, volumes, claims, attachments |
| `ingress` | ingress classes, controller, Ingress rules, Gateway API when installed |
| `certificates` | cert-manager, issuers, certificates, all TLS certificates sorted by days left |
| `system` | versions, API server health, namespaces, kube-system pods |
| `etcd` | member list, endpoint status and health, active leader |
| `health` | pods not ready, most restarts, incomplete workloads, latest warning events |

`longhorn` and `etcd` are offered only when the setup finds them. Consumption tables are left
out when the cluster has no metrics API.

**etcd** is read in one of two ways, chosen by the setup: `etcdctl` on this machine (on a
control plane that has it and may read the key), else `etcdctl` inside the etcd pod through
`kubectl exec`. The certificate paths come from the etcd pod itself. Clusters with a managed
control plane show no etcd pods, so the section is not offered there.

## Configuration file

| Who runs the setup | File |
|---|---|
| a user | `~/.config/k8s-info/<environment>.conf` |
| root | `/etc/k8s-info/<environment>.conf` |
| with `-c <file>` | that file |

`K8S_INFO_CONFIG_DIR` sets another directory. The file `default` in the same directory holds
the name of the environment that is used when none is named. The file is plain text with a comment above
every value and may be changed by hand. Things the setup does not ask, but the file can do:

- **Order** of the report: the order in `KI_SECTIONS`.
- **Command name** of a namespace section: `KI_APP_KEY`.
- **Leave lines out** of a namespace section: `KI_APP_EXCLUDE`, an extended regular
  expression, for example `maintenance|outage`.
- **Other resources** for a namespace: `KI_APP_RESOURCES`, for example `all,pvc,ingress`.
- **Two sections for one namespace**: add a second column with the same namespace, another
  key and another `KI_APP_EXCLUDE`.

The namespace lists (`KI_APP_*`) belong together: one column per namespace, the same position
in every list.

## Rights in the cluster

The report uses `get` and `list`, `kubectl top`, and for etcd in pod mode `pods/exec` in
`kube-system`. A section the account may not read shows the error of `kubectl` and the report
goes on. The TLS expiry table reads `tls.crt` of the TLS Secrets; answer no in the setup when
that is not wanted.

## Status

Version 1.0.0 is tested on self-managed clusters built with kubeadm (Kubernetes 1.31, Calico,
Longhorn, cert-manager, NGINX ingress). Managed clusters (EKS, GKE, AKS), k3s and OpenShift
are meant to work but are not tested yet - please open an issue with what you see.

## Licence

Apache License 2.0 - see [LICENSE](LICENSE).

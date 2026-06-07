# OS Images

- at this point in time CAPSTK has been tested only with Ubuntu 24.04
- required utilities such as `kubectl`, `containerd`, etc. are installed at runtime via cloud-init
- this will be more flexible in the future

```
stackit image list --project-id 4cf9e1f0-1f18-4c5b-bcc5-fbd3dd6675a5 --region eu01
--> 3ad2867e-695b-4ee6-9502-b563013413d4
```

## Flatcar Worker Nodes

`templates/cluster-template-flatcar-workers.yaml` creates a mixed image
cluster:

- control-plane machines keep using the Ubuntu image from `STACKIT_IMAGE_ID`
- worker machines use the Flatcar image from `STACKIT_WORKER_IMAGE_ID`

The currently tested STACKIT amd64 Flatcar image is:

```sh
export STACKIT_WORKER_IMAGE_ID=419c31da-39e3-4ea3-9bd8-699b44e8394f
```

This image is Flatcar Container Linux `4459.2.4`.

Flatcar does not run the Ubuntu `apt-get` based bootstrap commands from the
default template. Flatcar consumes STACKIT server user data as Ignition, usually
generated from Butane or Container Linux Config. The Flatcar worker template
therefore sets the worker `KubeadmConfigTemplate` to `format: ignition` and
uses `ignition.containerLinuxConfig.additionalConfig` to install the Kubernetes
node binaries under `/opt/bin` and wire the kubelet systemd unit to that path.
The template also reads `/run/metadata/flatcar` and uses
`COREOS_OPENSTACK_HOSTNAME` as the kubeadm node name. This is required because
Flatcar can otherwise register as `localhost`, which cloud-provider-stackit
cannot map back to the STACKIT server.

The upstream kubeadm bootstrap provider requires the experimental
`KubeadmBootstrapFormatIgnition` feature gate for this format. Enable it on the
kubeadm bootstrap controller before creating a Flatcar worker cluster:

```sh
kubectl -n capi-kubeadm-bootstrap-system patch deployment \
  capi-kubeadm-bootstrap-controller-manager \
  --type=json \
  -p='[{"op":"replace","path":"/spec/template/spec/containers/0/args/3","value":"--feature-gates=MachinePool=true,KubeadmBootstrapFormatIgnition=true,PriorityQueue=true,ReconcilerRateLimiting=true"}]'
```

Render and apply the template:

```sh
clusterctl generate cluster "${CLUSTER_NAME}" \
  --from templates/cluster-template-flatcar-workers.yaml \
  --kubernetes-version "${KUBERNETES_VERSION}" \
  --control-plane-machine-count "${CONTROL_PLANE_MACHINE_COUNT}" \
  --worker-machine-count "${WORKER_MACHINE_COUNT}" \
  | kubectl apply -f -
```

Notes:

- Flatcar SSH access uses the `core` user.
- The template does not install a CNI. Install the workload CNI after the API
  server is reachable.
- The Flatcar worker template fetches Kubernetes binaries from `dl.k8s.io`
  during first boot. A production image should bake or sysext-manage those
  binaries instead of downloading them during bootstrap.

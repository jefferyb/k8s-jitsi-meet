# Jitsi Meet on Kubernetes using kustomize

# Installation
This has been tested using [MicroK8s](https://microk8s.io/) but should still work with any Kubernetes

## 1. Create the `jitsi-meet namespace`:

```bash
$ kubectl create namespace jitsi-meet
```

## 2. Create the `jitsi-config secret`:
Make sure to change/set:
  - PUBLIC_URL: <Set to the Public URL for the web service>
  - DOCKER_HOST_ADDRESS: <Set the address for any node in the cluster here>

```bash
$ kubectl create secret generic jitsi-config -n jitsi-meet --from-literal=PUBLIC_URL='https://meet.example.com' --from-literal=DOCKER_HOST_ADDRESS='192.168.1.1' --from-literal=JICOFO_COMPONENT_SECRET="$(openssl rand -hex 16)" --from-literal=JICOFO_AUTH_PASSWORD="$(openssl rand -hex 16)" --from-literal=JVB_AUTH_PASSWORD="$(openssl rand -hex 16)" --from-literal=JIGASI_XMPP_PASSWORD="$(openssl rand -hex 16)" --from-literal=JIBRI_RECORDER_PASSWORD="$(openssl rand -hex 16)" --from-literal=JIBRI_XMPP_PASSWORD="$(openssl rand -hex 16)"
```

## 3. Add `"10000": jitsi-meet/jvb:10000`
Add `"10000": jitsi-meet/jvb:10000` to your nginx-ingress-udp-microk8s-conf/udp-services ConfigMap

PS: Make sure you open port `10000/UDP`on your firewall or your home router is directing that port to the right IP (`DOCKER_HOST_ADDRESS`) and it's accessible to from the outside, otherwise, your guest won't be able to join.

```bash
$ kubectl edit configmap nginx-ingress-udp-microk8s-conf -n ingress
```

And it should `"10000": jitsi-meet/jvb:10000` under data, like:
```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: nginx-ingress-udp-microk8s-conf
  namespace: ingress
data:
  "10000": jitsi-meet/jvb:10000
```
ref: 
  * https://kubernetes.github.io/ingress-nginx/user-guide/exposing-tcp-udp-services/
  * https://microk8s.io/docs/addon-ingress

## 4. Create a kustomization.yaml and other files

Create a kustomization.yaml and a ingress-patch.yaml file

```bash
# Create a ingress-patch.yaml file
cat <<EOF >./kustomization.yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization

bases:
  - jitsi
  - jitsi-etherpad

namespace: jitsi-meet

patchesJson6902:
  - path: meet-ingress-patch.yaml
    target:
      group: networking.k8s.io
      version: v1beta1
      kind: Ingress
      name: jitsi-ingress
EOF

# Create a meet-ingress-patch.yaml file
cat <<EOF >./meet-ingress-patch.yaml
- op: replace
  path: /spec/rules/0/host
  value: meet.your-host.com
EOF
```

To view the Deployment, run:

```bash
kubectl kustomize .
```

To apply/deploy, run:

```bash
kubectl kustomize . | kubectl apply -f -
#  OR
kubectl apply --kustomize .
```

### A few other examples
If you want to change the `JVB_PORT`, create a `'jvb-port-patch.yaml'` file and a `patchesJson6902` section to change the port

```bash
# Create a jvb-port-patch.yaml file
cat <<EOF >./jvb-port-patch.yaml
- op: replace
  path: /spec/template/spec/containers/0/env/9
  value:
    name: JVB_PORT
    value: "10000"

- op: replace
  path: /spec/template/spec/containers/0/ports/1
  value:
    name: proxy-udp-10000
    containerPort: 10000
    hostPort: 10000
    protocol: UDP
EOF

# Add jvb-port-patch.yaml to patchesJson6902 in the kustomization.yaml file
cat <<EOF >>./kustomization.yaml
  - path: jvb-port-patch.yaml
    target:
      group: apps
      version: v1
      kind: Deployment
      name: jvb
EOF

# view the Deployment, by running:
kubectl kustomize .

# apply/deploy, by running:
kubectl kustomize . | kubectl apply -f -
#  OR
kubectl apply --kustomize .
```

If you want to use configmap to configure `interface_config.js`, edit jitsi-config/interface_config.js and add it to `kustomization.yaml`

```bash
# Add to kustomization.yaml
cat <<EOF >>./kustomization.yaml
  - path: jitsi-config/interface-config-volumes-patch.yaml
    target:
      group: apps
      version: v1
      kind: Deployment
      name: jitsi

configMapGenerator:
  - name: jitsi-interface-config
    files:
      - jitsi-config/interface_config.js
EOF

# view the Deployment, by running:
kubectl kustomize .

# apply/deploy, by running:
kubectl kustomize . | kubectl apply -f -
#  OR
kubectl apply --kustomize .
```

## 5. Check on the deployment:
```bash
$ kubectl get pod -n jitsi-meet
NAME                       READY   STATUS    RESTARTS   AGE
jicofo-555df7f495-wqvld    1/1     Running   0          3h52m
jitsi-65bb9974f4-2lgnc     1/1     Running   0          3h52m
jvb-758ff57c4f-j8898       1/1     Running   0          3h51m
prosody-6446fd95fd-cn5nm   1/1     Running   0          3h52m
```

## 6. Access the web UI 
You should now be able to access your Jitsi Meet deployment at http://meet.your-host.com (or the Public URL that you set during step number 4)

ref:
  * https://jitsi.github.io/handbook/docs/devops-guide/devops-guide-docker
  * https://kubectl.docs.kubernetes.io/pages/examples/kustomize.html
  * https://kubernetes.io/docs/tasks/manage-kubernetes-objects/kustomization/
  * https://skryvets.com/blog/2019/05/15/kubernetes-kustomize-json-patches-6902/


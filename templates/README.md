# Jitsi Meet on Kubernetes

# Installation
This has been tested using [MicroK8s](https://microk8s.io/) but should still work with any Kubernetes

We're going to use ytt with our template. ytt is a fantastic templating engine for Kubernetes that gives us a lot of flexibility.

## 1. Edit the 'jitsi-meet-values.yaml' file with your values:

## 2. Create the `jitsi-meet namespace`:

```bash
$ kubectl create namespace jitsi-meet
```

## 3. Create the `jitsi-config secret`:
```bash
$ kubectl create secret generic jitsi-config -n jitsi-meet --from-literal=JICOFO_COMPONENT_SECRET="$(openssl rand -hex 16)" --from-literal=JICOFO_AUTH_PASSWORD="$(openssl rand -hex 16)" --from-literal=JVB_AUTH_PASSWORD="$(openssl rand -hex 16)" --from-literal=JIGASI_XMPP_PASSWORD="$(openssl rand -hex 16)" --from-literal=JIBRI_RECORDER_PASSWORD="$(openssl rand -hex 16)" --from-literal=JIBRI_XMPP_PASSWORD="$(openssl rand -hex 16)"
```

## 4. Add `"10000": jitsi-meet/jvb:10000`
Add `"10000": jitsi-meet/jvb:10000` to your nginx-ingress-udp-microk8s-conf/udp-services ConfigMap

PS: Make sure you open port `10000/UDP`on your firewall or your home router is directing that port to the right IP (`DOCKER_HOST_ADDRESS`) and it's accessible to from the outside, otherwise, your guest won't be able to join.

```bash
$ kubectl edit configmap nginx-ingress-udp-microk8s-conf -n ingress
```

And it should have `"10000": jitsi-meet/jvb:10000` under data, like:
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

## 5. Optional: Apply `jitsi-meet-configmap.yaml`
If you set `use_configmap: True` in your jitsi-meet-values.yaml file, then edit jitsi-meet-configmap.yaml and apply it

```bash
$ ytt -f jitsi-meet-configmap.yaml -f jitsi-meet-values.yaml | kubectl apply -f -
```

## 6. Apply `jitsi-meet-k8s.yaml`
Deploy the Jitsi and other services

```bash
$ ytt -f jitsi-meet-k8s.yaml -f jitsi-meet-values.yaml | kubectl apply -f -

# Or you can pass different values to ytt via command line, like:
$ ytt -f jitsi-meet-k8s.yaml -f jitsi-meet-values.yaml --data-value-yaml docker_host_address=192.168.3.37 --data-value-yaml jitsi.ingress.hostname=video.example.com | kubectl apply -f -

# Or you can use kapp for deploying your app
$ ytt -f jitsi-meet-k8s.yaml -f jitsi-meet-values.yaml | kapp deploy -y --diff-changes -a jitsi-meet -f-
```

## 7. Check on the deployment:
```bash
$ kubectl get pod -n jitsi-meet
NAME                       READY   STATUS    RESTARTS   AGE
jicofo-555df7f495-wqvld    1/1     Running   0          3h52m
jitsi-65bb9974f4-2lgnc     1/1     Running   0          3h52m
jvb-758ff57c4f-j8898       1/1     Running   0          3h51m
prosody-6446fd95fd-cn5nm   1/1     Running   0          3h52m
```

## 8. Access the web UI 
Access the web UI at http(s)://meet.example.com (or the Public URL that you set for jitsi.ingress.hostname in your jitsi-meet-values.yaml file)

ref:
  * https://jitsi.github.io/handbook/docs/devops-guide/devops-guide-docker
  * https://get-ytt.io/
  * https://get-kapp.io/


## 9. Update images weekly
If you want to update the latest images weekly, then edit crontab with
```bash
$ crontab -e

#Ansible: etherpad
0 3 * * 0 microk8s.kubectl rollout restart deployment/etherpad
#Ansible: jitsi
3 3 * * 0 microk8s.kubectl rollout restart deployment/jitsi
#Ansible: prosody
5 3 * * 0 microk8s.kubectl rollout restart deployment/prosody
#Ansible: jicofo
7 3 * * 0 microk8s.kubectl rollout restart deployment/jicofo
#Ansible: jvb
9 3 * * 0 microk8s.kubectl rollout restart deployment/jvb
```

# Useful commands

Here are some useful `kubectl` commands
https://kubernetes.io/docs/reference/kubectl/cheatsheet/

```bash
# Check on the 'jitsi-meet' events
$ kubectl get events -n jitsi-meet --all-namespaces --watch

# Get events with custom-columns
$ kubectl get events -o custom-columns=TIMESTAMP:lastTimestamp,TYPE:type,MESSAGE:message -w

# list ingress in 'jitsi-meet'
$ kubectl get ingress -n jitsi-meet

# check if the cert has been issued
$ kubectl describe ingress etherpad-ingress -n jitsi-meet | grep 'Successfully created Certificate'
    Normal  CreateCertificate  21m   cert-manager              Successfully created Certificate "letsencrypt-prod-etherpad"

# An example of using -o custom-columns
$ kubectl get pods --all-namespaces -o custom-columns=NAMESPACE:metadata.namespace,NAME:metadata.name,STATUS:status.phase

# Get logs of a container (if you have 1 pod in the namespace)
$ kubectl logs -f --namespace jitsi-meet pod/$(kubectl get pod --namespace jitsi-meet -o jsonpath='{.items[*].metadata.name}')

# Get logs of a container (using awk to select which pod)
$ kubectl logs -f --namespace jitsi-meet pod/$(kubectl get pod --namespace jitsi-meet -o jsonpath='{.items[*].metadata.name}' | awk '{print $1}')


# ref:
#   https://che.eclipse.org/running-eclipse-che-on-kubernetes-using-docker-desktop-for-mac-5d972ed511e1
```
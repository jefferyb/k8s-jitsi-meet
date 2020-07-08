# Jitsi Meet on Kubernetes

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

## 4. Optional: Apply `jitsi-meet-configmap.yaml`
If you want to configure the interface, such as the title of the application, etc.., then:
  * edit `jitsi-meet-configmap.yaml` with your settings, 
  * edit `jitsi-meet-k8s.yaml` and un-comment the sections under the volumeMounts & volumes in the jitsi Deployment section from
  ```yaml
          volumeMounts:
            ### OPTIONAL: Un-comment the next 3 lines to use the configMap
            # - name: jitsi-interface-config
            #   subPath: interface_config
            #   mountPath: /config/interface_config.js
      volumes:
        ### OPTIONAL: Un-comment the next 3 lines to use the configMap
        # - name: jitsi-interface-config
        #   configMap:
        #     name: jitsi-interface-config
  ```
  to 
  ```yaml
          volumeMounts:
            - name: jitsi-config
              mountPath: /config
            ### OPTIONAL: Un-comment the next 3 lines to use the configMap
            - name: jitsi-interface-config
              subPath: interface_config
              mountPath: /config/interface_config.js
      volumes:
        - name: jitsi-config
          emptyDir: {}
        ### OPTIONAL: Un-comment the next 3 lines to use the configMap
        - name: jitsi-interface-config
          configMap:
            name: jitsi-interface-config
  ```
  * apply it with:

```bash
$ kubectl apply -f jitsi-meet-configmap.yaml
```

## 5. Apply `jitsi-meet-k8s.yaml`
Deploy Jitsi and other services

```bash
$ kubectl apply -f jitsi-meet-k8s.yaml
```

## 6. Apply `jitsi-meet-ingress.yaml`
Change/edit the hostname, meet.example.com, and set it to the Public URL for the web service with

```bash
$ kubectl apply -f jitsi-meet-ingress.yaml
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
You should now be able to access your Jitsi Meet deployment at https://meet.example.com (or the Public URL that you set during step number 6)

ref:
  * https://jitsi.github.io/handbook/docs/devops-guide/devops-guide-docker

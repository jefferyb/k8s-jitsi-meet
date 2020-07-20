# Jitsi Meet on Kubernetes

# Installation
This has been tested using [MicroK8s](https://microk8s.io/) but should still work with any Kubernetes

# Methods
You have a choice of:
  1. using [kustomize](https://kustomize.io/), Kubernetes native configuration management. _(The Recommanded Method)_. You can find instructions in the __kustomize__ folder.
  1. using [ytt](https://get-ytt.io/), a templating tool that understands YAML structure
allowing you to focus on your data instead of how to properly escape it. You can find instructions in the __ytt__ folder.
  1. using the regular standard yaml files, but you'll have to edit more places in the files, depending what you want to run. You can find instructions in the __standard-yaml__ folder.
  

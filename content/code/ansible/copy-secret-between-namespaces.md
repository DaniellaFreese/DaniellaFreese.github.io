---
title: "How to Copy Secrets Between Namespaces with Kubernetes Core Ansible Collection"
date: 2023-01-15
draft: false

categories: ["ansible"]
tags: ["ansible", "kuberentes", "kuberentes core", "kubernetes secrets"]
toc: false
summary: "How to Copy Secrets Between Namespaces with Kubernetes Core Ansible Collection"
author: "djfreese"
---

With kubectl the typical command everyone uses to copy a secret from one namespace to another is the following:

```bash
kubectl get secret my-token --namespace default -o yaml | sed 's/namespace: .*/namespace: hello-world-namespace/' | kubectl apply -f -
```

It's a pretty straight forward, use `kubectl` get the secret as a yaml, use `sed` to replace the namespace and then `kubectl` again to apply the secret to the new namepace.

### Problem

In many of my playbooks I could do this same thing, as show in the sample task below:

```yaml
- name: "Fetch the token secret, Copy to Namespace {{ namespace }}"
  shell: |
    kubectl get secret m-token --namespace default -o yaml | sed 's/namespace: .*/namespace: {{ namespace }}/' | kubectl apply -f -
  register: ret
  failed_when: 
    - ret.rc == 1 
    - "'error when patching' not in ret.stderr"
```

But, the problem is now I have an external dependency to the kubectl binary, and that is not necesarily managed with ansible requirements file. So I deem this as a dependency that can easily get lost into the ether.

### Solution

So in order to rip out this dependency, I need to be able to the do the same thing with the kuberentes.core ansible collection.

This is what I came up with (this example happens to be a crt secret):

```yaml
- name: "Get Secret"
  kubernetes.core.k8s_info: 
    kind: Secret
    name: "{{ crt_secret }}" 
    wait: yes
    namespace: default
  register: crt_resource

- name: "Create crt_resource fact"
  ansible.builtin.set_fact: 
    crt_resource : "{{ crt_resource.resources[0] }}"

- name: "Create Secret in new Namespace"
  kubernetes.core.k8s:
    state: present
    force: yes 
    definition:
      apiVersion: v1
      kind: Secret
      metadata:
        annotations: "{{ crt_resource['metadata']['annotations'] }}"
        namespace: "{{ namespace }}"
        name: "{{ crt_secret }}"
      data: 
        tls.crt: "{{ crt_resource['data']['tls.crt'] }}"
        tls.key: "{{ crt_resource['data']['tls.key'] }}"
        ca.crt: "{{ crt_resource['data']['ca.crt'] }}"
```

Basically the steps are:

* register the secret as a fact
* clean up the fact a bit so it's easier to handle
* copy any data to a new secret in the new namespace

 In case I knew exactly what I was copying over, but if the attributes of data were to be more dynamic then you could even extend this examply furture with a jinja template to iterate and fill in the data portion of the manifest.

And that's it!

It may take an extra few steps but I've eliminated that kubectl dependency and gives me a little more piece of mind.

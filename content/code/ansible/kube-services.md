---
title: "Ansible - How to Wait for on a Kubernetes Service with the Kubernetes Core Collection"
date: 2023-01-13
draft: false

categories: ["ansible"]
tags: ["ansible", "kuberentes", "kuberentes core", "troubleshooting"]
toc: false
summary: "I recently got stuck trying to figure out how to wait for a Kuberentes Services - this post will cover how I handled it with the Kuberentes Core Ansible Collection."
author: "djfreese"
---

I have been using ansible to do some demo automation in Kubernetes and Openshift. I ran into an issue where a Kubernetes Service would still not be ready for consumption even though the automation has logic to wait until the Pod Status is ready. The main problem is for a Kuberentes Service of type ClusterIP the status portion of the object is never updated. To know if the Service is ready you need to check the endpoints object, if it is populated then the service is ready.

### Problem

The problem I was having was after installing the cert-manager in Openshift (via Operator Hub in Openshift) is it takes a few second to boostrap the resources and the next task, dependent on the cert-manager service to be available, would fail.

### Solution

To resolve the problem, after installing the operator, I wait for the cert-manager related pods to be to ready, then when I execute the next task, I put it into an ansible `until` loop, register the result and retry until the task succeeds. This was a slightly lazy hackish way of not having to parse out the endpoint object each time to check.

So this solution looks like the following:

```yaml
- name: Wait for Cert Manager Pod to be Ready
  kubernetes.core.k8s_info:
    kind: Pod
    api_version: v1
    label_selectors: 
      - app = "{{ item.label }}"
    namespace: openshift-operators
    wait: yes 
    wait_condition: 
      type: Ready
      status: True
  loop: 
    - { label: 'cert-manager' }
    - { label: 'webhook' }
    - { label: 'cainjector' }

# cert manager configuration
- name: Cert Manager Configuration
  block: 
    - name: Create Self-Signed Issuer Configuration 
      kubernetes.core.k8s: 
        state: present
        src: "files/selfsigned_issuer.yaml"
      register: result 
      until: result is not failed
      retries: 5

    - name: Wait for the SelfSigned Issuer to be Ready 
      kubernetes.core.k8s_info:
        api_version: cert-manager.io/v1
        kind: ClusterIssuer
        name: selfsigned-issuer
        wait: yes 
        wait_condition:
          type: Ready
          status: True
          reason: IsReady
    
    - name: Create CA Issuer from Self-Signed Issuer
      kubernetes.core.k8s: 
        state: present
        src: "files/selfsigned_ca_issuer.yaml"
      register: result 
      until: result is not failed
      retries: 5
      delay: 10
```

The End! It's not necessarily the most ideal solution but for the demo work I'm doing it hasn't failed me yet.
---
layout: post
title: Kubernetes on Proxmox and Talos - Day 1
description: Initial setup and image deployment
date: 2026-08-13 12:00:00 -0400
categories: [DevOps, Automation, Kubernetes, Talos, Proxmox]
tags: [devops, web development, automation, part 1]
author: Kyle Hartson
math: true
---

*A repository for this project can be found [here](https://github.com/khartson/proxmox-talos)*

## 1) Introduction and Goals 

This project is primarily for my own learning and experimentation with Kubernetes. Since my homelab is deployed almost exclusively on Proxmox, I wanted to use it as a platform to host a simple cluster. 

As always, the goal is to find a set of practical, repeatable, and automated procedures to deploy these resources. This series draws heavily on [this article](https://joshrnoll.com/creating-a-kubernetes-cluster-with-talos-linux-on-tailscale/) from Josh Noll and provided most of the framework for this experiment. 

__Day 1 Scope:__ 
  - Researching and determining the requirements to spec a Talos installation 
  - How to customize images on Talos Image Factory 
  - How to automate retrieval and deploment of these images to Proxmox with Terraform
  - Deploy a simple, 2 node cluster (1 worker, 1 control)

## 2) Specing an Image

By design, Talos is a lightweight distribution that ships with a minimal set of components to run Kubernetes. This may cause issues depending on which services or interfaces you plan on running in your cluster. Talos is immutable and cannot be modified after the installation, so it is important to spec a custom image that includes any necessary dependencies or customizations.

In my case I wil be following the same specs from Josh's article, but to overview the necessary requirements: 

### For Tailscale 

- `siderolabs/tailscale`

## For Longhorn 

- siderolabs/iscsi-tools
- `util-linux-tools` 

These are required because Longhorn needs to interact with the iSCSI protocol to manage storage across cluster nodes. `util-linux-tools` provides the required utilitiesi for running commands inside specific host namespaces. 

## For Proxmox 

- `siderolabs/qemu-guest-agent`

Determing which extensions you need may be a matter of a Google search, cross referencing documentation with Siderolabs' extensions, and trial and error. There are also a few additional configuration items to consider when creating the schematic, such as Cloud, machine architecture, etc. 

_A note on Tailscale:_ the extension is primarily for ease of remote _node_ management. Once the cluster is bootstrapped, however, if you need to expose service or ingress traffic, there will be an additional operator that can be used to expose the cluster's services to your tailnet.

## 3) Generating an image schematic

This can be done a number of ways - either through the interactive [web UI](https://factory.talos.dev) on the image factory website, or through via the API using a `yaml` schematic. For my first pass I used the UI, but you can easily get a schematic ID using the following. __Note:__ Extension order matters!!!! It will generate different schematic IDs, but it is not a huge deal. 

To do this via the API, create a schematic `yaml`: 

```yaml
# schematic.yaml
customization:
  systemExtensions:
    officialExtensions:
      - siderolabs/iscsi-tools
      - siderolabs/qemu-guest-agent
      - siderolabs/tailscale
      - siderolabs/util-linux-tools
```

Upload the schematic using `curl`:

```bash
curl -s -X POST https://factory.talos.dev/schematics -H \
  "Content-Type: application/yaml" \
  --data-binary @schematic.yaml
```

The response will look something like this:

```json
{
  "id": "077514df2c1b6436460bc60faabc976687b16193b8a1290fda4366c69024fec2",
  "schematic": "customization:\n    systemExtensions:\n        officialExtensions:\n            - siderolabs/iscsi-tools\n            - siderolabs/qemu-guest-agent\n            - siderolabs/tailscale\n            - siderolabs/util-linux-tools\n"
}
```

`077514df2c1b6436460bc60faabc976687b16193b8a1290fda4366c69024fec2` is the schematic ID you will need for the next step(s). The nice thing about the API is that this schematic ID remains stable/usable across version updates of Talos (at least for minor updates that I've tested).

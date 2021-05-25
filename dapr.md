---
layout: page
title: Introduce Dapr
---

## What is Dapr?

  I heard this <strong>Distributed Application Runtime</strong> ([Dapr](https://github.com/dapr/dapr)) concept from Scott Hanselman in his Azure Friday. 

  Dapr, an open source project that invented by Microsoft and contributed by all of the famous cloud providers, can be used to build event-driven, resilient distributed applications. It can help developers concentrate on implementing business logic without worrying about the cloud provider (e.g. AWS, Azure, GCP or Alibaba) or different SDKs from different cloud providers.

## Prerequisites

  * Install [Dapr CLI](https://docs.dapr.io/getting-started/install-dapr-cli/)
  * Install [kubectl](https://kubernetes.io/docs/tasks/tools/)
    "Docker Desktop" works fine for me, I will install "minikube" later. 

## Supported Bindings

  [https://docs.dapr.io/reference/components-reference/supported-bindings/](https://docs.dapr.io/reference/components-reference/supported-bindings/)
 
  ### Generic
   
   * [Twilio SendGrid](/2021/05/25/dapr-sendgrid/)
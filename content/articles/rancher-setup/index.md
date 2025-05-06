---
title: "Abstracting Cloud Providers and On Premise with Rancher"
date: 2025-05-07T14:23:00+02:00
draft: false
cover:
    image: "articles/rancher-setup/cover.png"
    alt: "cover"
    caption: ""
    relative: false
author:
    name: "Rajashree Gopalakrishnan"
    title: "Staff Devops Engineer"
    icon: "authors/rajashree.png"
    link: "https://www.linkedin.com/in/rajashreegopalakrishnan/"
summary: 
tags: 
- rancher
- argocd
- cloud
- infrastructure
---

This is the first part of a two-part blog series that explains how multi-cloud deployment is achieved using Rancher and ArgoCD at Equativ.

In today's cloud-native landscape, organizations often run Kubernetes workloads across multiple environments - public clouds like AWS, Google Cloud, as well as on-premises data centers. However, managing the clusters across different platforms can be challenging and adds operational complexity.

[Rancher](https://ranchermanager.docs.rancher.com/?_gl=1*i3nyr9*_ga*MTQxMTg1MjE5NS4xNzMzNzc3ODMy*_ga_Y7SFXF9L00*MTc0MzQyMzE1Ny4xMS4wLjE3NDM0MjMxNTkuNTguMC4w) simplifies multi-cluster Kubernetes management by providing a unified control plane that abstracts the underlying infrastructure, whether it's a cloud provider or an on-premises environment. The abstraction helps the developers to view all the K8s clusters at the same dashboard, whereas at the infra side we still have to manage them separately. At Equativ, we leverage Rancher to streamline operations across multiple clusters, managing environments across two cloud providers and bare-metal deployments in on-premise data centers. Rancher also provide a K8S distro that allow us to deploy on-premise, this could be replaced later by a kubeadm setup or more modern approaches like ClusterAPI. By abstracting the complexities of multi-cloud and on-premise deployments, Rancher enables DevOps teams to:

- Seamlessly manage clusters across various environments.
- Enforce consistent security policies and RBAC (Role-Based Access Control).
- Provide a centralized access point.

Though we use Rancher at Equativ, there are multiple other options available like k0rdent and Spectro cloud which could be explored. 

In this blog post, we will see technical details involved in setting up Rancher with various clusters from different environments. Whether you're managing Amazon EKS, Google GKE, or On premise Kubernetes clusters, Rancher ensures that all environments are treated equally - allowing you to focus on innovation rather than infrastructure complexity.

## Deploy Rancher in one of the Environments

Rancher can be deployed in either of the environments. In our case we have deployed it in our GKE cluster using the helm. We are using argoCD to help us bootstrap the rancher cluster. 
```yml
  source:
    repoURL: https://releases.rancher.com/server-charts/stable
    chart: rancher
    targetRevision: 2.8.5
    helm:
      releaseName: rancher
      values: |
        hostname: "rancher.example.com"
        ingress:
          ingressClassName: "haproxy-public"
          extraAnnotations:
            cert-manager.io/cluster-issuer: default-issuer
            cert-manager.io/issue-temporary-certificate: "true"
            external-dns.alpha.kubernetes.io/hostname: "rancher.example.com"
          tls:
            source: secret
            secretName: rancher-tls
```
It's important to note that:
- This refers to the deployment of the Rancher system itself.
- Since Rancher uses the etcd database, it's best practice to isolate it in a separate cluster to avoid potential side effects on the applications running in other clusters and to avoid cluttering it with internal resources.

Once this is setup, we should be able to access the rancher UI , which then enables us to monitor and access different Kubernetes cluster.

## Connect EKS, GKE and On Premise Kubernetes cluster to Rancher

This comes with an assumption that you already have created kubernetes clusters in AWS, GKE and/or Onpremise. The beauty of Rancher now is that, you can access all of these within a single dashboard using a single access point without having to jump around.

We would be doing this setup using Terraform.

To use Rancher with AWS, we'll need to create an IAM user with necessary permissions, and then configure Rancher to use those credentials to provision and manage Kubernetes clusters.

1. Create user

```hcl
resource "aws_iam_user" "delivery-rancher-admin-user" {
  name          = "rancher-user"
  path          = "/"
  force_destroy = true
}
```

2. Assign the permissions mentioned in [this document](https://ranchermanager.docs.rancher.com/how-to-guides/new-user-guides/kubernetes-clusters-in-rancher-setup/set-up-clusters-from-hosted-kubernetes-providers/eks#minimum-eks-permissions) to the above user

3. Create the Access Key for the user
```hcl
resource "aws_iam_access_key" "delivery-rancher-admin-user-key" {
  user = aws_iam_user.delivery-rancher-admin-user.name
}
```

4. Give the user access to Kubernetes API using AWS access entries. More information can be found [here](https://docs.aws.amazon.com/eks/latest/userguide/access-entries.html). And also, grant the user cluster admin role.

```hcl
resource "aws_eks_access_entry" "rancher_access" {
  cluster_name  = "eks_cluster"
  principal_arn = aws_iam_user.delivery-rancher-admin-user.arn
  type          = "STANDARD"
}

resource "aws_eks_access_policy_association" "rancher_access" {
  cluster_name  = "eks_cluster"
  policy_arn    = "arn:aws:eks::aws:cluster-access-policy/AmazonEKSClusterAdminPolicy"
  principal_arn = aws_iam_user.delivery-rancher-admin-user.arn

  access_scope {
    type       = "cluster"
    namespaces = []
  }
}
```

5. Now that the user is set up, we can proceed further to import EKS cluster into Rancher. Configure terraform provider for rancher2
```hcl
provider "rancher2" {
  api_url   = <rancher-host-url>
  token_key = <rancher-token-to-connect-to-rancher>
}
```
6. Create cloud credential for Rancher to be able to access AWS using the iam access key that we created in previous step.
```hcl
resource "rancher2_cloud_credential" "aws-eks-prod" {
  name        = "eks_cluster_cloud_credential"
  description = "rancher Cloud credential for EKS cluster"
  amazonec2_credential_config {
    access_key     = aws_iam_access_key.delivery-rancher-admin-user-key.id
    secret_key     = aws_iam_access_key.delivery-rancher-admin-user-key.secret
    default_region = "us-west-2"
  }
}
```
7. Import already existing EKS cluster into Rancher
```hcl
resource "rancher2_cluster" "cluster" {
  name        = "eks_cluster"
  description = "Terraform imported AWS cluster"
  eks_config_v2 {
    cloud_credential_id = rancher2_cloud_credential.aws-eks-prod.id
    name                = "eks_cluster"
    imported            = true
    region              = "us-west-2"
  }
}
```
8. We can also enable RBAC, to ensure restricted access to the clusters. For example here we are allowing prtg sensors (for alerting) to have access over the clusters.

```hcl
resource "rancher2_cluster_role_template_binding" "prtg-sensors" {
  name             = "prtg-sensors"
  cluster_id       = var.rancher_cluster_id
  role_template_id = data.rancher2_role_template.prtg-sensors.id
  user_id          = data.rancher2_user.prtg-sensors.id
}
```
With Rancher, we have successfully imported the AWS EKS cluster, allowing us to manage multiple clusters across different AWS regions through a unified dashboard. This abstraction ensures that irrespective of the region where an EKS cluster resides, Rancher provides a centralized platform for seamless management and governance.

Similarly, the same process can be applied to Google Cloud Platform (GCP). To connect a GKE cluster with Rancher:
You need to create a Service Account in GCP with the necessary permissions to manage the cluster. The detailed steps for creating the service account can be found [here](https://cloud.google.com/iam/docs/service-accounts-create).
Once the service account is generated, store its credentials securely in a vault.

These credentials can then be used to establish a connection between Rancher and the GKE cluster, enabling Rancher to manage and monitor GKE clusters just as effortlessly as EKS clusters.
```hcl
resource "rancher2_cloud_credential" "gcp" {
  name        = "gcp"
  description = "Cloud credential for cluster"
  google_credential_config {
    auth_encoded_json = data.vault_generic_secret.gcp-rancher-sa-credentials.data["credentials.json"]
  }
}

resource "rancher2_cluster" "cluster" {
  name        = "gke_cluster"
  description = "Terraform imported GKE cluster"
  gke_config_v2 {
    name                     = "gke_cluster"
    google_credential_secret = rancher2_cloud_credential.gcp.id
    region                   = "us-west-2"
    project_id               = "gcp-project"
    imported                 = true
  }

  depends_on = [
    module.cluster,
  ]
}
```

## Conclusion

With Rancher, the SRE team at Equativ can 
- Effortlessly manage multiple Kubernetes clusters across different platforms, all from a single unified dashboard. 
- Deploy to multi-cluster manually
- Have users groups and permissions be shared across cluster and environment
- Basic monitoring and alerting

However, we are currently missing few aspects including
- Unified networking
- Backups
- Centralized monitoring

As we continue to improve this process, be sure to follow this blog for updates on the latest enhancements.

The true power of Rancher is amplified when integrated with ArgoCD, offering a seamless and efficient deployment experience across various platforms and regions for both developers and SREs.

We will take a look at this in the next blog.
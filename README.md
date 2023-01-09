# K10-ExportImport-Ansible
***Status:** Work-in-progress. Please create issues or pull requests if you have ideas for improvement.*

# **Kasten K10 full automated Export and Import tasks with Ansible**
Example of using Ansible to automate K10 Export and Import policies to migrate applications to another Kubernetes cluster, or even to be used for Disaster Recovery purposes.

## Summary
This projects demostrates the process of exporting applications from a Production Kubernetes cluster using Kasten K10 policies, and then importing the same applications in a DR Production Kubernetes clusters using Kasten K10 Import Policies.  

All the automation is done using Ansible playbooks and leveraging the [Kasten K10 API](https://docs.kasten.io/latest/api/cli.html).

Kasten Export and Import policies can be used to migrate applications from one Kubernetes cluster to another (for instance from an on-prem Kubernetes cluster to an AWS EKS cluster), or even to provide a the tools for a Disaster Recovery strategy.

## Disclaimer
This project is an example of an deployment and meant to be used for testing and learning purposes only. Do not use in production. 


# Table of Contents

1. [Getting started](#Getting-started)
2. [Prerequisites](#Prerequisites)
3. [Recovering K10 and all Kubernetes Applications From a Disaster](#Recovering-K10-and-all-Kubernetes-Applications-From-a-Disaster)
4. [Parameters](#Parameters)
5. [Recovery duration](#Recovery-duration)
6. [File structure and deployment workflow](#File-structure-and-deployment-workflow)

# Getting started

The ability to move an application across clusters is an extremely powerful feature that enables a variety of use cases including Disaster Recovery (DR), Test/Dev with realistic data sets, and performance testing in isolated environments.

In particular, the K10 platform is being built to support application migration and mobility in a variety of different and overlapping contexts:

* Cross-Namespace: The entire application stack can be migrated across different namespaces in the same cluster (covered in restoring applications).
* Cross-Cluster: The application stack is migrated across non-federated Kubernetes clusters.
* Cross-Account: Mobility can additionally be enabled across clusters running in different accounts (e.g., AWS accounts) or projects (e.g., Google Cloud projects).
* Cross-Region: Mobility can further be enabled across different regions of the same cloud provider (e.g., US-East to US-West).
* Cross-Cloud: Finally, mobility can be enabled across different cloud providers (e.g., AWS to Azure).

To learn more about Kasten K10 export policies, please check this link: https://docs.kasten.io/latest/usage/migration.html#exporting-applications.
To learn more about Kasten K10 import policies, please check this link: https://docs.kasten.io/latest/usage/migration.html#importing-applications


## Prerequisites

To run this project you need to have some software installed and configured: 
1. A workstation with the next tools installed:
	- Kubectl
	- Kubernetes Collection for Ansible
	- Helm
	- jq
	- (If restoring in AWS EKS) aws-cli, aws-iam-authenticator and a AWS named profile.
	- (If restoring in Azure AKS) azure cli 
	- (If restoring in Google GKE) gcloud CLI and gke-gcloud-auth-plugin for use with kubectl 
1. A working [Ansible installation](https://docs.ansible.com/ansible/latest/installation_guide/intro_installation.html).
1. A working Kubernetes cluster with applications to be exported to another location (Production cluster).  In this project, we will provide the Ansible Playbooks for the following tasks:
  - Installing Kasten K10 in AWS EKS, Azure EKS or Google GKE.
  - Configuring Kasten Location Profiles to export the Applications
  - Configuring Kasten Export/Backup policies to protect the applications running in this cluster.  "Import Data" for every policy is captured by the Ansible playbook.

![Kasten DR](./docs/importing_data_details.png)	

1. A working clean Kubernetes cluster to be used to import all applications from Production cluster using Kasten K10.  In this project we will provide the Ansible Playbooks for the following tasks.
  - Installing Kasten K10 in AWS EKS, Azure EKS or Google GKE.
  - Configuring Kasten Location Profiles containing the data of the applications to be imported.
  - Configuring Kasten Import policies to import and restore the applications from the exported data.
1. Kasten K10 Export Policies protecting Kubernetes applications must have been run at least once to provide the required restore points for this project.

![Kasten DR](./docs/policies_migration_import.png)	
	

## Exporting Applications on Production Kubernetes cluster
Using K10 Export policies involves the following sequence of actions:

1. [Install Kasten K10 in the Production Kubernetes cluster](playbook/prod_cluster/01_k10_install.yaml)
1. [Configuring Locations Profiles](playbook/prod_cluster/02_k10_loc_profiles.yaml) to provide storage resources where the applications are going to be exported to. In this project, we can create multiple Location Profiles by providing the required data in a [CSV file](playbook/prod_cluster/vars/storage.csv)
  - profile_type (awss3, azblob, gcsa)
  - bucket_name: The name used when the bucket was created in the Cloud Provider.
  - profile_name: The name for the Kasten Location Profile:
  - region: Region required for AWS and Google Cloud
1. [Configure Export Policies](playbook/prod_cluster/03_k10_create_policy.yaml) for every application running in the Production cluster.   In this project, the Policies can be created automatically IF a label has been assigned to every NAMESPACE with one of these values:
  - backup=hourly for hourly backup
  - backup=daily for daily backup
  - backup=weekly for weekly backup
  - backup=monthly for monthly backup

All these steps will be automated using the Ansible playbooks provided for the [Production Cluster](playbook/prod_cluster)

## Parameters

**Deployment parameters:**

Some deployment variables must be set into the vars files.  Alter the parameters according to your needs:
1. For AWS EKS [playbook/aws_eks/vars/k10s3dr_vars.yaml](playbook/aws_eks/vars/k10s3dr_vars.yaml)
	- aws_access_key_id: AWS Access Key ID
	- aws_secret_access_key: AWS Secret Access Key
	- bucket_name: Name of the AWS S3 bucket containing the Kasten DR Backup
	- profile_name: Name of the Location Profile to be created in Kasten
	- region: Bucket Region when applicable 
	- passphrase: Passphrase used when enabling Kasten DR feature
	- clusterid: Cluster ID can be got from K10 when enabling Kasten DR feature
	- secret_name: Name of the secret to be created with Google Clous Storage Account Key
	- LOGIN: For K10 Basic Authentication.  Use 'htpasswd -n admin' and provide a password to autenticate to Kasten K10, then the result must be provided in this variable.
1. For Azure AKS [playbook/azure_aks/vars/k10aksdr_vars.yaml](playbook/azure_aks/vars/k10aksdr_vars.yaml)
	- tenantID: Azure Tenant ID
	- azureclientID: Azure Client ID
	- Azureclientsecret: Azure Client Secret
	- azure_storage_key: Azure Storage Access Key 
	- azure_storage_env: AzureCloud is the default in Azure.  More info in https://docs.kasten.io/latest/usage/configuration.html#azure-storage
	- bucket_name: Name of the Azure Blob Storage containing the Kasten DR Backup
	- profile_name: Name of the Location Profile to be created in Kasten
	- passphrase: Passphrase used when enabling Kasten DR feature
	- clusterid: Cluster ID can be got from K10 when enabling Kasten DR feature
	- secret_name: Name of the secret to be created with Azure Client secret
	- LOGIN: For K10 Basic Authentication.  Use 'htpasswd -n admin' and provide a password to autenticate to Kasten K10, then the result must be provided in this variable.
1. For Google Cloud GKE [playbook/google_gke/vars/k10gkedr_vars.yaml](playbook/google_gke/vars/k10gkedr_vars.yaml)
	- project_id: Google Cloud Project ID
	- bucket_name: Name of the Google Cloud Storage Account containing the Kasten DR Backup
	- profile_name: Name of the Location Profile to be created in Kasten
	- region: Bucket Region when applicable 
	- passphrase: Passphrase used when enabling Kasten DR feature
	- clusterid: Cluster ID can be got from K10 when enabling Kasten DR feature
	- secret_name: Name of the secret to be created with Google Clous Storage Account Key
	- LOGIN: For K10 Basic Authentication.  Use 'htpasswd -n admin' and provide a password to autenticate to Kasten K10, then the result must be provided in this variable.



# Recovery duration

The entire recovery process will take approx 30 minutes or more depending on the number of applications and data to recover. 

# File structure and deployment workflow

The Deployment consists thre main playbook triggering multible tasks:
1. 01_k10_install.yaml: 
	- This playbook installs a fresh Kasten K10 instance in the target Kubernetes cluster.
1. 02_k10_dr_restore.yaml:
	- This playbook creates the  Kubernetes Secret "k10-dr-secret" using the passphrase provided while enabling Disaster Recovery
	- Then, this playbook creates a Location Profile using the provided Object Storage, which of course MUST contain the Kasten configuration backup (this is the Location Profile used when enabling the Kasten K10 Disaster Recovery in the Production Kubernetes cluster).  
	- Finally this playbook restores the Kasten K10 configuration from the Location Profile created.
1. 03_k10_restoreapps.yaml: 
	- This playbook look for the most recent restore points available to restore the cluster-wide resources and each application.
	- Then the playbook restore the cluster-wide resources from the most recent restore point found in previous step.
	- Next the playbook creates a namespace for every application to be restored.
	- Finally the playbook restore every application from the most recent restore point found in previous step.

All these playbooks are available for all three Kubernetes deployments mentioned before:
1. [AWS EKS](playbook/aws_eks/)
1. [Azure AKS](playbook/azure_aks/)
1. [Google Cloud GKE](playbook/google_gke/)

Important: The playbooks must be run in the mentioned order in order to make the recovery process works.


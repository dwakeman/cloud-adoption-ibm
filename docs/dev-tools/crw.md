# CodeReady Workspaces

## Overview

One key tool that simplifies the life of a developer is [CodeReady Workspaces](https://developers.redhat.com/products/codeready-workspaces/overview).  

This section describes the steps to install and configure CodeReady Workspaces in [OpenShift in IBM Cloud](https://cloud.ibm.com/kubernetes/catalog/create?platformType=openshift).  It assumes that you already have a cluster, and that it is provisioned in VPC.

Steps:

1. Access the OpenShift console, access the OAuth CRD and add an identity provider.  Use Basic Authentication and use a fake URL, like https://doesnotexist.io.
1.  Install CRW using the operator.  Let it use `openshift-workspaces` as the project.
1. Configure the Che Cluster.  Change the PVC claim size from 1Gi to 10Gi.  This is because the minimum volume size in VPC is 10GB.  VPC does not support Read Write Many (RWX) so the same 
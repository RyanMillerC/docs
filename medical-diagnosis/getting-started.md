---
layout: default
title: Getting Started
grand_parent: Patterns
parent: Medical Diagnosis
nav_order: 1
---

# Deploying the Medical Diagnosis pattern

{: .no_toc }

## Table of contents

{: .no_toc .text-delta }

1. TOC
{:toc}

# Prerequisites

1. An OpenShift cluster (Go to [the OpenShift console](https://console.redhat.com/openshift/create)). Cluster must have a dynamic StorageClass to provision PersistentVolumes. See also [sizing your cluster](../../medical-diagnosis/cluster-sizing).
1. A GitHub account (and a token for it with repositories permissions, to read from and write to your forks)
1. S3-capable Storage set up in your public/private cloud for the x-ray images
1. The helm binary, see [here](https://helm.sh/docs/intro/install/)

The following packages need to be installed on your local system to seed the pattern correctly:

```bash
dnf install -y git make python3-pip
```

Use pip3 to install the following python packages required for ansible:

```bash
pip3 install --user --upgrade pip ansible kubernetes openshift awscli
```

Install the ansible-galaxy collection:

```bash
ansible-galaxy collection install kubernetes.core
```

The use of this pattern depends on having a Red Hat OpenShift cluster. In this version of the validated pattern
there is no dedicated Hub / Edge cluster for the **Medical Diagnosis** pattern.

If you do not have a running Red Hat OpenShift cluster you can start one on a
public or private cloud by using [Red Hat's cloud
service](https://console.redhat.com/openshift/create).

## Setting up an S3 Bucket for the xray-images

 An S3 bucket is required for image processing. Please see the [Utilities](#utilities) section below for creating a bucket in AWS S3. The following links provide information on how to create the buckets required for this validated pattern on several cloud providers.

* [AWS S3](https://docs.aws.amazon.com/AmazonS3/latest/userguide/creating-bucket.html)
* [Azure Blob Storage](https://docs.microsoft.com/en-us/azure/storage/common/storage-account-create?tabs=azure-portal)
* [GCP Cloud Storage](https://cloud.google.com/storage/docs/quickstart-console)

# Utilities

A number of utilities have been built by the validated patterns team to lower the barrier to entry for using the community or Red Hat Validated Patterns. To use these utilities you will need to export some environment variables for your cloud provider:

For AWS (replace with your keys):

```sh
export AWS_ACCESS_KEY_ID=AKXXXXXXXXXXXXX
export AWS_SECRET_ACCESS_KEY=gkXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXX
```

Then we need to create the S3 bucket and copy over the data from the validated patterns public bucket to the created bucket for your demo. You can do this on the cloud providers console or use the scripts provided on [validated-patterns-utilities](https://github.com/hybrid-cloud-patterns/utilities/) repository.

```sh
python s3-create.py -b mytest-bucket -r us-west-2 -p
python s3-sync-buckets.py -s com.validated-patterns.xray-source -t mytest-bucket -r us-west-2
```

The output should look similar to this edited/compressed output.

![Bucket setup](/videos/bucket-setup.svg)]

Keep note of the name of the bucket you created, as you will need it for further pattern configuration.
There is some key information you will need to take note of that is required by the 'values-global.yaml' file. You will need the URL for the bucket and its name. At the very end of the `values-global.yaml` file you will see a section for `s3:` were these values need to be changed.

# Preparation

1. Fork the [medical-diagnosis](https://github.com/hybrid-cloud-patterns/medical-diagnosis) repo on GitHub.  It is necessary to fork because your fork will be updated as part of the GitOps and DevOps processes.

1. Clone the forked copy of this repository.

   ```sh
   git clone git@github.com:<your-username>/medical-diagnosis.git
   ```

1. Create a local copy of the Helm values file that can safely include credentials

   **DO NOT COMMIT THIS FILE**

   You do not want to push credentials to GitHub.

   ```sh
   cp values-secret.yaml.template ~/values-secret.yaml
   vi ~/values-secret.yaml
   ```

  **values-secret.yaml example**

```yaml
secrets:
  xraylab:
    database-user: xraylab
    database-password: ## Insert your custom password here ##
    database-root-password: ## Insert your custom password here ##
    database-host: xraylabdb
    database-db: xraylabdb
    database-master-user: xraylab
    database-master-password: ## Insert your custom password here ##

  grafana:
    GF_SECURITY_ADMIN_PASSWORD: ## Insert your custom password here ##
    GF_SECURITY_ADMIN_USER: root
```

  When you edit the file you can make changes to the various DB and Grafana passwords if you wish.

1. Customize the `values-global.yaml` for your deployment

   ```sh
   git checkout -b my-branch
   vi values-global.yaml
   ```

**Replace instances of PROVIDE_ with your specific configuration**

   ```yaml
   ...omitted
   datacenter:
     cloudProvider: PROVIDE_CLOUD_PROVIDER #aws, azure
     storageClassName: PROVIDE_STORAGECLASS_NAME #gp2 (aws)
     region: PROVIDE_CLOUD_REGION #us-east-1
     clustername: PROVIDE_CLUSTER_NAME #OpenShift clusterName
     domain: PROVIDE_DNS_DOMAIN #blueprints.rhecoeng.com
   
    s3:
      # Values for S3 bucket access
      # Replace <region> with AWS region where S3 bucket was created
      # Replace <cluster-name> and <domain> with your OpenShift cluster values
      # bucketSource: "https://s3.<region>.amazonaws.com/<s3_bucket_name>"
      bucketSource: PROVIDE_BUCKET_SOURCE #"https://s3.us-east-2.amazonaws.com/com.validated-patterns.xray-source"
      # Bucket base name used for xray images
      bucketBaseName: "xray-source" 
   ```

   ```sh
   git add values-global.yaml
   git commit values-global.yaml
   git push origin my-branch
   ```

1. You can deploy the pattern using the [validated pattern operator](/infrastructure/using-validated-pattern-operator/). If you do use the operator then skip to Validating the Environment below.

1. Preview the changes that will be made to the Helm charts.

   ```sh
   make show
   ```

1. Login to your cluster using oc login or exporting the KUBECONFIG

   ```sh
   oc login
   ```

   or set KUBECONFIG to the path to your `kubeconfig` file. For example:

   ```sh
   export KUBECONFIG=~/my-ocp-env/auth/kubconfig
   ```

## Check the values files before deployment

You can run a check before deployment to make sure that you have the required variables to deploy the
Medical Diagnosis Validated Pattern.

You can run `make predeploy` to check your values. This will allow you to review your values and changed them in
the case there are typos or old values.  The values files that should be reviewed prior to deploying the
Medical Diagnosis Validated Pattern are:

| Values File | Description |
| ----------- | ----------- |
| values-secret.yaml | This is the values file that will include the xraylab section with all the database secrets |
| values-global.yaml | File that is used to contain all the global values used by Helm |

Make sure you have the correct domain, clustername, externalUrl, targetBucket and bucketSource values.

[![Predeploy](/videos/predeploy.svg)](/videos/predeploy.svg)

# Deploy

1. Apply the changes to your cluster

   ```sh
   make install
   ```

   If the install fails and you go back over the instructions and see what was missed and change it, then run `make update` to continue the installation.

1. This takes some time. Especially for the OpenShift Data Foundation operator components to install and synchronize. The `make install` provides some progress updates during the install. It can take up to twenty minutes. Compare your `make install` run progress with the following video showing a successful install.

   [![Demo](/videos/xray-deployment.svg)](/videos/xray-deployment.svg)

1. Check that the operators have been installed in the UI.

   ```text
   OpenShift UI -> Installed Operators
   ```

   The main operator to watch is the OpenShift Data Foundation.

## Using OpenShift GitOps to check on Application progress

You can also check on the progress using OpenShift GitOps to check on the various applications deployed.

1. Obtain the ArgoCD URLs and passwords.

   The URLs and login credentials for ArgoCD change depending on the pattern
   name and the site names they control.  Follow the instructions below to find
   them, however you choose to deploy the pattern.

   Display the fully qualified domain names, and matching login credentials, for
   all ArgoCD instances:

   ```sh
   ARGO_CMD=`oc get secrets -A -o jsonpath='{range .items[*]}{"oc get -n "}{.metadata.namespace}{" routes; oc -n "}{.metadata.namespace}{" extract secrets/"}{.metadata.name}{" --to=-\\n"}{end}' | grep gitops-cluster`
   CMD=`echo $ARGO_CMD | sed 's|- oc|-;oc|g'`
   eval $CMD
   ```

   The result should look something like:

   ```text
   NAME                       HOST/PORT                                                                                      PATH   SERVICES                   PORT    TERMINATION            WILDCARD
   hub-gitops-server   hub-gitops-server-medical-diagnosis-hub.apps.wh-medctr.blueprints.rhecoeng.com          hub-gitops-server   https   passthrough/Redirect   None
   # admin.password
   xsyYU6eSWtwniEk1X3jL0c2TGfQgVpDH
   NAME                      HOST/PORT                                                                         PATH   SERVICES                  PORT    TERMINATION            WILDCARD
   cluster                   cluster-openshift-gitops.apps.wh-medctr.blueprints.rhecoeng.com                          cluster                   8080    reencrypt/Allow        None
   kam                       kam-openshift-gitops.apps.wh-medctr.blueprints.rhecoeng.com                              kam                       8443    passthrough/None       None
   openshift-gitops-server   openshift-gitops-server-openshift-gitops.apps.wh-medctr.blueprints.rhecoeng.com          openshift-gitops-server   https   passthrough/Redirect   None
   # admin.password
   FdGgWHsBYkeqOczE3PuRpU1jLn7C2fD6
   ```

   The most important ArgoCD instance to examine at this point is `medical-diagnosis-hub`. This is where all the applications for the pattern can be tracked.

1. Check all applications are synchronised. There are thirteen different ArgoCD "applications" deployed as part of this pattern.

## Viewing the Grafana based dashboard

1. First we need to accept SSL certificates on the browser for the dashboard. In the OpenShift console go to the Routes for project openshift-storage. Click on the URL for the s3-rgw.

   [![Storage Routes](/images/medical-edge/storage-route.png)](/images/medical-edge/storage-route.png))

   Make sure that you see some XML and not an access denied message.

   [![Storage Routes](/images/medical-edge/storage-rgw-route.png)](/images/medical-edge/storage-rgw-route.png))

1. While still looking at Routes, change the project to `xraylab-1`. Click on the URL for the `image-server`. Make sure you do not see an access denied message. You ought to see a `Hello World` message.

   [![Storage Routes](/images/medical-edge/grafana-routes.png)](/images/medical-edge/grafana-routes.png))

1. Turn on the image file flow. There are three ways to go about this.

   You can go to the command-line (make sure you have KUBECONFIG set, or are logged into the cluster.

   ```sh
   oc scale deploymentconfig/image-generator --replicas=1 -n xraylab-1
   ```

   Or you can go to the OpenShift UI and change the view from Administrator to Developer and select Topology. From there select the `xraylab-1` project.

   [![Xraylab-1 Topology](/images/medical-edge/dev-topology.png)](/images/medical-edge/dev-topology.png))

   Right click on the `image-generator` pod icon and select `Edit Pod count`.

   [![Pod menu](/images/medical-edge/dev-topology-menu.png)](/images/medical-edge/dev-topology-menu.png))

   Up the pod count from `0` to `1` and save.

   [![Pod count](/images/medical-edge/dev-topology-pod-count.png)](/images/medical-edge/dev-topology-pod-count.png))

   Alternatively, you can have the same outcome on the Administrator console.

   Go to the OpenShift UI under Workloads, select Deploymentconfigs for Project xraylab-1. Click on `image-generator` and increase the pod count to 1.

   [![Image Pod](/images/medical-edge/start-image-flow.png)](/images/medical-edge/start-image-flow.png))

## Making some changes on the dashboard

You can change some of the parameters and watch how the changes effect the dashboard.

1. You can increase or decrease the number of image generators.

   ```sh
   oc scale deploymentconfig/image-generator --replicas=2
   ```

   Check the dashboard.

   ```sh
   oc scale deploymentconfig/image-generator --replicas=0
   ```

   Watch the dashboard stop processing images.

1. You can also simulate the change of the AI model version - as it's only an environment variable in the Serverless Service configuration.

   ```sh
   oc patch service.serving.knative.dev/risk-assessment --type=json -p '[{"op":"replace","path":"/spec/template/metadata/annotations/revisionTimestamp","value":"'"$(date +%F_%T)"'"},{"op":"replace","path":"/spec/template/spec/containers/0/env/0/value","value":"v2"}]'
   ```

   This changes the model version value, as well as the revisionTimestamp in the annotations, which triggers a redeployment of the service.

# Next Steps

[Help & Feedback](https://groups.google.com/g/hybrid-cloud-patterns){: .btn .fs-5 .mb-4 .mb-md-0 .mr-2 }
[Report Bugs](https://github.com/hybrid-cloud-patterns/medical-diagnosis/issues){: .btn .btn-red .fs-5 .mb-4 .mb-md-0 .mr-2 }

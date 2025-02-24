# OADP Demo

An Intro to OpenShift API for Data Protection (OADP).

## Intro

This demo highlights the basics for backing up and restoring a user namespace.  It uses OpenShift Data Foundation for storage classes and Object Storage.  In a production-ready environment, you would want to have object storage that is **external** to your OpenShift cluster.

## Step 1:  Install OADP Operator

You can do this with the CLI or the OpenShift console.  [Follow the documentation to deploy the OADP operator](https://docs.redhat.com/en/documentation/openshift_container_platform/4.16/html/backup_and_restore/oadp-application-backup-and-restore#installing-and-configuring-oadp).

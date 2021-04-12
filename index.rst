..
  Technote content.

  See https://developer.lsst.io/restructuredtext/style.html
  for a guide to reStructuredText writing.

  Do not put the title, authors or other metadata in this document;
  those are automatically added.

  Use the following syntax for sections:

  Sections
  ========

  and

  Subsections
  -----------

  and

  Subsubsections
  ^^^^^^^^^^^^^^

  To add images, add the image file (png, svg or jpeg preferred) to the
  _static/ directory. The reST syntax for adding the image is

  .. figure:: /_static/filename.ext
     :name: fig-label

     Caption text.

   Run: ``make html`` and ``open _build/html/index.html`` to preview your work.
   See the README at https://github.com/lsst-sqre/lsst-technote-bootstrap or
   this repo's README for more info.

   Feel free to delete this instructional comment.

:tocdepth: 1

.. Please do not modify tocdepth; will be fixed when a new Sphinx theme is shipped.

.. sectnum::

.. TODO: Delete the note below before merging new content to the master branch.

Introduction
============

`Velero`_ is an open-source tool to backup Kubernetes resources, including persistent volumes. 
It is an excellent complement to `Argo CD`_ for disaster recovery and data migration scenarios.

.. _Velero: https://velero.io
.. _Argo CD: https://argoproj.github.io/projects/argo-cd

With Argo CD, you can re-deploy your application and preserve the application state as long as the persistent volumes are preserved.
If a persistent volume (or any data in a persistent volume) is lost, then a disaster recovery tool like Velero becomes essential.

Velero implements custom resources like ``Backup``, ``Schedule``, and ``Restore`` and
includes controllers that process these custom resources to perform the related operations. 
You can back up or restore all Kubernetes objects in your cluster, or you can filter them by type, namespace, and/or label selectors.

Velero has a plugin architecture to work with different `cloud providers`_. 
But it also supports backing up and restoring Kubernetes volumes using a free open source backup tool called `restic`_.

.. _cloud providers: https://velero.io/docs/v1.5/supported-providers
.. _restic: https://github.com/restic/restic

This technote shows how to install and use Velero with the Velero Google Cloud Platform (GCP) plugin. 
The GCP plugin uses a Google Cloud Storage (GCS) bucket for the backup/restore location and Google Compute Engine (GCE) Snapshots to perform the persistent volume disks' snapshots.

Installation
============

In this section, you will install Velero with the GCP plugin into an existing Google Kubernetes Engine (GKE) cluster.

Create a GCS bucket
-------------------

Use the `gsutil`_ command-line tool to create a Cloud Storage bucket:

.. prompt:: bash $,# auto

   BUCKET=<bucket>
   $ gsutil mb gs://$BUCKET/

.. _gsutil: https://cloud.google.com/storage/docs/gsutil/commands/mb

Create an IAM service account
-----------------------------

Use the `gcloud`_ command-line tool to create an IAM service account:

.. prompt:: bash$ $,# auto 

   $ gcloud iam service-accounts create velero \
      --display-name "Velero service account"

.. _gcloud: https://cloud.google.com/sdk/gcloud/reference/iam/service-accounts

Create a role with enough permissions for Velero
------------------------------------------------

Get the ``project`` and service account ``email`` values and create a role with enough permissions for Velero:

.. prompt:: bash$ $,# auto

   PROJECT_ID=$(gcloud config get-value project)
   SERVICE_ACCOUNT_EMAIL=$(gcloud iam service-accounts list \
      --filter="displayName:Velero service account" \
      --format 'value(email)')
   ROLE_PERMISSIONS=(
      compute.disks.get
      compute.disks.create
      compute.disks.createSnapshot
      compute.snapshots.get
      compute.snapshots.create
      compute.snapshots.useReadOnly
      compute.snapshots.delete
      compute.zones.get
   )
   $ gcloud iam roles create velero.server \
      --project $PROJECT_ID \
      --title "Velero Server" \
      --permissions "$(IFS=","; echo "${ROLE_PERMISSIONS[*]}")"

Add an IAM policy binding to grant this role to Velero's GCP service account and to the bucket:

.. prompt:: bash$ $,# auto

   $ gcloud projects add-iam-policy-binding $PROJECT_ID \
      --member serviceAccount:$SERVICE_ACCOUNT_EMAIL \
      --role projects/$PROJECT_ID/roles/velero.server
   $ gsutil iam ch serviceAccount:$SERVICE_ACCOUNT_EMAIL:objectAdmin gs://${BUCKET}


Set permissions for Velero using Workload Identity 
--------------------------------------------------

Enable Workload Identity on your GKE cluster: 

.. prompt:: bash$ $,# auto
   
   CLUSTER=<cluster-name>
   ZONE=$(gcloud config get-value compute/zone)
   $ gcloud container clusters update $CLUSTER --zone=$ZONE \
      --workload-pool=$PROJECT_ID.svc.id.goog

Updated the existing node pools:

.. prompt:: bash$ $,# auto

   NODE_POOLS=<node-pools>
   $ gcloud container node-pools update $NODE_POOLS \
      --zone=$ZONE \
      --cluster=$CLUSTER \
      --workload-metadata=GKE_METADATA

Add an IAM policy binding to grant Velero's Kubernetes service account access to the GCP service account.

.. prompt:: bash$ $,# auto 

   $ gcloud iam service-accounts add-iam-policy-binding \
      --role roles/iam.workloadIdentityUser \
      --member "serviceAccount:$PROJECT_ID.svc.id.goog[velero/velero]" \
      velero@$PROJECT_ID.iam.gserviceaccount.com


Install Velero server using the GCP plugin
------------------------------------------

Finally, install the Velero server in your GKE cluster using the `Velero command-line client`_:

.. prompt:: bash$ $,# auto 

   $ velero install \
      --provider gcp \
      --plugins velero/velero-plugin-for-gcp:v1.2.0 \
      --bucket $BUCKET \
      --no-secret \
      --sa-annotations iam.gke.io/gcp-service-account=velero@$PROJECT_ID.iam.gserviceaccount.com \
      --backup-location-config serviceAccount=velero@$PROJECT_ID.iam.gserviceaccount.com \
      --wait

.. _`Velero command-line client`: https://velero.io/docs/v1.5/basic-install/

Backup and snapshot storage locations
=====================================

The Velero GCP plugin uses a GCS bucket to store backup and restore metadata, and the Kubernetes manifests for the resources included in the backup. 

The persistent volume backup is performed by `GCE disk snapshots`_. 

.. _GCE disk snapshots: https://cloud.google.com/compute/docs/disks/snapshots

Example:

.. code-block:: bash

   $ velero backup-location get
   NAME      PROVIDER   BUCKET/PREFIX        PHASE       LAST VALIDATED                  ACCESS MODE
   default   gcp        backup-sandbox-efd   Available   2021-04-06 16:13:34 -0700 MST   ReadWrite

   $ velero snapshot-location get
   NAME      PROVIDER
   default   gcp

Snapshots incrementally backup data from persistent disks. 


Velero backup and schedule
==========================

For an on-demand backup use the ``velero backup`` command, and for an scheduled backup use the ``velero schedule`` command instead.

**Example 1**: Schedule a backup of the entire application namespace every day with an expiration time set to 30 days.

.. prompt:: bash$ $,# auto 

   $ velero schedule create <schedule-name> \
      --schedule="@every 24h" \
      --include-namespaces <app-namespace> \
      --ttl 720h

**Example 2**: Schedule a backup of all persistent volumes in the cluster.

.. prompt:: bash$ $,# auto 

   $ velero schedule create <schedule-name> \
      --schedule="@every 24h" \
      --include-resources persistentVolumes\
      --ttl 720h

**Example 3**: Backup resources matching a label selector.

.. prompt:: bash$ $,# auto 

   $ velero backup create <backup-name> \
   --selector <key>=<value>


Velero restore
==============

You can use **namespace mapping** to restore the application to a different namespace.

.. prompt:: bash$ $,# auto 

   $ velero restore create \
      --from-schedule <schedule-name> \
      --namespace-mappings <original-namespace>:<restored-namespace>

You can also filter resources during a restore:

.. prompt:: bash$ $,# auto 

   $ velero restore create \
      --from-schedule <schedule-name> \
      --include-resources persistentvolumes


Disaster recovery
=================

**Scenario 1:** You have lost a persistent volume.

Use Velero to restore the persistent volume from the back up.

**Scenario 2:** "User A" accidentally deletes data from an application, and "User B" writes data to the same application roughly at the same time.

In this situation, you cannot simply restore the persistent volume from the backup. However, you can use Velero to restore the entire application namespace to a different namespace, connect to the application and manually restore the lost data.

Data migration
==============

**Scenario 1:** Migrate the application to another cluster.

Use Argo CD to deploy the application to the new cluster and use Velero to restore its previous state.

A practical example
===================

Let us take `Chronograf`_ as an example of a stateful application to illustrate how to restore *deleted data* from a backup. In this context, *delete data* could be a Chronograf dashboard, a Chronograf organization, or any configuration saved in the Chronograf database.

.. _`Chronograf`: https://docs.influxdata.com/chronograf/v1.8/

What follows assumes that Velero Server is installed in your cluster, and that the backup bucket is properly configured as described above. 

Create a Velero ``Schedule`` as follows:

.. prompt:: bash$ $,# auto 

   $ velero schedule create chronograf \
      --schedule="@every 24h" \
      --include-namespaces chronograf \
      --ttl 720h
   Schedule "chronograf" created successfully.
   $ velero schedule get
   NAME         STATUS    CREATED                         SCHEDULE      BACKUP TTL   LAST BACKUP   SELECTOR
   chronograf   Enabled   2021-04-09 12:44:50 -0700 MST   @every 24h    720h0m0s     4s ago        <none>

Open the Chronograf application and simulate a disaster by deleting a dashboard.

Restore the Chronograf application from the backup, but to a **different namespace**, for example, ``chronograf-restored``:

.. prompt:: bash$ $,# auto

   $ velero restore create \
      --from-schedule chronograf \
      --namespace-mappings chronograf:chronograf-restored
   Restore request "chronograf-20210409131002" submitted successfully.
   Run `velero restore describe chronograf-20210409131002` or `velero restore logs chronograf-20210409131002` for more details.

Use the following to disable authentication in the restored Chronograf application, otherwise, you'll be redirected to the original Chronograf application URL after logging in.

.. prompt:: bash$ $,# auto

   $ kubectl set env deployments --all TOKEN_SECRET- GH_CLIENT_ID- GH_CLIENT_SECRET- GH_ORGS- -n chronograf-restored

Ensure the Chronograf ``Pod`` has restarted:

.. prompt:: bash$ $,# auto

   $ kubectl delete --all pods -n chronograf-restored

Connect to the restored Chronograf application: 

.. prompt:: bash$ $,# auto

   $ kubectl port-forward -n chronograf-restored service/chronograf-chronograf 8000:80

Finally, export the deleted dashboard from the restored Chronograf application at ``http://localhost:8000``.

.. Do not include the document title (it's automatically added from metadata.yaml).

.. .. rubric:: References

.. Make in-text citations with: :cite:`bibkey`.

.. .. bibliography:: local.bib lsstbib/books.bib lsstbib/lsst.bib lsstbib/lsst-dm.bib lsstbib/refs.bib lsstbib/refs_ads.bib
..    :style: lsst_aa

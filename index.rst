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

.. note::

   **This technote is not yet published.**

   Instructions on how to deploy the EFD on Kubes (k3s), a lightweight Kubernetes.

TL;DR
=====

Summit EFD deployment on k3s managed by `Argo CD <https://argocd-summit.lsst.codes>`_.

If you need help, please drop a line on the ``#com-square-support`` LSSTC Slack channel.


Why are we doing this?
======================

The EFD system is based on Kafka and InfluxDB to record telemetry, commands, and events for the Rubin Observatory.
It was `prototyped and tested with the simulator for the M1M3 subsystem <https://sqr-029.lsst.io/#live-sal-experiment-with-avro-transformations>`_.
The next logical step is to deploy and test it with real hardware at the Summit.

The ability to deploy the EFD at different environments quickly and reproduce these deployments is crucial.
To solve this problem we've adopted `Kubernetes <https://kubernetes.io/>`_ as our deployment platform, `Helm <https://helm.sh/>`_ as the Kubernetes package manager and `Argo CD <https://argoproj.github.io/argo-cd/>`_ as the  continuous delivery tool for Kubernetes.

In this technote, we demonstrate that we can deploy the EFD on a single machine with Docker and `k3s <https://github.com/rancher/k3s>`_  while the final on-premise production cluster is not ready to receive the EFD.


Provisioning a k3s cluster
==========================

The first step is to provision our Kubernetes cluster.
`k3s <https://github.com/rancher/k3s>`_ is a lightweight Kubernetes that runs in a container, and `k3d <https://k3d.io>`_ is a helper tool to install k3s.

Requirements
------------

We assume a Linux box running Centos 7. We've installed `Docker CE <https://docs.docker.com/install/linux/docker-ce/centos/>`_ and `kubectl <https://kubernetes.io/docs/tasks/tools/install-kubectl/#install-kubectl-on-linux>`_:

  - Docker CE 20.10.10
  - kubectl 1.22.3

.. note::

  If you try these instructions locally, note that on Docker Desktop for Mac you `cannot route traffic from the host to the container <https://docs.docker.com/docker-for-mac/networking/>`_ (the ``--network host`` option does not work).

Configure Docker to start up on boot
------------------------------------

CentOS uses ``systemd`` to manage which services start when the system boots. Run the following to configure Docker to start on boot.

.. code-block:: bash

  sudo systemctl enable docker


Install k3d
-----------

We've installed:

  - k3d version v5.1.0
  - k3s version v1.21.5-k3s2 (default)

.. code-block:: bash

  curl -s https://raw.githubusercontent.com/rancher/k3d/main/install.sh | bash


Create the EFD cluster
----------------------

Create the EFD cluster with the following commands:

.. code-block:: bash

  export HOST_PATH=/data
  export CONTAINER_PATH=/var/lib/rancher/k3s/storage/

  sudo /usr/local/bin/k3d cluster create efd  --network host --no-lb --volume ${HOST_PATH}:${CONTAINER_PATH} --k3s-arg "--disable=traefik"

This command creates a cluster with only one node.

k3d ensures that the k3s container is automatically restarted after a system reboot.

k3d already deploys `local-path-provisioner <https://github.com/rancher/local-path-provisioner>`_.
Data is persisted at ``$HOST_PATH`` with the ``--volume`` option.
However you have to patch the persistent volumes later to change the reclaim policy to ``retain`` (the default is ``delete``).

The ``--network host`` option routes network traffic from the host to the k3s container, we need that to reach the different services running inside k3s using port ``443``.

Note that we are not deploying Traefik Ingress Controller which is the default option in k3d. We deploy the NGINX Ingress Controller with the EFD instead.

To connect to the cluster you need to copy the kubeconfig file from the container, and set the ``KUBECONFIG`` environment variable so that ``kubectl`` knows how to connect to the cluster:

.. code-block:: bash

  sudo /usr/local/bin/k3d kubeconfig get efd > k3s.yaml
  export KUBECONFIG=$(pwd)/k3s.yaml

  kubectl cluster-info

  Kubernetes control plane is running at https://0.0.0.0:6443
  CoreDNS is running at https://0.0.0.0:6443/api/v1/namespaces/kube-system/services/kube-dns:dns/proxy
  Metrics-server is running at https://0.0.0.0:6443/api/v1/namespaces/kube-system/services/https:metrics-server:/proxy

  To further debug and diagnose cluster problems, use 'kubectl cluster-info dump'

To connect to the cluster from another machine, copy the ``k3s.yaml`` file and replace ``localhost`` by the machine IP address.


Add workers (optional)
----------------------

If there are other machines available, you might want to add more servers to the cluster.  Refer to the `k3d documentation <https://k3d.io/v5.1.0/usage/multiserver/>`_ on how to create a multi-server cluster.

Install Argo CD
===============

With Argo CD we keep the `EFD deployment configuration on GitHub <https://github.com/lsst-sqre/argocd-efd>`_. This way we can use GitHub to control changes in the EFD deployments and easily bootstrap new EFD deployments.


.. code-block:: bash

  kubectl create namespace argocd
  kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml

This creates a new namespace, ``argocd``, where Argo CD services and application resources will live.

Follow `these instructions to install Argo CD CLI <https://argoproj.github.io/argo-cd/cli_installation/>`_.

Additional Argo CD configuration includes `Single Sign On (SSO) <https://argoproj.github.io/argo-cd/operator-manual/sso/>`_ and the `Role Based Access Control (RBAC) <https://argoproj.github.io/argo-cd/operator-manual/rbac/>`_.

In particular, this is the RBAC configuration we added to the ArgoCD ConfigMap ``argocd-rbac-cm`` to let members of the EFD ops team in the ``lsst-sqre`` GitHub organization to synchronize the EFD configuration, while granting read-only permission for other members.

.. code-block:: bash

  data:
    policy.csv: |
      p, lsst-sqre:EFD ops, applications, sync, default/*, allow

      g, lsst-sqre:EFD ops, role:admin
    policy.default: role:readonly


Deploy the EFD
==============

Once Argo CD is installed we can deploy the EFD.

Argo CD manages the deployment of the EFD on multiple environments.

For example, the following bootstraps an EFD deployment using the configuration for the ``summit`` environment:

.. code-block:: bash

  kubectl port-forward svc/argocd-server -n argocd 8080:443

  kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}" | base64 -d

  argocd login --insecure localhost:8080
  argocd app create efd --dest-namespace argocd --dest-server https://kubernetes.default.svc --repo https://github.com/lsst-sqre/argocd-efd.git --path apps/efd --helm-set env=summit

  argocd app sync efd

See the Argo CD getting started guide for further instructions on `how to login using the CLI <https://argoproj.github.io/argo-cd/getting_started/#4-login-using-the-cli>`_.

The secrets used by the EFD are stored on `LSST's Vault Service <https://vault.lsst.codes/>`_ but you need to create at least one secret manually with the Vault read token for this environment.

Sync the ``vault-secrets-operator`` app:

.. code-block:: bash

  argocd app sync vault-secrets-operator

.. code-block:: bash

  export VAULT_TOKEN=<vault token>
  export VAULT_TOKEN_LEASE_DURATION=8640

  kubectl create secret generic vault-secrets-operator --from-literal=VAULT_TOKEN=$VAULT_TOKEN --from-literal=VAULT_TOKEN_LEASE_DURATION=$VAULT_TOKEN_LEASE_DURATION --namespace vault-secrets-operator

Sync the ``ingress-nginx`` app:

.. code-block:: bash

  argocd app sync ingress-nginx

Finally sync the other apps either using Argo CD CLI or the GUI.

Create the `tls-certs` secret for Argo CD:

.. code-block:: bash

  cat << EOF | kubectl apply -f -
  apiVersion: ricoberger.de/v1alpha1
  kind: VaultSecret
  metadata:
    name: argocd-server-tls
    namespace: argocd
  spec:
    path: secret/k8s_operator/summit-lsp.lsst.codes/efd/tls-certs
    type: Opaque
  EOF

Service names follow the convention ``<app>-<environment>-efd.lsst.codes``, for example, ``chronograf-summit-efd.lsst.codes``

Change reclaim policy of new volumes to retain

.. code-block:: bash

  kubectl patch pv <new pv> -p '{"spec":{"persistentVolumeReclaimPolicy":"Retain"}}'


For Kafka, the broker URL for the Summit EFD is ``kafka-0-summit-efd.lsst.codes:31090``.

Testing the EFD
===============

The EFD deployment can be tested using `kafkacat <https://docs.confluent.io/current/app-development/kafkacat-usage.html>`_  a command line utility implemented with ``librdkafka`` the Apache Kafka C/C++ client library.

Run in producer mode (``-P``) to produce messages for a test topic:

.. code-block:: bash

  kafkacat -P -b <kafka broker url> -t test_topic
  Hello EFD!
  ^D

Run in metadata listing mode (``-L``) to retrieve metadata from the cluster:

.. code-block:: bash

  kafkacat -L -b <kafka broker url>

The ``-d`` option enables ``librdkafka`` debugging. For instance, ``-d broker`` can be used to debug connection issues with the cluster:

.. code-block:: bash

  kafkacat -L -b <kafka broker url>  -d broker

Run in consumer mode (``-C``) to consume topics from the cluster:

.. code-block:: bash

  kafkacat -C -b <kafka broker url> -t <topic name>


Restarting the EFD manually
==============================

k3d ensures k3s automatically starts after a system reboot.
In case you need to start the k3s container manually, first check its status:

.. code-block:: bash

  sudo docker ps -a

If the k3s container status is ``Exited``, start it with the following command:

.. code-block:: bash

  sudo docker start <k3s container name>

After a few minutes, all Kubernetes pods should be running again:

.. code-block:: bash

  export KUBECONFIG=$(pwd)/k3s.yaml
  kubectl cluster-info
  kubectl get pods --all-namespaces


Inspecting logs, restarting pods, etc.
======================================

You can inspect logs, restart pods and do other operations including synchronization of the deployment using the Argo CD UI or the CLI.

In the case of the EFD, we have most often needed to restart the kafka connector pod ``confluent-cp-kafka-connect`` in the ``cp-helm-charts`` namespace.

Accessing EFD data
=====================

Use the Chronograf UI for data visualization and dashboarding.

In this `notebook <https://github.com/lsst-sqre/notebook-demo/blob/master/experiments/efd/Accessing_EFD_data.ipynb>`_ we show how to access EFD data using the EFD Python client, and proceed with data analysis using Pandas dataframes.


.. Add content here.
.. Do not include the document title (it's automatically added from metadata.yaml).

.. .. rubric:: References

.. Make in-text citations with: :cite:`bibkey`.

.. .. bibliography:: local.bib lsstbib/books.bib lsstbib/lsst.bib lsstbib/lsst-dm.bib lsstbib/refs.bib lsstbib/refs_ads.bib
..    :style: lsst_aa

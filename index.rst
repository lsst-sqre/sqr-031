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

   Instructions on how to deploy the EFD on Kubes, a lightweight Kubernetes.

TL;DR
=====

.. list-table::

   * - Summit EFD
     - https://argocd-summit.lsst.codes
   * - Tucson Teststand EFD
     - https://argocd-tucson-teststand.lsst.codes
   * - NCSA Teststand EFD
     - https://argocd-ncsa-teststand.lsst.codes


Follow ``#com-efd`` at LSSTC Slack for updates.

Why are we doing this?
======================

The EFD is a solution based on Kafka and InfluxDB for recording telemetry, commands, and events for LSST. It was `prototyped and tested with the simulator for the M1M3 subsystem <https://sqr-029.lsst.io/#live-sal-experiment-with-avro-transformations>`_. The next logical step is to deploy and test the EFD with real hardware.

The ability to deploy the EFD at different environments quickly and reproduce these deployments is crucial.  To solve this problem we've adopted `Kubernetes <https://kubernetes.io/>`_ as our deployment platform, `Helm <https://helm.sh/>`_ as the Kubernetes package manager and `Argo CD <https://argoproj.github.io/argo-cd/>`_ as the  continuous delivery tool for Kubernetes.

In this technote, we demonstrate that we can deploy the EFD on a single machine with Docker and `k3s  <https://github.com/rancher/k3s>`_ ("Kubes") while the final on-premise deployment platform is not ready.


Provisioning a k3s ("kubes") cluster
====================================

The first step is to provision our Kubernetes cluster.  `K3s <https://github.com/rancher/k3s>`_ is a lightweight Kubernetes that runs in a container.

Requirements
------------

We assume a Linux box running Centos 7. We've installed `Docker CE <https://docs.docker.com/install/linux/docker-ce/centos/>`_ and `kubectl <https://kubernetes.io/docs/tasks/tools/install-kubectl/#install-kubectl-on-linux>`_:

 - Docker CE 18.09.6
 - kubectl 1.14.1

.. note::

  We also tried k3s locally, on Docker Desktop for Mac, but it `can not route traffic from the host to the container <https://docs.docker.com/docker-for-mac/networking/>`_ (the ``--network host`` option does not work).

Configure Docker to start on boot
---------------------------------

CentOS uses ``systemd`` to manage which services start when the system boots. Run the following to configure Docker to start on boot.

.. code-block:: bash

  sudo systemctl enable docker


Run the k3s master
------------------

Run the k3s master with the following commands:

.. code-block:: bash

  export K3S_PORT=6443
  export K3S_URL=http://localhost:${K3S_PORT}
  export K3S_TOKEN=$(date | base64)
  export HOST_PATH=/data # change depending on your host
  export CONTAINER_PATH=/opt/local-path-provisioner
  sudo docker run  -d --restart always --tmpfs /run --tmpfs /var/run --volume ${HOST_PATH}:${CONTAINER_PATH} -e K3S_URL=${K3S_URL} -e K3S_TOKEN=${K3S_TOKEN} --privileged --network host --name master docker.io/rancher/k3s:v0.5.0-rc1 server --https-listen-port ${K3S_PORT} --no-deploy traefik

The  ``--restart always`` option ensures that the k3s master is `automatically restarted <https://docs.docker.com/config/containers/start-containers-automatically/>`_ after a system reboot.

Data is persisted at ``$HOST_PATH`` with the ``--volume`` option (see also :ref:`local-path-provisioner`).

The ``--network host`` option routes network traffic from the host to the container, we need that to reach the different services running inside k3s.

Note that we are not deploying `Traefik Ingress Controller` which is included in the k3s docker image, because the EFD already deploys the `NGINX Ingress Controller`.

To connect to the master you need to copy the kubeconfig file from the container, and set the ``KUBECONFIG`` environment variable so that `kubectl` knows how to connect to the cluster:

.. code-block:: bash

  sudo docker cp master:/etc/rancher/k3s/k3s.yaml k3s.yaml
  export KUBECONFIG=$(pwd)/k3s.yaml
  kubectl cluster-info

  Kubernetes master is running at https://localhost:6443
  CoreDNS is running at https://localhost:6443/api/v1/namespaces/kube-system/services/kube-dns:dns/proxy

  To further debug and diagnose cluster problems, use 'kubectl cluster-info dump'.

To connect to the cluster from another machine, copy the ``k3s.yaml`` file and replace ``localhost`` by ``140.252.32.142`` for the lab instance and ``139.229.162.114`` for the summit instance.

Note that we will likely also keep current versions of the configuration files in 1Password.  Look for ``k3s-summit.yaml`` and ``k3s-test.yaml``.

.. _local-path-provisioner:

Deploy the local-path provisioner
---------------------------------

The `local-path provisioner <https://github.com/rancher/local-path-provisioner>`_ creates ``hostPath`` persistent volumes on the node automatically. The directory ``/opt/local-path-provisioner`` is used as the path for provisioning. The provisioner is installed in the ``local-path-storage`` namespace by default.


.. code-block:: bash

  kubectl apply -f https://raw.githubusercontent.com/rancher/local-path-provisioner/master/deploy/local-path-storage.yaml

At this point you should see the following pods running in the cluster:

.. code-block:: bash

  kubectl get pods --all-namespaces
  NAMESPACE            NAME                                      READY   STATUS    RESTARTS   AGE
  kube-system          coredns-695688789-r9gkt                   1/1     Running   0          5m
  local-path-storage   local-path-provisioner-5d4b898474-vz2np   1/1     Running   0          4s


Add workers (optional)
----------------------

If there are other machines, you can easily add workers to the cluster. Copy the ``node-token`` from the master:

.. code-block:: bash

  sudo docker cp master:/var/lib/rancher/k3s/server/node-token node-token

and start the worker(s):

.. code-block:: bash

  export SERVER_URL=https://<master external IP>:${K3S_PORT}
  export NODE_TOKEN=$(cat node-token)
  export WORKER=kube-0
  export HOST_PATH=/data # change depending on your host
  export CONTAINER_PATH=/opt/local-path-provisioner
  sudo docker run -d --tmpfs /run --tmpfs /var/run -v ${HOST_PATH}:${CONTAINER_PATH} -e K3S_URL=${SERVER_URL} -e K3S_TOKEN=${NODE_TOKEN} --privileged --name ${WORKER} rancher/k3s:v0.5.0-rc1

.. note::

	By default ``/opt/local-path-provisioner`` is used across all the nodes to store persistent volume data, see `local-path provisioner configuration <https://github.com/rancher/local-path-provisioner#configuration>`_.

Install Argo CD
===============

With Argo CD we keep the `EFD deployment configuration on GitHub <https://github.com/lsst-sqre/argocd-efd>`_. This way we can use GitHub to control changes in the EFD deployments and easily bootstrap new EFD deployments.


.. code-block:: bash

  kubectl create namespace argocd
  kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml


This creates a new namespace, ``argocd``, where Argo CD services and application resources will live.

Follow these instructions to `install Argo CD CLI <https://argoproj.github.io/argo-cd/cli_installation/>`_.

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

Argo CD manages the deployment of the EFD on multiple environments. The possible environments are ``summit``, ``tucson-teststand``, ``ncsa-teststand``, ``ldf``, and ``gke``.

For example, the following bootstraps an EFD deployment using the configuration for the ``summit`` environment:


.. code-block:: bash

  kubectl port-forward svc/argocd-server -n argocd 8080:443

  argocd login localhot:8080
  argocd app create efd --dest-namespace argocd --dest-server https://kubernetes.default.svc --repo https://github.com/lsst-sqre/argocd-efd.git --path apps/efd --helm-set env=summit
  argocd app sync efd

The secrets used by the EFD are stored on `LSST's Vault Service <https://vault.lsst.codes/>`_ but you need to create at least one secret manually with the VAULT_TOKEN:

.. code-block:: bash

  export VAULT_TOKEN=<vault token>
  export VAULT_TOKEN_LEASE_DURATION=<vault token lease duration>

  kubectl create secret generic vault-secrets-operator --from-literal=VAULT_TOKEN=$VAULT_TOKEN --from-literal=VAULT_TOKEN_LEASE_DURATION=$VAULT_TOKEN_LEASE_DURATION --namespace vault-secrets-operator

Service names for the apps follow the convention ``<app>-<environment>-efd.lsst.codes``, for example, ``chronograf-summit-efd.lsst.codes``

In particular, the broker URL for the Summit EFD is ``kafka-0-summit-efd.lsst.codes:30190``.

Testing the EFD
===============

The EFD deployment can be tested using `kafkacat <https://docs.confluent.io/current/app-development/kafkacat-usage.html>`_  a command line utility implemented with ``librdkafka`` the Apache Kafka C/C++ client library.

Run in producer mode (``-P``) to produce messages for a test topic:

.. code-block::

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

k3s is configured to automatically start after a system reboot (``--restart-always`` flag). In case you need to start the k3s master manually, first check its status:

.. code-block:: bash

  sudo docker ps -a

If k3s master status is ``Exited`` start with the following command:

.. code-block:: bash

  sudo docker start master

After a few minutes, all Kubernetes pods should be running again:

.. code-block:: bash

  export KUBECONFIG=$(pwd)/k3s.yaml
  kubectl cluster-info
  kubectl get pods --all-namespaces


Inspecting logs, restarting pods, etc.
======================================

You can inspect logs, restart pods and doing other operations including synchronization of the deployment using the Argo CD UI or the CLI.

In the case of the EFD, we have most often needed to restart the kafka connector pod ``confluent-cp-kafka-connect`` in the ``cp-helm-charts`` namespace.

Accessing EFD data
=====================

Use the Chronograf interface for time-series visualization and dashboarding.

In this `notebook <https://github.com/lsst-sqre/notebook-demo/blob/master/experiments/efd/Accessing_EFD_data.ipynb>`_ we show how to extract data from the EFD using `aioinflux <https://aioinflux.readthedocs.io/en/stable/index.html>`_, a Python client for InfluxDB, and proceed with data analysis using Pandas dataframes.







.. Add content here.
.. Do not include the document title (it's automatically added from metadata.yaml).

.. .. rubric:: References

.. Make in-text citations with: :cite:`bibkey`.

.. .. bibliography:: local.bib lsstbib/books.bib lsstbib/lsst.bib lsstbib/lsst-dm.bib lsstbib/refs.bib lsstbib/refs_ads.bib
..    :style: lsst_aa

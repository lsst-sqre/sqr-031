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

   Instructions to deploy and operate the DM-EFD

Why are we doing this?
======================

The DM-EFD is a solution based on Kafka and InfluxDB for recording telemetry, commands, and events for LSST. It was `prototyped and tested with the simulator for the M1M3 subsystem <https://sqr-029.lsst.io/#live-sal-experiment-with-avro-transformations>`_. The next logical step is to deploy and test the DM-EFD with real hardware.

The Auxilary Telescope Camera (ATCamera) is being tested at the Tucson lab, and it presents an excellent opportunity for testing the DM-EFD by recording data from these tests.

We might need to deploy the DM-EFD at the Summit, Base facility, and LDF. Thus, the ability to deploy the DM-EFD at different environments quickly and reproduce these deployments is crucial.  To solve this problem we've adopted `Docker <https://www.docker.com/>`_ and `Kubernetes <https://kubernetes.io/>`_ as our deployment platform, and a combination of `Terragrunt <https://www.gruntwork.io/>`_, `Terraform <https://www.terraform.io/>`_, and `Helm <https://helm.sh/>`_ to manage and automate the deployments.

In this technote, we demonstrate that we can deploy the DM-EFD on a single machine with Docker and `k3s  <https://github.com/rancher/k3s>`_ ("kubes"), a lightweight Kubernetes using the same Terraform modules and Helm charts that we used at Google Could. We also provide instructions on how to operate and use the DM-EFD system.

The ATCamera test environment
=============================

As of June 2019, the ATCamera test environment at the Tucson lab runs SAL 3.9. The following subsystems are being tested and produce data: ``ATCamera``, ``ATHeaderService``, ``ATArchiver``, ``ATMonochromator``, ``ATSpectrograph``, and ``Electrometer``.

Kafka writers are responsible for sending messages from each SAL topic to the Kafka brokers.

The DM-EFD runs on a dedicated machine ``ts-csc-01 (140.252.32.142)``. The first step to deploy it is to provision a Kubernetes cluster.

Provisioning a k3s ("kubes") cluster
====================================

`k3s <https://github.com/rancher/k3s>`_ is a lightweight Kubernetes that runs in a container.

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

Note that we are not deploying `Traefik Ingress Controller` which is included in the k3s docker image, because the DM-EFD already deploys the `NGINX Ingress Controller`.

To connect to the master you need to copy the kubeconfig file from the container, and set the ``KUBECONFIG`` environment variable so that `kubectl` knows how to connect to the cluster:

.. code-block:: bash

  sudo docker cp master:/etc/rancher/k3s/k3s.yaml k3s.yaml
  export KUBECONFIG=$(pwd)/k3s.yaml
  kubectl cluster-info

  Kubernetes master is running at https://localhost:6443
  CoreDNS is running at https://localhost:6443/api/v1/namespaces/kube-system/services/kube-dns:dns/proxy

  To further debug and diagnose cluster problems, use 'kubectl cluster-info dump'.

To connect to the cluster from another machine, copy the ``k3s.yaml`` file and replace ``localhost`` by ``140.252.32.142``.

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

Deploy the DM-EFD
=================

Once the cluster is ready, we can deploy the DM-EFD.

Requirements
------------

 - AWS credentials (we save the deployment configuration to an S3 bucket and create names for our services on Route53)
 - TLS/SSL certificates for the ``lsst.codes`` domain (we share certificates via SQuaRE Dropbox account)
 - Deployment configuration for the DM-EFD test environment (we share secrets via SQuaRE 1Password account)

.. note::

  The current mechanism to share secrets and certificates is not ideal. We still need to integrate our DM-EFD deployment with the `Vault service implemented by SQuaRE <https://dmtn-112.lsst.io/>`_.

We automate the deployment of the DM-EFD with `Terraform <https://www.terraform.io/>`_ and `Helm <https://helm.sh/>`_.  `Terragrunt <https://www.gruntwork.io/>`_ is used to manage the different deployment environments (dev, test, and production) while keeping the Terraform modules environment-agnostic. We also use Terragrunt to save the Terraform configuration and the state of a particular deployment remotely.

Install Terragrunt, Terraform, and Helm.

.. code-block:: bash

  git clone https://github.com/lsst-sqre/terragrunt-live-test.git
  cd terragrunt-live-test
  make all
  export PATH="${PWD}/bin:${PATH}"

Install the SSL certificates (this step requires access to the SQuaRE Dropbox account).

.. code-block:: bash

  make tls


Initialize the deployment environment
-------------------------------------

The following commands initialize the deployment environment. (Terragrunt uses an S3 bucket to save the deployment configuration, so this step requires the AWS credentials).

.. code-block:: bash

  export AWS_ACCESS_KEY_ID=""
  export AWS_SECRET_ACCESS_KEY=""

  cd afausti/efd
  make all
  terragrunt init --terragrunt-source-update
  terragrunt init


Deployment configuration
------------------------

The DM-EFD deployment configuration on k3s is defined by a set of Terraform variables listed in the  `terraform-efd-k3s <https://github.com/lsst-sqre/terraform-efd-k3s>`_ repository.

Edit the ``terraform.tfvars`` file with the values obtained from the SQuaRE 1Password account. Search for ``terraform vars (efd test)``.

Finally, deploy the DM-EFD with the following commands:

.. code-block:: bash

  terragrunt plan
  terragrunt apply


Outputs
-------

If everything is correct you should see an output similar to this, indicating the new services deployed:

.. code-block:: bash

  confluent_lb_ips = [140.252.32.142]
  grafana_fqdn = test-grafana-efd.lsst.codes
  influxdb_fqdn = test-influxdb-efd.lsst.codes
  nginx_ingress_ip = 140.252.32.142
  prometheus_fqdn = test-prometheus-efd.lsst.codes


The Kafka cluster can be reached at ``test-efd.lsst.codes:31090``.

Testing the DM-EFD
==================

The DM-EFD deployment can be tested using `kafkacat <https://docs.confluent.io/current/app-development/kafkacat-usage.html>`_  a command line utility implemented with ``librdkafka`` the Apache Kafka C/C++ client library.

Run in producer mode (-P) to produce messages for a test topic:

.. code-block:: bash

  kafkacat -P -b test-efd.lsst.codes:31090 -t test_topic
  Hello EFD!
  ^D

Run in Metadata listing mode (-L) to retrieve metadata from the cluster:

.. code-block:: bash

  kafkacat -L -b test-efd0.lsst.codes:31090

The ``-d`` option enables ``librdkafka`` debugging. For instance, ``-d broker`` can be used to debug connection issues with the cluster:

.. code-block:: bash

  kafkacat -L -b test-efd0.lsst.codes:31090 -d broker


Monitoring
==========

The DM-EFD deployment includes `dashboards for monitoring the k3s cluster and Kafka <https://test-grafana-efd.lsst.codes>`_ instrumented by `Prometheus <https://test-prometheus-efd.lsst.codes>`_ metrics. You can login with your GitHub credentials if you are a member of the ``lsst-sqre`` organization.


Restarting the DM-EFD manually
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

Currently, Kafka writers containers are not managed by Kubernetes and need to be started manually:

.. code-block:: bash

  sudo docker start kafka_writers_kafka_Test_1 kafka_writers_kafka_ATCamera_1 kafka_writers_kafka_ATHeaderService_1  kafka_writers_kafka_ATArchiver_1 kafka_writers_kafka_ATMonochromator_1 kafka_writers_kafka_ATSpectrograph_1 kafka_writers_kafka_Electrometer_1


Using the DM-EFD
================

In this section, we document some procedures that are useful for operating
the DM-EFD. Please refer to `DM-EFD prototype implementation based on Kafka and InfluxDB <https://sqr-029.lsst.io>`_ for an overview of the DM-EFD system.


Initialize a SAL subsystem
--------------------------

The following command initializes a SAL subsystem, deploy the corresponding SAL transform app and configure the InfluxDB Sink Connector to consume the SAL topics of that subsystem. In the example, we initialize the ``ATCamera`` subsystem:

.. code-block:: bash

  helm install --name ATCamera --namespace kafka-efd-apps --set subsystem=ATCamera lsstsqre/kafka-efd-apps

Check a SAL transform app
-------------------------

Inspect the logs of a SAL transform app for a particular subsystem. In this example the ``ATCamera``:

.. code-block:: bash

  kubectl logs $(kubectl get pods --namespace kafka-efd-apps -l "app=saltransform,subsystem=ATCamera" -o jsonpath="{.items[0].metadata.name}") --n kafka-efd-apps


Check the InfluxDB Sink connector
---------------------------------

Inspecting the Kafka connect logs:

.. code-block:: bash

  kubectl logs $(kubectl get pods --namespace kafka -l "app=cp-kafka-connect,release=confluent" -o jsonpath="{.items[0].metadata.name}") cp-kafka-connect-server --namespace kafka -f


Getting data from the DM-EFD
----------------------------

InfluxDB provides an `HTTP API <https://docs.influxdata.com/influxdb/v1.7/tools/api/>`_ for accessing the data. Here we show a Python code snippet to get data from a particular SAL topic from the DM-EFD. In the example, we retrieve the `Temperature for CCD 0 in the last 24h`:

.. code-block:: python3

  import requests

  INFLUXDB_API_URL = "https://test-influxdb-efd.lsst.codes"
  INFLUXDB_DATABASE = "efd"

  def get_topic_data(topic, period="24h"):
    params={'q': 'SELECT * FROM "{}\"."autogen"."{}" where time > now()-{}'.format(INFLUXDB_DATABASE, topic, period)}
    r = requests.post(url=INFLUXDB_API_URL + "/query", params=params)

    return r.json()

  get_topic_data("lsst.sal.ATCamera.ccdTemp0")







.. Add content here.
.. Do not include the document title (it's automatically added from metadata.yaml).

.. .. rubric:: References

.. Make in-text citations with: :cite:`bibkey`.

.. .. bibliography:: local.bib lsstbib/books.bib lsstbib/lsst.bib lsstbib/lsst-dm.bib lsstbib/refs.bib lsstbib/refs_ads.bib
..    :style: lsst_aa

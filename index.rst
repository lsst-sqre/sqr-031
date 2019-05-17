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

The DM-EFD is a solution based on Kafka and InfluxDB for recording the LSST telemetry and events. It was prototyped and `tested with the simulator for the M1M3 subsystem <https://sqr-029.lsst.io/#live-sal-experiment-with-avro-transformations>`_. The next logical step is to deploy and test the DM-EFD it in a more realistic environment. The Tucson test stand is currently testing the Auxilary Telescope Camera (ATCam) and allows us to do this exercise.

The adoption of Docker and Kubernetes as our deployment platform solves the problem of porting the DM-EFD to different environments like the Google Cloud Platform (GCP) for development, the Tucson and the NCSA test stands for testing and the LSST Summit for operations.

In this technote, we demonstrate that we can deploy the DM-EFD on a single machine with Docker and k3s ("kubes"), a lightweight Kubernetes using the same Terraform modules and Helm charts that we use for the GCP deployment.


Deploy k3s ("kubes")
====================

`k3s <https://github.com/rancher/k3s>`_ is a lightweight Kubernetes that we can run in a container.


Requirements
------------

We assume you have a Linux box with `Docker CE <https://docs.docker.com/install/linux/docker-ce/centos/>`_,  `kubectl <https://kubernetes.io/docs/tasks/tools/install-kubectl/#install-kubectl-on-linux>`_ and `Helm <https://helm.sh/docs/using_helm/#installing-helm>`_ installed. We used:

 - Docker CE 18.09.6
 - kubectl 1.14.1
 - Helm client 2.11.0 (The DM-EFD deploys the Helm server 2.11.0, the Helm client and server versions must match.)

.. note::

  We have tested the DM-EFD deployment on k3s using Docker Desktop for Mac, but it `cannot route traffic from the host to the container <https://docs.docker.com/docker-for-mac/networking/>`_ (``--network host`` option). That limits our deployment to Linux.

Start the k3s master
--------------------

Start the k3s server with the following commands:

.. code-block:: bash

  export K3S_PORT=6443
  export K3S_URL=http://localhost:${K3S_PORT}
  export K3S_TOKEN=$(date | base64)
  export HOST_PATH=<local path to store persistent data>
  export CONTAINER_PATH=/opt/local-path-provisioner
  sudo docker run  -d --tmpfs /run --tmpfs /var/run -v ${HOST_PATH}:${CONTAINER_PATH} -e K3S_URL=${K3S_URL} -e K3S_TOKEN=${K3S_TOKEN} --privileged --network host --name master2 docker.io/rancher/k3s:v0.5.0-rc1 server --https-listen-port ${K3S_PORT} --no-deploy traefik

Note that we are not deploying ``traefik`` because the DM-EFD already includes the ``nginx-ingress`` controller.

To connect to the master you need to copy the kubeconfig file from the container:

.. code-block:: bash

  sudo docker cp master:/etc/rancher/k3s/k3s.yaml k3s.yaml


at this point you can access the cluster:

.. code-block:: bash

  export KUBECONFIG=$(pwd)/k3s.yaml
  kubectl cluster-info

  Kubernetes master is running at https://localhost:6443
  CoreDNS is running at https://localhost:6443/api/v1/namespaces/kube-system/services/kube-dns:dns/proxy

  To further debug and diagnose cluster problems, use 'kubectl cluster-info dump'.


Deploy the local-path provisioner
---------------------------------

The `local-path provisioner <https://github.com/rancher/local-path-provisioner>`_ will create ``hostPath`` persistent volumes on the node automatically. The directory ``/opt/local-path-provisioner`` will be used as the path for provisioning. The provisioner will be installed in ``local-path-storage`` namespace by default.


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

If there are more machines you can easily add workers to the cluster. Copy the ``node-token`` from the master:

.. code-block:: bash

  sudo docker cp master:/var/lib/rancher/k3s/server/node-token node-token

and start the worker(s):

.. code-block:: bash

  export SERVER_URL=https://<master external IP>:${K3S_PORT}
  export NODE_TOKEN=$(cat node-token)
  export WORKER=kube-0
  export HOST_PATH=<local path to store persistent data>
  export CONTAINER_PATH=/opt/local-path-provisioner
  sudo docker run -d --tmpfs /run --tmpfs /var/run -v ${HOST_PATH}:${CONTAINER_PATH} -e K3S_URL=${SERVER_URL} -e K3S_TOKEN=${NODE_TOKEN} --privileged --name ${WORKER} rancher/k3s:v0.5.0-rc1

.. note::

	By default ``/opt/local-path-provisioner`` will be used across all the nodes to store persistent volume data. See the `local-path provisioner configuration <https://github.com/rancher/local-path-provisioner#configuration>`_ to customize this path on each node.


Deploy the DM-EFD
=================

Once the cluster is ready we can deploy the DM-EFD.

Requirements
------------

Inputs
------


Outputs
-------


Using the DM-EFD
================

Initializing a SAL subsystem
----------------------------

Checking Kafka
--------------

Checking the InfluxDB Sink connector
------------------------------------

Checking influxDB
-----------------

Visualizing SAL topics with Chronograf
--------------------------------------

Getting data from the DM-EFD
----------------------------





.. Add content here.
.. Do not include the document title (it's automatically added from metadata.yaml).

.. .. rubric:: References

.. Make in-text citations with: :cite:`bibkey`.

.. .. bibliography:: local.bib lsstbib/books.bib lsstbib/lsst.bib lsstbib/lsst-dm.bib lsstbib/refs.bib lsstbib/refs_ads.bib
..    :style: lsst_aa

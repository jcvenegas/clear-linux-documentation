  .. _kubernetes:

Install Kubernetes\*
#########################

This tutorial describes how to install, configure and run Kubernetes\* on
|CLOSIA| using Kubernetes' runtime interface *cri-o* and Docker's
runtime *runc*.
Kubernetes\* is an open source system for automating deployment,
scaling, and management of containerized applications.
It groups containers that make up an application into logical units for easy
management and discovery.

Prerequisites
*************

This tutorial assumes you have installed |CL| on your host system.
For detailed instructions on installing |CL| on a bare metal system, follow
the :ref:`bare metal installation tutorial<bare-metal-install>`.

Before you install any new packages, update |CL| with the following command:

   .. code-block:: bash

      sudo swupd update

Install Kubernetes
***********************

Kubernetes is included in the `cloud-native-basic <https://github.com/clearlinux/clr-bundles/blob/master/bundles/cloud-native-basic>`_ bundle. To install the
framework, enter the following command:

   .. code-block:: bash

      sudo swupd bundle-add cloud-native-basic

Configure Kubernetes
***********************

We will use the basic default Kubernetes configuration for this tutorial
examples. Note, however, that you must define a Kubernetes configuration
accordingly both to the specifics of the deployment you would implement and to
your security needs.

#. Create (or edit if it exists) the :file:`/etc/systemd/system/docker.service.d/50-runtime.conf`
   file to set your desired runtime. For the purpose of this tutorial, we set
   *runc* as default runtime for docker:

    .. code-block:: bash

      [Service]
      Environment="DOCKER_DEFAULT_RUNTIME=--default-runtime runc"

#. Enable IP forwarding:

   Create (or edit if it exists) the :file:`/etc/sysctl.conf` to include the
   following line:

    .. code-block:: bash

      net.ipv4.ip_forward = 1

   apply the change:

   .. code-block:: bash

      sudo sysctl -p
      sudo iptables -P FORWARD ACCEPT

   .. note::

      IP forwarding should only be done in systems with appropriate firewall
      settings in place.

#. Enable kubelet service:

    .. code-block:: bash

      sudo systemctl enable kubelet.service

#. Disable swap, either:

   a) Temporarily (swap will be enabled at next reboot, causing failures in
      your cluster):

      .. code-block:: bash

         sudo swapoff -a

   or:

   b) Permanently:

      Get the Device corresponding to *Linux swap*:

      .. code-block:: bash

         sudo fdisk -l

      Mask the swap partition:

      .. code-block:: bash

         sudo systemctl mask dev-sdXX.swap

      where `dev-sdXX` corresponds to the Device with Type *Linux swap* listed
      from previous command.

      .. note::

         Swap disabling is a temporary requirement since Kubernetes support for
         swap enabled at the host is `planned for the future <https://github.com/kubernetes/kubernetes/issues/53533>`_.
         On limited-resources systems, some performance degradation may be observed
         while swap is disabled.

Customization requirements (optional)
*************************************

If your plan is to customize the default kubernetes configuration, you must
perform the following steps. You can skip this section if your plan is to use
the default settings.

#. Create administrator-specific configuration directories with the commands:

   .. code-block:: bash

      sudo mkdir -p /etc/systemd/system/kubelet.service.d
      sudo mkdir -p /etc/crio

#. Copy and rename the default templates from the distribution-provided
   directories to the administrator-specific configuration directories with the
   commands:

   .. code-block:: bash

      sudo cp /usr/lib/systemd/system/kubelet.service /etc/systemd/system/kubelet.service.d/10-kubeadm.conf
      sudo cp /usr/share/defaults/crio/* /etc/crio/

   .. note:: You must implement any customizations using the files you've
      just created under the :file::`/etc/` directory. Since |CL| is a
      stateless system, you should never modify the files under the :file:`/usr/`
      directory. The software updater overwrites those files.

Proxy configuration (optional)
******************************

If you use a proxy server, you must also set your proxy environment variables
and create an appropriate proxy configuration file for each one of the services:
*kubelet*, *crio* and *docker*.
Consult your IT department if you are behind a corporate proxy for the
appropriate values, but ensure that your local IP IS EXPLICITLY INCLUDED in the
variable *NO_PROXY* (*localhost* is not enough).

If you have already set your proxy environment variables, you can run the
following commands as a shell script to configure all of these services in
one step:

   .. code-block:: bash

      services=('kubelet' 'crio' 'docker')
      for s in "${services[@]}"; do
      sudo mkdir -p "/etc/systemd/system/${s}.service.d/"
      cat << EOF | sudo tee "/etc/systemd/system/${s}.service.d/proxy.conf"
      [Service]
      Environment="HTTP_PROXY=${http_proxy}"
      Environment="HTTPS_PROXY=${https_proxy}"
      Environment="SOCKS_PROXY=${socks_proxy}"
      Environment="NO_PROXY=${no_proxy}"
      EOF
      done


Run Kubernetes for the first time
**********************************

#. Prepare your system to run kubernetes for the first time with the
   following commands:

   .. code-block:: bash

      yes | sudo kubeadm reset --cri-socket=/run/crio/crio.sock
      sudo systemctl daemon-reload
      sudo systemctl restart crio.service
      sudo systemctl restart docker
   
#. You can initialize the master control plane with the following command:

   .. code-block:: bash

      sudo -E kubeadm init --pod-network-cidr 10.244.0.0/16 --cri-socket=/run/crio/crio.sock

   You must save the *IP*, *token* and *hash* values that will be displayed once the
   cluster is initialized.

**Congratulations!**

You've successfully installed and set up Kubernetes in |CLOSIA| using *crio*
and *runc*. You are now ready to follow on-screen instructions to deploy a pod
network to the cluster and join worker nodes with the displayed token and IP
information.

Kubernetes examples
###################

The following examples are for demonstration purposes only and are NOT SUITED
FOR PRODUCTION ENVIRONMENTS due to security concerns. You must evaluate and
implement the appropriate configuration for your deployment needs.

Example 1: Deploy Kubernetes with 2 nodes (master and a minion)
***************************************************************

#. Install and Setup Kubernetes at the master machine (Follow instructions at
   top section in this tutorial).

#. Install and Setup Kubernetes at the minion machine. DO NOT initialize a
   control plane in it.

At the master machine:

#. Initialize the master control plane with the *kubeadm init* command and save
   the *token* and *hash* that will be displayed next to the *kubeadm join*
   command in the following format:
  
   .. code-block:: bash

      kubeadm join <master-ip>:<master-port> --token <token> --discovery-token-ca-cert-hash <hash>

#. Create a user configuration file and directory with the commands:

   .. code-block:: bash

      mkdir -p $HOME/.kube
      sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
      sudo chown $(id -u):$(id -g) $HOME/.kube/config

#. Install flannel as pod network with the command:

   .. code-block:: bash

      kubectl apply -f https://github.com/coreos/flannel/raw/master/Documentation/kube-flannel.yml

You can verify that the installation was successfull with the command:

   .. code-block:: bash

      kubectl get pods --all-namespaces

and get at the output the new *kube-flannel-...* pod among others.

At the minion machine:

#. Run the following command to join the minion to the master machine with the
   values displayed when the master control plane was initialized:

   .. code-block:: bash

      kubeadm join <master-ip>:<master-port> --token <token> --discovery-token-ca-cert-hash <hash> --cri-socket=/run/crio/crio.sock

   .. note::

      If your master and minion are behind a proxy, you may need to add the master's IP address to the minion's *NO_PROXY* environment variable before joining the minion.

You can check at the master machine that the node was added with the command:

   .. code-block:: bash

      kubectl get nodes

**Congratulations!**
You've just deployed a Kubernets cluster with 2 nodes.

Example 2: Install Kubernetes Dashboard in master machine
*********************************************************
https://github.com/kubernetes/dashboard

At master machine:

#. Install the Kubernetes dashboard to be exposed over http only with the command:

   .. code-block:: bash

      kubectl apply -f https://raw.githubusercontent.com/kubernetes/dashboard/master/src/deploy/alternative/kubernetes-dashboard.yaml

You can verify that a new `kubernetes-dashboard-...` pod is running with the command:

   .. code-block:: bash

      kubectl get pods --all-namespaces

#. To make the dashboard available from outside the cluster, run the command:

   .. code-block:: bash

      kubectl -n kube-system edit service kubernetes-dashboard

and change *type: ClusterIP* to *type: NodePort*. Then save the file.

#. Create the :file:`/kube-dashboard-access.yaml` file to grant dashboard the
   cluster-admon role with the following content.
   WARNING: this is not suitable for production:

   .. code-block:: bash

      apiVersion: rbac.authorization.k8s.io/v1beta1
      kind: ClusterRoleBinding
      metadata:
         name: kubernetes-dashboard
         labels:
            k8s-app: kubernetes-dashboard
      roleRef:
         apiGroup: rbac.authorization.k8s.io
         kind: ClusterRole
         name: cluster-admin
      subjects:
      - kind: ServiceAccount
         name: kubernetes-dashboard
         namespace: kube-system

#. Apply the change:

   .. code-block:: bash

      kubectl create -f kube-dashboard-access.yaml

#. Get the port on which dashboard is running with the command:

   .. code-block:: bash

      kubectl -n kube-system get service kubernetes-dashboard

for example, from a displayed *PORT(S)* field with value *80:32116/TCP*, we know
the port is *32116*.

#. Access the dashboard from web using the master's IP and the port from previous step:

   .. code-block:: bash

      http://<node-ip>:<nodePort>

**Congratulations!**

You've successfully installed and configured the Kubernetes Dashboard.

Troubleshooting
###############
* <HOSTNAME> not found in <IP> message. Your DNS server may not be appropriately
  configured. You can try adding an entry to the :file:`/etc/hosts` with your
  host's IP and Name (use the commands *hostname* and *hostname -I* to retrieve
  them).
  For example:
  100.200.50.20   myhost

* Images cannot be pulled. You may be behind a proxy server. Try configuring
  your proxy settings, using the variables *HTTP_PROXY*, *HTTPS_PROXY* and
  *NO_PROXY* as required in your environment.

* Connection refused error. If you are behind a proxy server, you may need to
  add the master's ip to the variable *NO_PROXY*

.. _Kubernetes: https://kubernetes.io/

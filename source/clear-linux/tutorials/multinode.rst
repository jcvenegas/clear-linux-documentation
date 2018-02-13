.. _multinode

Set up a multinode cluster with Hadoop\* and Spark\*
##################################################

This tutorial walks you through the process of configuring a basic multinode cluster using Apache\* Hadoop and Apache\* Spark on |CLOSIA|, and then shows you how to run applications that can benefit from this highly fault-tolerant file system setup.

Prerequisites
*************

Before following this tutorial, you should follow the :ref:`bare-metal-install` to ensure you have installed |CLOSIA| in all of the machines in your cluster.

Before you install any new packages, update |CL| with the following command in all of the machines in the cluster:

.. code-block:: bash

   $ sudo swupd update

For the purposes of this tutorial, we will install both Hadoop and Spark in two machines that will run the master and slave daemons in the setup.

Install and Configure Hadoop and Spark
***********************************

#. Follow the :ref:`hadoop` tutorial up to the *Run the Hadoop daemons* section in all of the machines in the cluster.
#. Follow the :ref:`spark` tutorial up to the *Start the master server and a worker* section in all of the machines in the cluster.
#. Shutdown each single node with the following commands in each of the machines in the cluster:

   .. code-block:: bash

      $ sudo stop-yarn.sh

      $ sudo stop-dfs.sh

   All of the machines in the cluster must be able to reach each other over the network. Ensure ssh service is active with the following command in all of the machines:

   .. code-block:: bash

      $ sudo systemctl status sshd.service

#. Open (or create if not existing) the *hosts* file using the editor of your choice and update/add the ip address and role of all of the machines in the cluster:

   .. code-block:: xml

        ..
        10.20.30.01     master
        10.20.30.01     slave
        ..

   .. note:: Hadoop's user in *master* machine must be able to connect through ssh to all of the *slave* machines. In this tutorial we are using *root* user to run the applications and connect to remote machines, but this is not recommended for production instances and you must evaluate what the most appropriate setup for your cluster should be.

#. Create the *masters* file using the editor of your choice and add the master's host name in one line (this file is only required at the master machine). Optionally, you can execute the following command:

   .. code-block:: bash

      $ echo master | sudo tee /etc/hadoop/masters

#. Create the *slaves* file using the editor of your choice and add the host names list, one machine name per line, where the Hadoop slaves will run (this file is only required at the master machine). In this tutorial we use use *master* and *slave* as names for clarity, but you can use whatever naming that makes sense for your setup. Optionally, you can execute the following command:

   .. code-block:: bash

      $ printf "master\nslave" | sudo tee /etc/hadoop/slaves


   .. note:: In this tutorial we are including the master machine as slave, since we want it to run both master and slave daemons. You should match what makes sense for your setup in the slave host names at the *slaves* file and the same applies to what you entered in the *hosts* file in step 4.

#. Open the *core-site.xml* file using the editor of your choice and update the following property (repeat this step in ALL of the cluster machines):

   .. code-block:: xml

        ..
        <property>
            <name>fs.default.name</name>
            <value>hdfs://master:8020</value>
        </property>
        ..

#. Open the *hdfs-site.xml* file using the editor of your choice and update the *dfs.replication* property to the number of slave nodes in your setup. In this tutorial we have 2 slaves:

   .. code-block:: xml

        ..
        <property>
            <name>dfs.replication</name>
            <value>2</value>
        </property>
        ..

#. Format the HDFS file system:

   .. code-block:: bash

      $ sudo hdfs namenode -format

   .. note:: Do not format a running cluster because this will erase all existing data in the HDFS file system.

#. Start the HDFS daemons on master machine:

   .. code-block:: bash

      $ sudo start-dfs.sh

   Output of this command includes the loggin path at the slave machines, so you can verify there that the cluster is working as expected. For example:

   .. code-block:: xml

      "slave: starting datanode, logging to /var/log/hadoop/hadoop-root-datanode-clr-machine.out"

#. Open the *spark-env.sh* file using the editor of your choice and update the *SPARK_MASTER_HOST* variable with the master's IP entered in Step 4 and export the Hadoop configuration directory path:

   .. code-block:: xml

        ..
        SPARK_MASTER_HOST="10.20.30.01"
        export HADOOP_CONF_DIR=/etc/hadoop
        export YARN_CONF_DIR=/etc/hadoop
        ..

#. Open the *spark-defaults.conf* file using the editor of your choice and update the *spark.master* variable:

   .. code-block:: xml

        ..
        spark.master    yarn
        ..
            
#. Launch a spark application in cluster mode. For example, use the following command to launch the *SparkPi* example application:

   .. code-block:: bash

      $ sudo spark-submit --class org.apache.spark.examples.SparkPi --master yarn \
        --deploy-mode cluster --driver-memory 2g --executor-memory 1g --executor-cores 1 \
        --queue default /usr/share/apache-spark/examples/jars/spark-examples_2.11-2.2.1.jar 10

#. You can view at the console output what is the tracking URL where you can view the application results. Optionally, you can click the link to the application overview at the Hadoop web GUI, for example:

   .. code-block:: xml

      "http://10.20.30.01:8088/cluster/apps"
    
Troubleshooting
***************

DataNode not showing either on master or on slaves:
    - Stop both dfs and yarn daemons in master machine using:

    .. code-block:: bash

       $ sudo stop-dfs.sh

       $ sudo stop-yarn.sh

    - Remove the hadoop application temp directory (in all of the affected machine(s)):

    .. code-block:: bash

       $ sudo rm -Rf /tmp/hadoop-root/

    - Re-format the hdfs (any existing data in the HDFS file system will be removed):

    .. code-block:: bash

       $ sudo hdfs namenode -format

    - Start the dfs and yarn daemons again.

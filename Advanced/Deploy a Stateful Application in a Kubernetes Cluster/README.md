Description

Stateful applications are applications with memory of the past. Kubernetes has achieved wide adoption for deploying stateless applications. Kubernetes also provides a rich set of features for running stateful applications. Adoption of Kubernetes for stateful applications is growing and so is the list of stateful workloads that are supported by Kubernetes. In this lab, you will deploy a stateful application in a Kubernetes cluster to understand how Kubernetes supports stateful applications.
Learning Objectives

Upon completion of this lab you will be able to:

    Describe methods for deploying Kubernetes clusters in the cloud
    Explain how Kubernetes can deploy stateful applications
    Determine if it is a good idea to move a stateful application into Kubernetes
    Use Kubernetes to automatically provision persistent cloud storage volumes
    Expose Kubernetes applications via cloud load balancers
    Monitor and manage Kubernetes clusters using the Kubernetes Dashboard

Prerequisites

You should be familiar with:

    Basic Kubernetes concepts such as pods, services, and deployments
    AWS EBS volumes
    Working on the command line in Linux 


![alt text](image.png)

Step 1: Connecting to the Kubernetes Cluster
Introduction

You will use a multi-node Kubernetes cluster to explore stateful applications in this lab.
This lab provides you with a Kubernetes cluster initialized with kubeadm, opens in a new tab and running on Ubuntu. The cluster is similar to the clusters used in official CNCF Kubernetes certification exams. It has a bastion node for connecting to the Kubernetes cluster nodes that include a single control-plane node. The infrastructure is deployed on AWS. You can log in to AWS using the lab credentials and Open button in the Cloud environment panel when needed.

The following instructions connect to the cluster using a web browser terminal. You can also connect using your preferred SSH client rather than the browser terminal by using the PPK (Windows) or PEM (Mac/Linux) key files in the Credentials section of this lab to connect to the bastion host's IP address which appears below the credentials once it is available.

Instructions

1. Navigate to the lab-provided browser terminal.

2. At the login prompt enter the following credentials:

    login: Enter ca
    Password: Press enter (no password)

You are then presented with the shell prompt:

3. Wait for the lab environment to fully provision:

4. From the Cloud environment panel, copy the Cluster SSH command that appears at the bottom of the lab Credentials section:

5. In the browser terminal, enter the Cluster SSH command at the prompt: 

ssh ubuntu@44.250.178.191 -oStrictHostKeyChecking=no

You are now connected to the bastion node in the cluster:

6. To ensure the cluster has one available worker node (a node with ROLES set to <none> in addition to the control-plane node), before proceeding to use the cluster, enter the following command to watch the cluster come up and press ctrl+c to stop watching: 

```
watch kubectl get nodes
```

It usually takes
2 minutes
from the time you start the lab to see a worker node join the cluster. You can proceed to the next step before seeing the worker node, but one worker node should be available before you start working with the cluster.

Note: You may see The connection to the server 10.0.0.100:6443 was refused - did you specify the right host or port? while the control plane's API server prepares to accept requests. It is not a problem and the control-plane node will appear before long.

Step 2: Inspecting the Kubernetes Cluster


Introduction

In the previous lab step, you connected to a bastion host using SSH. In this lab step, you will verify that the Kubernetes cluster is ready to go.

Instructions

1. List all of the nodes in the cluster:

```
kubectl get nodes
```

Observe that the control-plane node's private IP is included in one of the internal DNS names in the list of nodes. The other nodes are a part of the Auto Scaling group.

2. Enter the following command to describe the control plane node:

```
kubectl describe nodes -l node-role.kubernetes.io/control-plane | more
```

The output is piped into the more command to be displayed one page at a time by pressing the space bar.

The describe command uses the node-role.kubernetes.io/control-plane label to select only nodes that have that label present, of which there is only one. In the output, you will see a Taints section with a value of node-role.kubernetes.io/control-plane:NoSchedule. Taints cause the node to repel pods unless the pod declares a toleration of the taint. For the NoSchedule case, this means no pods will be assigned to the control plane node unless you declare a toleration of node-role.kubernetes.io/control-plane when creating the pod. 

3. Type the following incomplete command and press tab twice to get the possible completions:

```
kubectl get namespace
```

The default namespace is used when you schedule pods without declaring a specific namespace. The kube-system namespace is where pods essential to the operation of the Kubernetes cluster live. kubernetes-dashboard contains the resources for a monitoring and graphical interface for managing the cluster. You will use the dashboard in a later lab step.

4. List all of the pods in the kube-system namespace:

```
kubectl get pods --namespace=kube-system
```

All of the components of the cluster are running in pods in the kube-system namespace:

    aws-cloud-controller: Provides interaction between the cluster and AWS service APIs and allows a cluster to leverage AWS resources.
    calico: The container network used to connect each node to every other node in the cluster. Calico also supports network policy. Calico is one of many possible container networks that can be used by Kubernetes.
    coredns: Provides DNS services to nodes in the cluster
    etcd: The primary data store of all cluster state
    kube-apiserver: The REST API server for managing the Kubernetes cluster
    kube-controller-manager: Manager of all of the controllers in the cluster that monitor and change the cluster state when necessary
    kube-proxy: Network proxy that runs on each node
    kube-scheduler: Control plane process which assigns Pods to Nodes
    metrics-server: Not an essential component of a Kubernetes cluster but it is used in this lab to provide metrics for viewing in the Kubernetes dashboard.
    ebs-csi: Not an essential component of a Kubernetes cluster but is used to manage the lifecycle of Amazon EBS volumes for persistent volumes.

Step 3: Deploying a Stateful Application in the Kubernetes Cluster

Introduction

Stateful applications are applications that have a memory of what happened in the past. Databases are an example of stateful applications. Kubernetes provides support for stateful applications through StatefulSets and related primitives.

This lab will deploy a replicated MySQL database as a StatefulSet. MySQL is one of the most popular databases in the world. This lab won't focus on the details specific to configuring MySQL. The focus is on the features of Kubernetes that allow a stateful application to be deployed. The Kubernetes community maintains a wide variety of stateful applications that are ready to deploy through Helm, opens in a new tab. Helm acts as a package manager for Kubernetes and can deploy entire applications to a cluster using templates called charts. A list of charts is available here, opens in a new tab. In addition to MySQL, there are charts for many other popular databases including MongoDB and PostgreSQL, as well as popular applications like WordPress and Joomla. The concepts illustrated in this lab will help you understand the common elements used to deploy all of these stateful applications in Kubernetes.

Managing your stateless and stateful applications with Kubernetes provides efficiencies and simplifies automation. However, before using StatefulSets for your own stateful applications, you should consider if any of the following apply:

    You embrace microservices
    You frequently create new service footprints that include stateful applications
    Your current solution for storing state can't scale to meet predicted demand
    Your stateful applications can meet performance requirements without using specialized hardware and could effectively run on the same hardware used for stateless applications
    You value flexible reallocation of resources, consolidation, and automation over squeezing the most and having highly predictable performance

If any of the previous bullets apply to your situation, it may make sense to use Kubernetes for your stateful applications.

There are quite a few concepts at work in this lab step. Begin by reviewing the following background section and then jump into the instructions.

Background

ConfigMaps: A type of Kubernetes resource that is used to decouple configuration artifacts from image content to keep containerized applications portable. The configuration data is stored as key-value pairs.

Headless Service: A headless service is a Kubernetes service resource that won't load balance behind a single service IP. Instead, a headless service returns a list of DNS records that point directly to the pods that back the service. A headless service is defined by declaring the clusterIP property in a service spec and setting the value to None. StatefulSets currently require a headless service to identify pods in the cluster network.

Stateful Sets: Similar to Deployments in Kubernetes, StatefulSets manage the deployment and scaling of pods given a container spec.StatefulSets differ from Deployments in that the Pods in a stateful set are not interchangeable. Each pod in a StatefulSet has a persistent identifier that it maintains across any rescheduling. The pods in a StatefulSet are also ordered. This provides a guarantee that one pod can be created before following pods. In this lab, this is useful for ensuring the MySQL primary is provisioned first.

PersistentVolumes (PVs) and PersistentVolumeClaims (PVCs): PVs are Kubernetes resources that represent storage in the cluster. Unlike regular Volumes which exist only until while containing pod exists, PVs do not have a lifetime connected to a pod. Thus, they can be used by multiple pods over time, or even at the same time. Different types of storage can be used by PVs including NFS, iSCSI, and cloud-provided storage volumes, such as AWS EBS volumes. Pods claim PV resources through PVCs.

MySQL replication: This lab uses a single primary, asynchronous replication scheme for MySQL. All database writes are handled by a single primary. The database replicas asynchronously synchronize with the primary. This means the primary will not wait for the data to be copied onto the replicas. This can improve the performance of the primary at the expense of having replicas that are not always exact copies of the primary. Many applications can tolerate slight differences in the data and are able to improve the performance of database read workloads by allowing clients to read from the replicas.

Instructions

1. In your SSH shell, enter the following to declare a ConfigMap to allow primary MySQL pods to be configured differently than replica pods:


```
cat <<EOF > mysql-configmap.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: mysql
  labels:
    app: mysql
data:
  master.cnf: |
   # Apply this config only on the primary.
   [mysqld]
   log-bin
  slave.cnf: |
    # Apply this config only on replicas.
    [mysqld]
    super-read-only
EOF
```

This ConfigMap will be referenced later in the StatefulSet declaration. The master.cnf key maps to a value that declares a MySQL configuration which includes replication logs. The slave.cnf key maps to a MySQL configuration that enforces read-only behavior.

2. Create the ConfigMap resource:

```
kubectl create -f mysql-configmap.yaml
```

3. Enter the following command to declare the services for the MySQL application:

Two services are defined:

    A headless service (clusterIP: None) for pod DNS resolution. Because the service is named mysql, pods are accessible via pod-name.mysql.
    A service name mysql-read to connect to for database reads. This service uses the default ServiceType of ClusterIP which assigns an internal IP address that load balances request to all the pods labeled with app: mysql.

Database writes need to be sent to the primary. The primary is the first pod provisioned in the StatefulSet and assigned a name mysql-0. The pod is thus accessed by the DNS entry in the headless service for mysql-0.mysql.

4. Create the MySQL services:

kubectl create -f mysql-services.yaml

5. Enter the following command to declare a default storage class that will be used to dynamically provision general-purpose (gp2) EBS volumes for the Kubernetes PVs:


```
cat <<EOF > mysql-storageclass.yaml
kind: StorageClass
apiVersion: storage.k8s.io/v1
metadata:
  name: general
provisioner: kubernetes.io/aws-ebs
parameters:
  type: gp2
EOF
```

The built-in aws-ebs storage provision is specified along with the type gp2.

6. Create the storage class:

kubectl create -f mysql-storageclass.yaml

7. Enter the following command to declare the MySQL StatefulSet:

```
cat <<'EOF' > mysql-statefulset.yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: mysql
spec:
  selector:
    matchLabels:
      app: mysql
  serviceName: mysql
  replicas: 3
  template:
    metadata:
      labels:
        app: mysql
    spec:
      initContainers:
      - name: init-mysql
        image: mysql:5.7.35
        command:
        - bash
        - "-c"
        - |
          set -ex
          # Generate mysql server-id from pod ordinal index.
          [[ `hostname` =~ -([0-9]+)$ ]] || exit 1
          ordinal=${BASH_REMATCH[1]}
          echo [mysqld] > /mnt/conf.d/server-id.cnf
          # Add an offset to avoid reserved server-id=0 value.
          echo server-id=$((100 + $ordinal)) >> /mnt/conf.d/server-id.cnf
          # Copy appropriate conf.d files from config-map to emptyDir.
          if [[ $ordinal -eq 0 ]]; then
            cp /mnt/config-map/master.cnf /mnt/conf.d/
          else
            cp /mnt/config-map/slave.cnf /mnt/conf.d/
          fi
        volumeMounts:
        - name: conf
          mountPath: /mnt/conf.d
        - name: config-map
          mountPath: /mnt/config-map
      - name: clone-mysql
        image: gcr.io/google-samples/xtrabackup:1.0
        command:
        - bash
        - "-c"
        - |
          set -ex
          # Skip the clone if data already exists.
          [[ -d /var/lib/mysql/mysql ]] && exit 0
          # Skip the clone on primary (ordinal index 0).
          [[ `hostname` =~ -([0-9]+)$ ]] || exit 1
          ordinal=${BASH_REMATCH[1]}
          [[ $ordinal -eq 0 ]] && exit 0
          # Clone data from previous peer.
          ncat --recv-only mysql-$(($ordinal-1)).mysql 3307 | xbstream -x -C /var/lib/mysql
          # Prepare the backup.
          xtrabackup --prepare --target-dir=/var/lib/mysql
        volumeMounts:
        - name: data
          mountPath: /var/lib/mysql
          subPath: mysql
        - name: conf
          mountPath: /etc/mysql/conf.d
      containers:
      - name: mysql
        image: mysql:5.7
        env:
        - name: MYSQL_ALLOW_EMPTY_PASSWORD
          value: "1"
        ports:
        - name: mysql
          containerPort: 3306
        volumeMounts:
        - name: data
          mountPath: /var/lib/mysql
          subPath: mysql
        - name: conf
          mountPath: /etc/mysql/conf.d
        resources:
          requests:
            cpu: 100m
            memory: 200Mi
        livenessProbe:
          exec:
            command: ["mysqladmin", "ping"]
          initialDelaySeconds: 30
          timeoutSeconds: 5
        readinessProbe:
          exec:
            # Check we can execute queries over TCP (skip-networking is off).
            command: ["mysql", "-h", "127.0.0.1", "-e", "SELECT 1"]
          initialDelaySeconds: 5
          timeoutSeconds: 1
      - name: xtrabackup
        image: gcr.io/google-samples/xtrabackup:1.0
        ports:
        - name: xtrabackup
          containerPort: 3307
        command:
        - bash
        - "-c"
        - |
          set -ex
          cd /var/lib/mysql
          # Determine binlog position of cloned data, if any.
          if [[ -f xtrabackup_slave_info ]]; then
            # XtraBackup already generated a partial "CHANGE MASTER TO" query
            # because we're cloning from an existing replica.
            mv xtrabackup_slave_info change_master_to.sql.in
            # Ignore xtrabackup_binlog_info in this case (it's useless).
            rm -f xtrabackup_binlog_info
          elif [[ -f xtrabackup_binlog_info ]]; then
            # We're cloning directly from primary. Parse binlog position.
            [[ `cat xtrabackup_binlog_info` =~ ^(.*?)[[:space:]]+(.*?)$ ]] || exit 1
            rm xtrabackup_binlog_info
            echo "CHANGE MASTER TO MASTER_LOG_FILE='${BASH_REMATCH[1]}',\
                  MASTER_LOG_POS=${BASH_REMATCH[2]}" > change_master_to.sql.in
          fi
          # Check if we need to complete a clone by starting replication.
          if [[ -f change_master_to.sql.in ]]; then
            echo "Waiting for mysqld to be ready (accepting connections)"
            until mysql -h 127.0.0.1 -e "SELECT 1"; do sleep 1; done
            echo "Initializing replication from clone position"
            # In case of container restart, attempt this at-most-once.
            mv change_master_to.sql.in change_master_to.sql.orig
            mysql -h 127.0.0.1 <<EOF
          $(<change_master_to.sql.orig),
            MASTER_HOST='mysql-0.mysql',
            MASTER_USER='root',
            MASTER_PASSWORD='',
            MASTER_CONNECT_RETRY=10;
          START SLAVE;
          EOF
          fi
          # Start a server to send backups when requested by peers.
          exec ncat --listen --keep-open --send-only --max-conns=1 3307 -c \
            "xtrabackup --backup --slave-info --stream=xbstream --host=127.0.0.1 --user=root"
        volumeMounts:
        - name: data
          mountPath: /var/lib/mysql
          subPath: mysql
        - name: conf
          mountPath: /etc/mysql/conf.d
        resources:
          requests:
            cpu: 100m
            memory: 50Mi
      volumes:
      - name: conf
        emptyDir: {}
      - name: config-map
        configMap:
          name: mysql
  volumeClaimTemplates:
  - metadata:
      name: data
    spec:
      accessModes: ["ReadWriteOnce"]
      resources:
        requests:
          storage: 2Gi
      storageClassName: general
EOF
```

There is a lot going on in the StatefulSet. Don't focus too much on the bash scripts that are performing MySQL-specific tasks. Some highlights to focus on, following the order they appear in the file are:

    init-containers: Run to completion before any containers in the Pod spec
        init-mysql: Assigns a unique MySQL server ID starting from 100 for the first pod and incrementing by one, as well as copying the appropriate configuration file from the config-map. Note the config-map is mounted via the VolumeMounts section. The ID and appropriate configuration file are persisted on the conf volume.
        clone-mysql: For pods after the primary, clone the database files from the preceding pod. The xtrabackup tool performs the file cloning and persists the data on the data volume.
    spec.containers: Two containers in the pod
        mysql: Runs the MySQL daemon and mounts the configuration in the conf volume and the data in the data volume
        xtrabackup: A sidecar container that provides additional functionality to the mysql container. It starts a server to allow data cloning and begins replication on replicas using the cloned data files.
    spec.volumes: conf and config-map volumes are stored on the node's local disk. They are easily re-generated if a failure occurs and don't require PVs.
    volumeClaimTemplates: A template for each pod to create a PVC with. ReadWriteOnceaccessMode allows the PV to be mounted by only one node at a time in read/write mode. The storageClassName references the AWS EBS gp2 storage class named general that you created earlier. 

8. Create the StatefulSet and start watching the associated pods:

kubectl create -f mysql-statefulset.yaml
kubectl get pods -l app=mysql --watch

The --watch option causes any updates to the pods to be written to the output. It takes a few minutes to initialize all three replicas.

9. In order to view the logs in S3, open the AWS Management Console by clicking the following button to access the lab’s cloud environment:

10. Enter the following credentials created just for your lab session, and click Sign In:

    Account ID or alias: Keep the pre-populated value
    IAM user name: student
    Password: KvloVHB_DWk61P

11. Navigate to Elastic Block Store > Volumes in the AWS EC2 Console, opens in a new tab.

The 2GiB PVs are listed here as each pod is created. Notice the Tags which relay information about the PV and associated PVC.

12. Return to your SSH shell and press ctrl+C to stop the watch when you see both containers (2/2) in the mysql-2 pod running: 

13. Describe the PVs and PVCs:

kubectl describe pv
kubectl describe pvc

The PV descriptions include the AWS VolumeIDs, file system types (FSType), and associated PVC (Claim). The PVC description includes whether the PVC is currently Bound to a pod.

14. Get the StatefulSet to confirm the current number of replicas matches the desired:

kubectl get statefulset

Summary 

In this lab step, you created several Kubernetes cluster resources to deploy the MySQL database as an example stateful application:

    A ConfigMap for decoupling primary and replica configuration from the containers
    Two Services: one headless service to manage network identity of pods in the StatefulSet, and one to load balance read access to the MySQL replicas
    A StorageClass to provision EBS PVs dynamically
    A StatefulSet that declared two init-containers, two containers, and one PVC template

You observed the ordered sequence of pods being initialized and the PVs created in AWS to facilitate the StatefulSet.

Step 4: Working with the Stateful Application

ntroduction

With MySQL up and running, you will execute some commands against the database to demonstrate everything is working correctly and Kubernetes can gracefully handle failures and other requests.

Instructions

1. Run a temporary container to use mysql to connect to the primary at mysql-0.mysql and run a few SQL commands:

kubectl run mysql-client --image=mysql:5.7 -i -t --rm --restart=Never --\
  /usr/bin/mysql -h mysql-0.mysql -e "CREATE DATABASE mydb; CREATE TABLE mydb.notes (note VARCHAR(250)); INSERT INTO mydb.notes VALUES ('k8s Cloud Academy Lab');"

The SQL commands create a mydb database with a notes table in it with one record.

2. Run a query using the mysql-read endpoint to select all of the notes in the table:

kubectl run mysql-client --image=mysql:5.7 -i -t --rm --restart=Never --\
  /usr/bin/mysql -h mysql-read -e "SELECT * FROM mydb.notes"

3. Run an SQL command that outputs the MySQL server's ID to confirm that the requests are distributed to different pods:

kubectl run mysql-client-loop --image=mysql:5.7 -i -t --rm --restart=Never --\
  bash -ic "while sleep 1; do /usr/bin/mysql -h mysql-read -e 'SELECT @@server_id'; done"

Eventually, a request will be sent to each MySQL server. Recall that the primary has an ID of 100.

4. Press ctrl+c to stop the loop.

5. List the pods including the node name column:

kubectl get pod -o wide

Notice that Kubernetes evenly distributes the pods among the worker nodes as best it can.

6. Enter the following command to simulate taking the node running the mysql-2 pod out of service for maintenance:

node=$(kubectl get pods --field-selector metadata.name=mysql-2 -o=jsonpath='{.items[0].spec.nodeName}')
kubectl drain $node --force --delete-emptydir-data --ignore-daemonsets

where you replace the node environment variable is set to the name of the node running mysql-2 using a field-selector and jsonpath output to select the nodeName of the Pod. It will resemble ip-10-0-#-#.us-west-2.compute-internal. The drain command prevents new pods from being scheduled on the node and then evicts existing pods scheduled to it.

7. Watch the mysql-2 pod get rescheduled to a different node:

kubectl get pod -o wide --watch

8. Uncordon the node you drained so that pods can be scheduled on it again:

kubectl uncordon $node

9. Delete the mysql-2 pod to simulate a node failure and watch it get rescheduled automatically:

kubectl delete pod mysql-2
kubectl get pod mysql-2 -o wide --watch

10. Press ctrl+c to stop watching the pods.

11. Scale the number of replicas up to 5:

kubectl scale --replicas=5 statefulset mysql

12. Watch as new pods get scheduled in the cluster:

kubectl get pods -l app=mysql --watch

13. Press ctrl+c to stop watching the pods.

14. Verify that you see the new MySQL server IDs:

```
kubectl run mysql-client-loop --image=mysql:5.7 -i -t --rm --restart=Never --\
  bash -ic "while sleep 1; do /usr/bin/mysql -h mysql-read -e 'SELECT @@server_id'; done"
```

15. Confirm that the data is replicated in the new mysql-4 pod:

```
kubectl run mysql-client --image=mysql:5.7 -i -t --rm --restart=Never --\
  /usr/bin/mysql -h mysql-4.mysql -e "SELECT * FROM mydb.notes"
```


16. Display the internal virtual IP of the mysql-read endpoint:

```
kubectl get services mysql-read
```

Recall the mysql-read service used the default type of ClusterIP so it is only accessible inside the cluster.

17. Append a load balancer type to the mysql-read service declaration:

echo "  type: LoadBalancer" >> mysql-services.yaml

18. Apply the changes to the mysql-read service:

kubectl apply -f mysql-services.yaml

The apply command can update existing resources. It will create resources if they don't already exist. Because you created the services using the create command instead of apply you will see a warning. The warnings can be ignored for this demonstration.

19. Re-display the mysql-read service status:

kubectl get services mysql-read

Kubernetes is in the process of provisioning an elastic load balancer (ELB) to access the mysql-read service from outside the cluster.

20. After a minute, describe the mysql-read service to find the DNS name of the external load balancer endpoint:

kubectl describe services mysql-read | grep "LoadBalancer Ingress"

21. Use the external load balancer to send some read requests to the cluster:

```
load_balancer=$(kubectl get services mysql-read -o=jsonpath='{.status.loadBalancer.ingress[0].hostname}')
kubectl run mysql-client-loop --image=mysql:5.7 -i -t --rm --restart=Never --\
  bash -ic "while sleep 1; do /usr/bin/mysql -h $load_balancer -e 'SELECT @@server_id'; done"
```

where the load_balancer variable stores the ELB DNS name. It will resemble a830e12d78dcd11e79aba028416f4825-905974806.us-west-2.elb.amazonaws.com. You are now accessing the cluster from the ELB which provides access outside of the cluster.

Note: It can take a minute for the nodes to be added to the load balancer. You will see unknown MySQL host messages until the nodes are added. Let the loop continue to run until you start seeing server IDs being displayed.

22. Press ctrl+c to stop the temporary container.

Summary 

In this lab step, you worked with the MySQL stateful application to demonstrate:

    How to directly address pods in the cluster
    How to access the cluster via the mysql-read service
    How to take nodes out of service
    How Kubernetes automatically recovers from a simulated node failure
    How to permit access to the mysql-read service via an automatically provisioned ELB


Step 5: Monitoring Your Kubernetes Cluster Using Kubernetes Dashboard

ntroduction

Earlier you saw that the Kubernetes Dashboard pod is running in the kube-system namespace. The Kubernetes Dashboard provides a website for monitoring and managing your Kubernetes cluster. In this lab step, you will familiarize yourself with the Kubernetes Dashboard and also demonstrate how to use a port forward to connect to the control plane node. To gain full access to the dashboard you will first bind the cluster-admin role to the dashboard's service account.

Instructions

1. In your SSH shell, enter the following to create a cluster role binding for the kubernetes-dashboard service account:

```
cat << EOF > dashboard-admin.yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: dashboard-admin
  namespace: kubernetes-dashboard
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: dashboard-admin
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: cluster-admin
subjects:
- kind: ServiceAccount
  name: dashboard-admin
  namespace: kubernetes-dashboard
EOF
```

The roleRef reference the cluster-admin role and the subjects to bind the role to is the kubernetes-dashboardServiceAccount.

2. Enter the following to create the cluster role binding:

kubectl create -f dashboard-admin.yaml

3. Enter the following to create a token for the kubernetes dashboard service account which you will need to sign in to the dashboard later and save the token to your notepad:

kubectl -n kubernetes-dashboard create token dashboard-admin

4. Run a Kubernetes proxy server on the bastion in order to proxy requests to the control plane node from outside the private subnet:

kubectl port-forward -n kubernetes-dashboard --address 0.0.0.0 service/kubernetes-dashboard 8001:443

The --address option allows connections from anywhere. This is only acceptable for demonstration purposes.

5. Open a new browser tab, enter the following url to open the Kubernetes Dashboard for the cluster:

Your browser will likely flag the TLS certificate as being invalid. This is expected since the cluster used a self-signed certificate.

6. Allow your browser to view the website. in recent versions of Chrome, you can do so by typing thisisunsafe

The dashboard asks you for a method to authenticate, you will use the token you output before a token to sign in.

7. Copy the token you output earlier at the step number 3 (in the web terminal, you only need to highlight the token and it is automatically copied to your clipboard) and paste it onto the Enter token line and click Sign in.

The dashboard is revealed:

From the dashboard, you can get the same information found using kubectl commands but with an easy-to-navigate interface.

8. Navigate to Workloads > Stateful Sets > mysql:

The CPU and Memory column graphs are visible because metrics server, opens in a new tab is running in your Kubernetes cluster.

9. Click on the Scale button near the top-right corner of the mysql Stateful Sets view:

10. In the Scale a resource dialog, enter the following value and click Scale:

    Desired replicas: 3

11. Scroll down to the Pods section and observe the mysql-4 pod is removed first, followed by the mysql-3 pod.

Pods are removed in the reverse order that they are created in StatefulSets.

12. Navigate to Cluster > Persistent Volumes:

Notice that there are five PVs. PVs remain after the pods are deleted in a scale-down event. If you want to delete PVs after scaling down, you have to manually perform the operation.

13. Click the three dots in the last column of the table for the PV associated with Claim default/data-mysql-4 (it is the first row) and click Delete.

14. In the confirmation dialog, click Delete.

The operation will not delete the PV because it is currently bound by a PVC.

15. Navigate to Config and Storage> Persistent Volume Claims and click on data-mysql-4.

16. Delete the PVC by clicking Delete in the upper-right corner.

17. Click Delete in the confirmation dialog.

18. Return. to the PV view and confirm there are now only 4 PVs. 

Summary 

In this lab step, you bound the cluster-admin role to the dashboard service account to gain full access to the dashboard without authentication. You ran a kubectl port-forward in order to connect to the cluster from the bastion host. You also explored the cluster monitoring and management capabilities of the Kubernetes Dashboard.

Feel free to use the remaining time in the lab session to explore more of the Kubernetes dashboard.
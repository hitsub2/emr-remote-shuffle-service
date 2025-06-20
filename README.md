# EMR remote shuffle service

Remote Shuffle Service (RSS) provides the capability for Apache Spark applications to store shuffle data 
on a cluster of remote servers. See more details on Spark community document: 
[[SPARK-25299][DISCUSSION] Improving Spark Shuffle Reliability](https://docs.google.com/document/d/1uCkzGGVG17oGC6BJ75TpzLAZNorvrAU3FRd2X-rVHSM/edit?ts=5e3c57b8).

In this repo, we select [Aapche Celeborn](https://celeborn.apache.org/) as the remote shuffle service for EMR. The high level design of Apache Celeborn can be found [here](https://github.com/apache/incubator-celeborn).

# Setup instructions:
* [1. Install Apache Celeborn for EMR on EKS](#1-install-apache-celeborn-on-eks)
* [2. Install Apache Celeborn for EMR on EC2](#2-install-apache-celeborn-on-ec2)

## Prerequisite
1. [AWS CLI](https://docs.aws.amazon.com/cli/latest/userguide/install-cliv2.html). Configure the CLI by `aws configure`.
2. [kubectl >=1.24](https://docs.aws.amazon.com/eks/latest/userguide/install-kubectl.html)
3. [eksctl >= 0.143.0](https://docs.aws.amazon.com/eks/latest/userguide/eksctl.html)
4. [helm](https://helm.sh/docs/intro/install/)
5. [Terraform](https://learn.hashicorp.com/tutorials/terraform/install-cli)
6. [OPTIONAL: Buildx included by Docker Desktop installation](https://docs.docker.com/desktop/)

## Infrastructure
If you do not have your own environment to test the remote shuffle solution, we will use the Data on EKS blueprint to provision the cluster, deploy Karpenter, Amazon Managed Prometheus, grafana-dashboard. Change the EKS cluster name and AWS region if needed in the ```variables.tf```.

```bash
git clone https://github.com/awslabs/data-on-eks.git
cd ./data-on-eks/analytics/terraform/emr-eks-karpenter
terraform init
```

To deploy the EMR Spark Operator Add-on. You need to set the the below value to `true` in `variables.tf` file.

```bash
variable "enable_emr_spark_operator" {
  description = "Enable the Spark Operator to submit jobs with EMR Runtime"
  default     = true
  type        = bool
}
```

:::

Deploy the pattern

```bash
terraform apply
```

Enter `yes` to apply.

## Verify the resources

Let’s verify the resources created by `terraform apply`.

Verify the Spark Operator and Amazon Managed service for Prometheus.

```bash

helm list --namespace spark-operator -o yaml

aws amp list-workspaces --alias amp-ws-emr-eks-karpenter

```

Verify Namespace `emr-data-team-a` and Pod status for `Prometheus`, `Vertical Pod Autoscaler`, `Metrics Server` and `Cluster Autoscaler`.

```bash
aws eks --region us-west-2 update-kubeconfig --name spark-operator-doeks # Creates k8s config file to authenticate with EKS Cluster

kubectl get nodes # Output shows the EKS Managed Node group nodes

kubectl get ns | grep emr-data-team # Output shows emr-data-team-a for data team

kubectl get pods --namespace=vpa  # Output shows Vertical Pod Autoscaler pods

kubectl get pods --namespace=kube-system | grep  metrics-server # Output shows Metric Server pod

kubectl get pods --namespace=kube-system | grep  cluster-autoscaler # Output shows Cluster Autoscaler pod
```

Create karpenter nodepools and ec2nodeclass

Please update the settings in nodepool and ec2nodeclass, like ```role`` etc.

```bash

git clone https://github.com/aws-samples/emr-remote-shuffle-service.git
cd emr-remote-shuffle-service
kubectl apply -f Karpenter

```


## Enable Remote Shuffle Server (RSS)

Apache Celeborn supports Spark 2.4/3.0/3.1/3.2/3.3/3.4/3.5 and flink 1.14/1.15/1.17. The test was done under the default Java 8 environment. However, you can compile the RSS project based on other Java versions. For example:

```bash
./build/make-distribution.sh -Pspark-3.5 -Pjdk-21
```
There are 2 options to host the RSS server:

### 1. Install Apache Celeborn on EKS

```bash
git clone https://github.com/aws-samples/emr-remote-shuffle-service.git
cd emr-remote-shuffle-service
```

#### Build Docker Images
For the best practice in security, it's recommended to build your own images and publish them to your own container repository.

<details>
<summary>OPTIONAL: How to build docker image</summary>

```bash
# Login to ECR
ACCOUNT_ID=$(aws sts get-caller-identity --query Account --output text)
ECR_URL=$ACCOUNT_ID.dkr.ecr.$AWS_REGION.amazonaws.com
aws ecr get-login-password --region $AWS_REGION | docker login --username AWS --password-stdin $ECR_URL

# create a new repository as a one-off task
aws ecr create-repository --repository-name celeborn-server \
  --image-scanning-configuration scanOnPush=true
aws ecr create-repository --repository-name clb-spark-benchmark \
  --image-scanning-configuration scanOnPush=true
```  

```bash
# Build & push server & client docker images
JDK_VERSION=8  #17
SPARK_VERSION=3.3
# build server
docker build -t $ECR_URL/celeborn-server:spark${SPARK_VERSION}_jdk${JDK_VERSION} \
  --build-arg SPARK_VERSION=${SPARK_VERSION} \
  --build-arg JDK_VERSION=${JDK_VERSION} \
  --build-arg java_image_tag=${JDK_VERSION}-jdk-focal \
  -f docker/celeborn-server/Dockerfile .
# push the image to ECR
docker push $ECR_URL/celeborn-server:spark${SPARK_VERSION}_jdk${JDK_VERSION}
```

Alternatively, we can build a single multi-arch docker image (x86_64 and arm64) by the following steps:

```bash
# validate if the Docker Buildx CLI extension is installed
docker buildx version
# (once-off task) create a new builder that gives access to the new multi-architecture features
docker buildx create --name mybuilder --use
# build and push the custom image supporting multi-platform
JDK_VERSION=8  #17
SPARK_VERSION=3.3
docker buildx build \
--platform linux/amd64,linux/arm64 \
-t $ECR_URL/celeborn-server:spark${SPARK_VERSION}_jdk${JDK_VERSION} \
--build-arg SPARK_VERSION=${SPARK_VERSION} \
--build-arg JDK_VERSION=${JDK_VERSION} \
--build-arg java_image_tag=${JDK_VERSION}-jdk-focal \
-f docker/celeborn-server/Dockerfile \
--push .
```

```bash
# build client for EMR on EKS
JDK_VERSION=8  #17
SPARK_VERSION=3.3
EMR_VERSION=emr-6.10.0
aws ecr-public get-login-password --region us-east-1 | docker login --username AWS --password-stdin public.ecr.aws

docker build -t $ECR_URL/clb-spark-benchmark:${EMR_VERSION}_clb \
  --build-arg SPARK_VERSION=${SPARK_VERSION} \
  --build-arg JDK_VERSION=${JDK_VERSION} \
  --build-arg SPARK_BASE_IMAGE=public.ecr.aws/emr-on-eks/spark/${EMR_VERSION}:latest \
  --build-arg java_image_tag=${JDK_VERSION}-jdk-focal \
  -f docker/celeborn-emr-client/Dockerfile .

docker push $ECR_URL/clb-spark-benchmark:${EMR_VERSION}_clb
```

```bash
# build client for OSS Spark
SPARK_BASE_IMAGE=public.ecr.aws/myang-poc/spark:3.3.1_hadoop_3.3.3
SPARK_VERSION=3.3
ECR_URL=$ACCOUNT_ID.dkr.ecr.$AWS_REGION.amazonaws.com
docker build -t $ECR_URL/clb-spark-benchmark:spark${SPARK_VERSION}_client \
  --build-arg SPARK_VERSION=${SPARK_VERSION} \
  --build-arg SPARK_BASE_IMAGE=${SPARK_BASE_IMAGE} \
  -f docker/celeborn-oss-client/Dockerfile .

docker push $ECR_URL/clb-spark-benchmark:spark${SPARK_VERSION}_client
```

</details>

#### Run Celeborn shuffle service in EKS
Celeborn helm chart comes with a monitoring feature. Check out the `OPTIONAL` step to install a Prometheus Operator in order to collect the RSS server metrics on EKS. 

To Setup Amazon Managed Grafana dashboard sourced from Amazon Managed Prometheus, check out the instruction [here](https://github.com/melodyyangaws/karpenter-emr-on-eks/blob/main/setup_grafana_dashboard.pdf). Two pre-build Grafana dashbaords can be imported to your dashboard: [EMR on EKS dashboard](https://raw.githubusercontent.com/awslabs/data-on-eks/main/analytics/terraform/emr-eks-karpenter/emr-grafana-dashboard/emr-eks-grafana-dashboard.json),and the [Celeborn dashboard](https://github.com/apache/incubator-celeborn/blob/main/assets/grafana/celeborn-dashboard.json).
<details>
<summary>OPTIONAL: Install Prometheus for monitoring</summary>
Celeborn's helm chart installs Prometheus Operator by default. In this exmaple, we will use AWS serverelss offerings: Amazon Managed Prometheus (AMP) and Amazon Managed Grafana to collect metrics and visulaize the RSS server performance in EKS.

```bash
ACCOUNT_ID=$(aws sts get-caller-identity --query Account --output text)
kubectl create namespace prometheus
amp=$(aws amp list-workspaces --query "workspaces[?alias=='$EKSCLUSTER_NAME'].workspaceId" --output text)
if [ -z "$amp" ]; then
    echo "Creating a new prometheus workspace..."
    export WORKSPACE_ID=$(aws amp create-workspace --alias $EKSCLUSTER_NAME --query workspaceId --output text)
else
    echo "A prometheus workspace already exists"
    export WORKSPACE_ID=$amp
fi
sed -i -- 's/{AWS_REGION}/'$AWS_REGION'/g' charts/celeborn-shuffle-service/prometheusoperator_values.yaml
sed -i -- 's/{ACCOUNTID}/'$ACCOUNT_ID'/g' charts/celeborn-shuffle-service/prometheusoperator_values.yaml
sed -i -- 's/{WORKSPACE_ID}/'$WORKSPACE_ID'/g' charts/celeborn-shuffle-service/prometheusoperator_values.yaml
sed -i -- 's/{EKSCLUSTER_NAME}/'$EKSCLUSTER_NAME'/g' charts/celeborn-shuffle-service/prometheusoperator_values.yaml
```
```bash
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo update
# check the `yaml`, ensure varaibles are populated first
helm upgrade --install prometheus prometheus-community/kube-prometheus-stack -n prometheus -f charts/celeborn-shuffle-service/prometheusoperator_values.yaml --debug
# validate in a web browser - localhost:9090, go to menu of status->targets
kubectl --namespace prometheus port-forward service/prometheus-kube-prometheus-prometheus 9090

# create pod monitor for Spark apps
kubectl apply -f charts/celeborn-shuffle-service/spark-podmonitor.yaml
```
</details>

```bash
# config celeborn server and replace docker images if needed
vi charts/celeborn-shuffle-service/values.yaml
```

```bash
# install Celeborn for Spark3.3 (EMR 6.10)
helm install celeborn charts/celeborn-shuffle-service  -n celeborn \
--set image.tag=spark3.3_8-jdk --create-namespace 
# install another Celeborn cluster for Spark3.5(EMR 7.0) if needed
helm install celeborn-spark3-5 charts/celeborn-shuffle-service  -n celeborn \
--set image.tag=spark3.5_jdk17 --create-namespace=false
```
<details>
<summary>OPTIONAL: validate after the RSS installation</summary>

Check Celeborn pods' status:

```bash
>> kubectl get all -n celeborn
NAME                    READY   STATUS    RESTARTS   AGE
pod/celeborn-master-0   1/1     Running   0          20m
pod/celeborn-master-1   1/1     Running   0          20m
pod/celeborn-master-2   1/1     Running   0          2m28s
pod/celeborn-worker-0   1/1     Running   0          20m
pod/celeborn-worker-1   1/1     Running   0          20m
pod/celeborn-worker-2   1/1     Running   0          17m
```
Ensure all workers are up running and are registered to a single master node.
In this case, all workers were registered with "master-1".

```bash
kubectl logs celeborn-master-0 -n celeborn | grep Registered
kubectl logs celeborn-master-1 -n celeborn | grep Registered
kubectl logs celeborn-master-2 -n celeborn | grep Registered
```
```bash
>> kubectl logs celeborn-master-1 -n celeborn | grep Registered
Defaulted container "celeborn" out of: celeborn, chown-celeborn-master-volume (init)
24/11/02 02:03:33,176 INFO [celeborn-dispatcher-3] Master: Registered worker 
24/11/02 02:03:33,856 INFO [celeborn-dispatcher-0] Master: Registered worker 
24/11/02 02:03:33,905 INFO [celeborn-dispatcher-2] Master: Registered worker 
```
Check if multiple disks were mounted to the worker node correctly

```bash
>> kubectl exec -it celeborn-worker-1 -n celeborn -- df -h
Defaulted container "celeborn" out of: celeborn, chown-celeborn-worker-volume (init)
Filesystem      Size  Used Avail Use% Mounted on
overlay          20G  4.1G   16G  21% /
tmpfs            64M     0   64M   0% /dev
tmpfs            94G     0   94G   0% /sys/fs/cgroup
/dev/nvme1n1    6.9T   49G  6.8T   1% /rss1/disk1
/dev/nvme2n1    6.9T   49G  6.8T   1% /rss2/disk2
/dev/nvme0n1p1   20G  4.1G   16G  21% /etc/hosts
```
Check if Spark jobs are writing to the RSS
```bash
>> kubectl logs celeborn-spark3-5-worker-1 -n celeborn 
# some data were persisted to the Direct memory usage(offheap memory) & disk
24/11/03 22:17:52,477 INFO [worker-memory-manager-reporter] MemoryManager: Direct memory usage: 416.0 MiB/24.0 GiB, disk buffer size: 96.3 MiB, sort memory size: 0.0 B, read buffer size: 0.0 B
# some shuffle files were commited to RSS from a Spark app
24/11/03 22:17:57,533 INFO [worker-forward-message-scheduler] StorageManager: Updated diskInfos:
DiskInfo(maxSlots: 0, committed shuffles 2, running applications 1, shuffleAllocations: Map(spark-000000034nhngp02t92-2 -> 334), mountPoint: /rss2/disk2, usableSpace: 6.0 TiB, avgFlushTime: 234546 ns, avgFetchTime: 0 ns, activeSlots: 334, storageType: SSD) status: HEALTHY dirs /rss2/disk2/celeborn-worker/shuffle_data
```
```bash
# OPTIONAL: only if prometheus operator is installed
kubectl get podmonitor -n celeborn
```
</details>

```bash
# scale worker or master if needed
kubectl scale statefulsets celeborn-worker -n celeborn  --replicas=5
kubectl scale statefulsets celeborn-master -n celeborn  --replicas=1

# uninstall celeborn if needed
helm uninstall celeborn -n celeborn
```
### 2. Install Apache Celeborn on EC2

Before setup the RSS server on EC2, we need to build a Celeborn binary as below. A package apache-celeborn-${project.version}-bin.tgz will be generated. 

```bash
git clone git@github.com:apache/incubator-celeborn.git
cd incubator-celeborn
./build/make-distribution.sh -Pspark-3.3
```
For a quick start, you can download the pre-compiled version 0.2.2 for Spark3.3 from the [link](https://meloyang-emr-bda.s3.amazonaws.com/spark3.3-apache-celeborn-0.2.2-SNAPSHOT-bin.tgz), then deploy the binary to 4 X EC2 instance - 1 master + 3 workers. In this exmaple, we use the instance type `i3en.6xlarge` to host the cluster of RSS server.

Firstly, spin up and login to an EC2 instance, follow the steps below to build an RSS-enabled AMI, then deploy to other 3 EC2 nodes:

1.Mount 2 X instance store to the host:

```bash
# Create partition, format and mount it
sudo parted /dev/nvme1n1 mktable gpt
sudo parted /dev/nvme1n1 mkpart primary ext4 1MB 100%
sudo parted /dev/nvme2n1 mktable gpt
sudo parted /dev/nvme2n1 mkpart primary ext4 1MB 100%

sudo mkfs -t ext4 /dev/nvme1n1 /dev/nvme2n1
sudo mkdir -p /mnt/disk1
sudo mount /dev/nvme1n1 /mnt/disk1
sudo mount /dev/nvme2n1 /mnt/disk2
sudo chown -R celeborn:celeborn /mnt/disk1 /mnt/disk2
```

2.Create a celeborn user and CELEBORN_HOME:

```bash
export celeborn_uid=10006
export celeborn_gid=10006
export CELEBORN_HOME=/opt/celeborn

apt-get update && \
    apt-get install -y git bash tini bind9-utils telnet net-tools procps dnsutils krb5-user && \
    ln -snf /bin/bash /bin/sh && \
    rm -rf /var/cache/apt/* && \
    groupadd --gid=${celeborn_gid} celeborn && \
    useradd  --uid=${celeborn_uid} --gid=${celeborn_gid} celeborn -d /home/celeborn -m && \
    mkdir -p ${CELEBORN_HOME}
```

3.Unzip the tarball to $CELEBORN_HOME:

```bash
curl https://meloyang-emr-bda.s3.amazonaws.com/spark3.3-apache-celeborn-0.2.2-SNAPSHOT-bin.tgz
cat *.tgz | tar -xvzf - && mv apache-celeborn-*-bin /opt/celeborn
```

4.Modify directory permission in CELEBORN_HOME :

```bash
chown -R celeborn:celeborn ${CELEBORN_HOME} && \
    chmod -R ug+rw ${CELEBORN_HOME} && \
    chmod a+x ${CELEBORN_HOME}/bin/* && \
    chmod a+x ${CELEBORN_HOME}/sbin/*
```

5.Modify environment variables in `$CELEBORN_HOME/conf/celeborn-env.sh`

Example can be found here:

```yaml
vi /opt/celeborn/conf/celeborn-env.sh

CELEBORN_MASTER_MEMORY=8g
CELEBORN_WORKER_MEMORY=8g
CELEBORN_WORKER_OFFHEAP_MEMORY=130g
CELEBORN_NO_DAEMONIZE=1
CELEBORN_WORKER_JAVA_OPTS="-XX:-PrintGC -XX:+PrintGCDetails -XX:+PrintGCTimeStamps -XX:+PrintGCDateStamps -Xloggc:/opt/celeborn/logs/gc-worker.out -Dio.netty.leakDetectionLevel=advanced"
CELEBORN_MASTER_JAVA_OPTS="-XX:-PrintGC -XX:+PrintGCDetails -XX:+PrintGCTimeStamps -XX:+PrintGCDateStamps -Xloggc:/opt/celeborn/logs/gc-master.out -Dio.netty.leakDetectionLevel=advanced"
CELEBORN_PID_DIR="/opt/celeborn/pids"
CELEBORN_LOG_DIR="/opt/celeborn/logs"
```

6.Modify configurations in `$CELEBORN_HOME/conf/celeborn-defaults.conf`

EXAMPLE: a single master RSS cluster:

```yaml
# HA master mode in the EKS example
# however, we use the single master mode to simplify the EC2 setup
celeborn.ha.enabled=false

# used by client and worker to connect to master
# the endpoint can be either an alias or use your EC2's private IP DNS name
celeborn.master.endpoints=celeborn-master-0:9097
# celeborn.master.endpoints=ip-10-0-49-238.us-west-2.compute.internal:9097

celeborn.metrics.enabled=true
celeborn.network.timeout=2000s
celeborn.worker.storage.dirs /mnt/disk1,/mnt/disk2
celeborn.master.metrics.prometheus.port=9098
celeborn.worker.metrics.prometheus.port=9096
# If your hosts have disk raid or use lvm, set celeborn.worker.monitor.disk.enabled to false
celeborn.worker.monitor.disk.enabled=false
celeborn.push.io.numConnectionsPerPeer=8
celeborn.replicate.io.numConnectionsPerPeer=24
celeborn.worker.closeIdleConnections=true
celeborn.worker.commit.threads=128
celeborn.worker.fetch.io.threads=256
celeborn.worker.flusher.buffer.size=128k
# celeborn.worker.flusher.threads=512
celeborn.worker.flusher.ssd.threads: 512
celeborn.worker.push.io.threads=128
celeborn.worker.replicate.io.threads=128
# # worker recover - ip & port must be the same after a worker-restart.
# celeborn.worker.graceful.shutdown.enabled: true
# celeborn.worker.graceful.shutdown.recoverPath: /tmp/recover
# celeborn.worker.rpc.port: 9094
# celeborn.worker.fetch.port: 9092
# celeborn.worker.push.port: 9091
# celeborn.worker.replicate.port: 9089
```
7.Create a file called `$CELEBORN_HOME/conf/log4j2.xml` copied from `$CELEBORN_HOME/conf/log4j2.xml.template`
    
8.OPTIONAL: setup the /etc/hosts, make sure all worker and master nodes can SSH to each other:

```yaml
#Example:

192.168.51.237 ip-192-168-51-237.us-west-2.compute.internal celeborn-worker-0
192.168.48.26 ip-192-168-48-26.us-west-2.compute.internal celeborn-worker-1
```

9.OPTIONAL: add a host file to $CELEBORN_HOME/conf/hosts, update the hostname by your alias accordingly.
 
```bash
#Example:

[master]
celeborn-master-0

[worker]
celeborn-worker-0
celeborn-worker-1
celeborn-worker-2
```

10.OPTIONAL: Use the command to start all services. 

```bash
$CELEBORN_HOME/sbin/start-all.sh
```

11.Alternatively,  without setup the hosts files, you can manually start the service by SSH to each EC2 instance.
  
- Login to an EC2 act as a master node, then start the Celeborn master service:

```bash
    $CELEBORN_HOME/sbin/start-master.sh
```

- Login to the rest of 3 X EC2s, start the Celeborn worker:

```bash
    $CELEBORN_HOME/sbin/start-worker.sh
```

12.If Celeborn starts successfully, the output of the Master's log should look like this:

```yaml
22/10/08 19:29:11,805 INFO [main] Dispatcher: Dispatcher numThreads: 64
22/10/08 19:29:11,875 INFO [main] TransportClientFactory: mode NIO threads 64
22/10/08 19:29:12,057 INFO [main] Utils: Successfully started service 'MasterSys' on port 9097.
22/10/08 19:29:12,113 INFO [main] Master: Metrics system enabled.
22/10/08 19:29:12,125 INFO [main] HttpServer: master: HttpServer started on port 9098.
22/10/08 19:29:12,126 INFO [main] Master: Master started.
22/10/08 19:29:57,842 INFO [dispatcher-event-loop-19] Master: Registered worker
Host: 192.168.15.140
RpcPort: 37359
PushPort: 38303
FetchPort: 37569
ReplicatePort: 37093
SlotsUsed: 0()
LastHeartbeat: 0
Disks: {/mnt/disk1=DiskInfo(maxSlots: 6679, committed shuffles 0 shuffleAllocations: Map(), mountPoint: /mnt/disk1, usableSpace: 448284381184, avgFlushTime: 0, activeSlots: 0) status: HEALTHY dirs , /mnt/disk3=DiskInfo(maxSlots: 6716, committed shuffles 0 shuffleAllocations: Map(), mountPoint: /mnt/disk3, usableSpace: 450755608576, avgFlushTime: 0, activeSlots: 0) status: HEALTHY dirs , /mnt/disk2=DiskInfo(maxSlots: 6713, committed shuffles 0 shuffleAllocations: Map(), mountPoint: /mnt/disk2, usableSpace: 450532900864, avgFlushTime: 0, activeSlots: 0) status: HEALTHY dirs , /mnt/disk4=DiskInfo(maxSlots: 6712, committed shuffles 0 shuffleAllocations: Map(), mountPoint: /mnt/disk4, usableSpace: 450456805376, avgFlushTime: 0, activeSlots: 0) status: HEALTHY dirs }
WorkerRef: null
```

Now, let's configure the RSS Client side.

1.Setup Spark client

Copy `$CELEBORN_HOME/spark/*.jar` from your unzipped the tarball directory to `/usr/lib/spark/jars/` on your EMR on EC2 cluster where runs Spark applications. Or directly download the [pre-compiled jar](https://meloyang-emr-bda.s3.amazonaws.com/celeborn-client-spark-3-shaded_2.12-0.2.2-SNAPSHOT.jar) for Spark version 3.3 to each EMR on EC2 nodes. 

**NOTE**: Both Celeborn and Spark versions must match between the Server and Client side. For instance, if you chose the quick start approach, your client jar must be compiled for the Celeborn version 0.2.2 and Spark 3.3. 


2.Or skip the step 1, and simply submit a Spark job mapping the client jar to your Spark's class path location during the submission:
```
--jars /usr/lib/spark/jars/celeborn-client-spark-3-shaded_2.12-0.2.2-SNAPSHOT.jar
```

Additionally, the following spark configurations should be included for a RSS-enabled Spark job:

```bash
# Shuffle manager class name changed in 0.3.0:
# < 0.3.0: org.apache.spark.shuffle.celeborn.RssShuffleManager
# >= 0.3.0: org.apache.spark.shuffle.celeborn.SparkShuffleManager
spark.shuffle.manager org.apache.spark.shuffle.celeborn.RssShuffleManager
# must use kryo serializer because java serializer do not support relocation
spark.serializer org.apache.spark.serializer.KryoSerializer
# celeborn master
spark.celeborn.master.endpoints celeborn-master-0:9097
spark.shuffle.service.enabled false
# options: hash, sort
# Hash shuffle writer use (partition count) * (celeborn.push.buffer.max.size) * (spark.executor.cores) memory.
# Sort shuffle writer uses less memory than hash shuffle writer, if your shuffle partition count is large, try to use the sort hash writer.  
spark.celeborn.client.spark.shuffle.writer hash
# We recommend setting spark.celeborn.client.push.replicate.enabled to true to enable server-side data replication
# If you have only one worker, this setting must be false 
# If your Celeborn is using HDFS, it's recommended to set this setting to false
spark.celeborn.client.push.replicate.enabled true
# Support for Spark AQE only tested under Spark 3
# we recommend setting localShuffleReader to false to get better performance of Celeborn
spark.sql.adaptive.localShuffleReader.enabled false
# If Celeborn is using HDFS
# spark.celeborn.storage.hdfs.dir hdfs://<namenode>/celeborn
# we recommend enabling aqe support to gain better performance
spark.sql.adaptive.enabled true
spark.sql.adaptive.skewJoin.enabled true

spark.celeborn.shuffle.chunk.size 4m
spark.celeborn.client.push.maxReqsInFlight 128
spark.celeborn.rpc.askTimeout 240s
spark.celeborn.client.push.blacklist.enabled true
spark.celeborn.client.push.excludeWorkerOnFailure.enabled true
spark.celeborn.client.fetch.excludeWorkerOnFailure.enabled true
spark.celeborn.client.commitFiles.ignoreExcludedWorker true

spark.sql.optimizedUnsafeRowSerializers.enabled false

```

## Run TPCDS Benchmark
### OPTIONAL: generate the TCP-DS source data
Execute the following job, which will generate TPCDS source data at 3TB scale to your S3 bucket `s3://'$S3BUCKET'/BLOG_TPCDS-TEST-3T-partitioned/`. Alternatively, directly copy the source data from `s3://blogpost-sparkoneks-us-east-1/blog/BLOG_TPCDS-TEST-3T-partitioned` to your S3.

```bash
kubectl apply -f examples/tpcds-data-gen.yaml
```

### Run EMR on EKS Spark benchmark:
All jobs will run in a single namespace `emr` but seperate nodegroups in EKS cluster. Update the docker image name to your ECR URL in the following shell scripts, then run:

```bash
# go to the project root directory
cd emr-remote-shuffle-service
export EMRCLUSTER_NAME=<YOUR_EMR_VIRTUAL_CLUSTER_NAME:emr-on-eks-rss>
export AWS_REGION=<YOUR_REGION:us-west-2>

# create a job template first
aws emr-containers create-job-template --cli-input-json file://example/pod-template/clb-dra-job-template.json
aws emr-containers create-job-template --cli-input-json file://example/pod-template/dra-tracking-job-template.json

# Example 1: Without a patch for DRA in Spark 3.3, run TPCDS test with RSS+DRA
./example/emr6.10-benchmark-celeborn.sh
# Example 2: Without shuffle tracking in Spark3.5+, run TPCDS test with RSS+DRA
./example/emr7.0-benchmark-celeborn.sh
# Example 3: Without RSS, run a usual job with DRA on. Expect Data Loss.
./example/emr6.10-benchmark-norss.sh
# check job progress
kubectl get po -n emr
kubectl logs <DRIVER_POD_NAME> -n emr spark-kubernetes-driver
```


### OPTIONAL: Run OSS Spark benchmark
NOTE: some queries may not be able to complete, due to the limited resources alloated to the large scale test. Update the docker image to your image repository URL, then test the performance with the remote shuffle service enabled. 

For example:

```bash
# oss spark without RSS
kubectl apply -f oss-benchmark.yaml
# with RSS
kubectl apply -f oss-benchmark-celeborn.yaml
# turn on DRA with RSS
kubectl apply -f oss-benchmark-celeborn-dra.yaml
```

```bash
# check job progress
kubectl get pod -n oss
# check application logs
kubectl logs celeborn-benchmark-driver -n oss
```

# Log file analyzer - a distributed service!
### This is a streaming data pipeline with an actor-model service using [Akka](https://akka.io/) that ingests logfile generated data in real time and delivers it via an event-based service called [Kafka](https://kafka.apache.org/) to [Spark](https://spark.apache.org/) for further processing. 


## Development Environment
+ Language : Scala v
+ IDE : Intellij IDEA Community Edition 
+ Build Tool : SBT 1.5.2
+ Frameworks Used : CloudFlow,Akka,Spark, Kafka
+ Deployment : AWS EKS, AWS Lambda

## Installations
+ Install and build [cloudflow](https://cloudflow.io/docs/dev/administration/installing-cloudflow.html)
+ Installing [Kafka and Strimzi](https://cloudflow.io/docs/dev/administration/how-to-install-and-use-strimzi.html)
+ Adding [Spark support](https://cloudflow.io/docs/dev/administration/installing-spark-operator.html)
+ Install and setup [docker](https://docs.docker.com/get-docker/)

## Development and Deployment

To run cloudflow application, we first have to install cloudflow locally along with the cloudflow kubectl plugin. [Ref](https://cloudflow.io/docs/dev/administration/installing-cloudflow.html)

To deploy the application, we first create a Kubernetes cluster in EKS. Cloudflow also provides a (script)[https://cloudflow.io/docs/2.0.0/get-started/setup-k8s-cluster.html] to spin up a Kubernetes cluster with the required configurations. Once the cluster is ready, we add the context of the aws cluster to our local kubectl. `aws eks update-kubeconfig --name clusterName`.

Now we use Helm to install Cloudflow within the kubernetes cluster along with (Strimzi-kafka)[https://cloudflow.io/docs/dev/administration/how-to-install-and-use-strimzi.html] and the (Cloudflow spark operator)[https://cloudflow.io/docs/dev/administration/how-to-install-and-use-strimzi.html].

Once the installation is completed, we would be able to see all the operators running as pods in the cluster as below:

![image](https://user-images.githubusercontent.com/33444577/144808247-c86d778f-48f8-43dc-8bd0-24811f3561f6.png)

We also have to create a PersistentVolume in the kubernetes cluster with some storage provisioner and create a PersistentVolumeClaim of the name "cloudflow-spark" for the spark operator to store the data as explained in the cloudflow documentation.

![image](https://user-images.githubusercontent.com/33444577/144808348-8bf418c9-2671-4b38-b270-78d70f973e49.png)

In this case we use a open source NFS storage provisioner to create our Persistent Volume that the spark operator will use.

Once the application is developed, we first have to package our application into docker containers by adding the registry-repository details in the file `target-env.sbt'

+ Create a docker repo
```
ThisBuild / cloudflowDockerRegistry := Some("docker.io")
ThisBuild / cloudflowDockerRepository := Some("<your docker repository>")
  ```
 Once this is done, the cloudflow application will automatically create the docker images and push them to the repository specified.

+ Build and publish the app using 
```
sbt buildApp
```

+ To deploy the applications to kubernetes cluster 
```
$ cat dockerpwd.txt | kubectl cloudflow deploy /path/to/CS441-CourseProject/target/CS441-CourseProject.json -u dockeruser --password-stdin

If using AWS ECR, provide those credentials instead.
```
+ After executing these commands you can see the streamlets running in different pods.

## Project Structure
+ Logprocessor - Write the logs to s3.
+ akka-log-ingestor - Contains the ingestor streamlet which receives the key from the lambda function and adds the records in kafka streams.
+ log-processor-pipeline - Contains the blueprint which defines the flow of the streamlets.
+ spark-aggregator - Contains the spark aggregation program to perform some data aggregation tasks.
+ EmailProc.scala - Contains the code to dispatch email using AWS SES.

Complete flow along with loggers can be see[n here](flow.md)

### About Cloudflow Applications

Cloudflow provides an easy way to package and deploy structured streaming applications for streaming pipelines. Cloudflow uses the blueprint to orchestrate the streaming pipeline which are physically implemented as Kafka streams that are made typesafe during compile time using on Avro Schemas.

Some concepts on cloudflow.

## [Cloudflow Streamlets](https://developer.lightbend.com/docs/cloudflow/current/streamlets.html)
Streamlets are cannonical class names which have inlets and outlets. A Streamlet can have more than one inlet and outlet.
There are different streamlet shapes
+ Ingress - Ingress are streamlets with zero inlets and one or more outlets
+ Processor - A processor has one inlet and one outlet.
+ Fanout - FanOut-shaped streamlets have a single inlet and two or more outlets.
+ Fanin - FanIn-shaped streamlets have a single outlet and two or more inlets.
+ Egress - Egress has inlets but zero outlets.

## [Avro Schemas]((https://developer.lightbend.com/docs/cloudflow/current/streamlets.html))
Inlets and outlets of specific streamlets can handle data specified by Avro schemas.

## [Blueprint](https://developer.lightbend.com/docs/cloudflow/current/streamlets.html)
The shape of Streamlets with inlets and outlets are specified in a blueprint.

The image below shows a general understanding of cloudflow model.

![image](./images/cloudflow_image.png)

The figure shows the structures of akka streamlets and kafka streams in a cloudflow project.
##Implementation
The overall architecture of our model is given by the image below

![architecture](./images/architecture.drawio.png) 

We have written a cronjob to add files to s3 bucket every 2 minutes. This triggers an AWS Lambda function which sends the key of the updated file to an akka streamlet which performs some file processing on "WARN" and "ERROR" messages.<br>
This contents are transferred to kafka streams which are accessed by the spark streamlet which performs some aggregations on these messages and sends the outputs to clients using AWS email service.

## Running the application
+ execute ``` sbt buildApp```
+ Deploy the application in amazon EKS as given in the description above
+ Run the cron job to add files to s3. After files have been added to s3 you should see the following output <br/>
![output](./images/output.jpg) <br>

This is the output produced by the spark program which is a generated email that contains the count of "WARN" and "ERROR" messages.

## Tests
To run the test execute the command `sbt test` <br>
The test/Test.scala has all the tests to be performed
1. Test if the bucket exists in s3
2. Test for name of the s3 bucket key of the file
3. Test for name of s3 region
4. Test for checking the email dispatching function
5. Test for Spark window












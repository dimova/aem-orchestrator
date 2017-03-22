[![Build Status](https://img.shields.io/travis/shinesolutions/aem-orchestrator.svg)](http://travis-ci.org/shinesolutions/aem-orchestrator)

# AEM Orchestrator
AEM Orchestrator is a stateless Java application for orchestrating AEM infrastructure created using [aem-aws-stack-builder](https://github.com/shinesolutions/aem-aws-stack-builder). It's primary function is to keep Adobe Experience Manager (AEM) running in a healthy state despite scaling events or other such impacts on the stack. It does this by listening to a predefined SQS queue and reacting to changes on the stack.


## Build

This project requires Java 8 to compile and run the source code. Apache Maven 3.3 was used as the build tool.

### Create JAR file
```
$ mvn clean package
```
This will create a JAR file in the '\target' directory called aem-orchestrator-x.x.x.jar. 
By default the generated JAR file will contain all of the required dependencies
  

## Usage
### Requirements
The AEM Orchestrator requires [aem-aws-stack-builder](https://github.com/shinesolutions/aem-aws-stack-builder) to have created a stack in order to work. You'll need to get this running before attempting to run AEM Orchestrator. If using puppet then you should also take a look at [puppet-aem-orchestrator](https://github.com/shinesolutions/puppet-aem-orchestrator).

The AEM Orchestrator uses [AWS Instance Profiles](http://docs.aws.amazon.com/IAM/latest/UserGuide/id_roles_use_switch-role-ec2_instance-profiles.html) authentication. If you do not already have this, you will need to set it up.

The application requires there to be an application.properties file in the same base directory as the JAR file. Here is an example of a properties file with the minimum properties set:

```properties
aws.cloudformation.stackName.author = example-aem-author-stack
aws.cloudformation.stackName.authorDispatcher = example-aem-author-dispatcher-stack
aws.cloudformation.stackName.publish = example-aem-publish-stack
aws.cloudformation.stackName.publishDispatcher = example-aem-publish-dispatcher-stack
aws.sns.topicName = example-aem-asg-event-topic
aws.sqs.queueName = example-aem-asg-event-queue
```

The aem-aws-stack-builder will generate these names for you, they just need to be added to this Orchestrator application.properties file. See [here](docs/configuration.md) for more information on configuration properties. You can also view the base [application.properties](src/main/resources/application.properties) file.

### Running the JAR
The JAR file is created as a 'fully executable' jar. See Spring Boot [deployment and install](http://docs.spring.io/spring-boot/docs/current/reference/html/deployment-install.html).
There are two ways to run the JAR:

```
$ java -jar aem-orchestrator-x.x.x.jar
```

or as a 'fully executable' direct application (only works on Unix/Linux based systems):

```
$ ./aem-orchestrator-x.x.x.jar
```

### Logging
By default the Orchestrator will log to a file called *orchestrator.log* in the root directory. It uses [logback](https://logback.qos.ch/) and here is the default [configuration file](src/main/resources/logback.xml). Note that debug is enabled by default. To override the default logging, place your custom logback.xml file in the same directory as the JAR file.


## Functionality
The AEM Orchestrator reacts to three different types of events:

1. Instance scale up
2. Instance scale down
3. Alarms

### Scale Events
The scale up and scale down events are generated by the AWS autoscaling groups (ASG). These can occur at the following tiers of the stack:

* Author Dispatcher
* Publish
* Publish Dispatcher

For example if a Publish instance stopped responding the ASG would terminate it, which would generate a message to the Orchestrator to perform a *Scale Down Publish Action*. The ASG would also start a new Publish instance, which would generate a *Scale Up Publish Action*. In each case, the AWS instance ID is passed to the Orchestrator so that it can perform it's action.

### Alarms
Currently there is only one alarm, the [AEM Content Health Check](https://github.com/shinesolutions/aem-aws-stack-provisioner/blob/master/templates/aem-tools/content-healthcheck.py.epp). This will trigger if the Publish Dispatcher is not seeing healthy content (defined in a [descriptor file](https://github.com/shinesolutions/aem-aws-stack-provisioner/blob/master/examples/content-healthcheck-descriptor.json)) on the Publish instance. When the alarm triggers the Orchestrator is notified via an SQS message and it will perform an *Alarm Content Health Check Action*.

### Recovery
Aside from stack startup, the AEM Orchestrator is deisgned to recover the stack from multiple scenarios:

* Termination of a single instance (Author Dispatcher, Publish or Publish Dispatcher)
* Termination of many or all instances
* Scale up of a single instance (Author Dispatcher, Publish or Publish Dispatcher)
* Scale up of all instances

NOTE: The AEM Orchestrator must be running to perform this recovery, however due to it's stateless nature, the AEM Orchestrator can be started post termination or scale up of an instance. Upon startup, it will begin processing messages off the SQS queue. In essence the queue holds the state of the stack not the AEM Orchestrator.

### Reading Messages
The AEM Orchestrator attaches itself as a listner to the SQS queue defined in the application.properties. It works in an asyncronous manner and does **not** poll the queue. If the Orchestrator fails to perform an action on a message for what ever reason, the message will not be acknowledged and instead kept in flight on the queue to be reprocessed at a later time. Only one message is processed at a time (no concurrency). There is no garantee to the order of processing, but the Orchestrator is designed to handle messages in any order.

The message format is the Amazon SNS [HTTP/HTTPS Notification JSON Format](http://docs.aws.amazon.com/sns/latest/dg/json-formats.html). A JSON definition of the format can be found [here](https://sns.us-west-2.amazonaws.com/doc/2010-03-31/Notification.json).


## Troubleshooting
If all goes well at start up you should see something like this in your log file:

```
DEBUG c.s.a.s.OrchestratorMessageListener - Initialising message receiver, starting SQS connection
INFO  c.s.aemorchestrator.AemOrchestrator - AEM Orchestrator started
```
Other wise you will need to view the *orchestrator.log* and try and decipher what is going wrong. Most likely causes are:

* Stack not properly initialised
* Invalid or missing IAM role permissions
* Invalid or missing property in the application.properties file
* AEM not in a healthy state on author or publish instances
* AWS Instance Profile authentication not set up

### How do I know it's working?
Upon startup of the stack you will see the Orchestrator log showing the processing of many messages. Here is a log example (with debug off) for a *Scale Up Publish Dispatcher Action*:

```
INFO  c.s.a.s.OrchestratorMessageListener - Message received ID:3dfea271-254c-4cef-a793-ff67a2218239
INFO  c.s.a.a.ScaleUpPublishDispatcherAction - ScaleUpPublishDispatcherAction executing
INFO  c.s.a.a.ScaleUpPublishDispatcherAction - Desired capacity already matching for publish auto scaling group and it's dispatcher. No changes will be made
INFO  c.s.a.s.OrchestratorMessageListener - Acknowledged message (removing from queue): ID:3dfea271-254c-4cef-a793-ff67a2218239
```

If the Orchestrator log isn't moving but there are definately messages in the SQS queue, then something may be wrong (refer to likely causes above). 
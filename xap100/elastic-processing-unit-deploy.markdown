---
layout: post100
title:  Deployment
categories: XAP100
parent: elastic-processing-unit-overview.html
weight: 200
---

{% summary %}{% endsummary %}

The deployment of a partitioned (space based) EPU and stateless/web EPU is done via the Admin API.

In order for the deployment to work, the Admin API must first discover a running GSM, ESM (managers) and running GSAs (GigaSpaces agents).

{% highlight java %}
// Wait for the discovery of the managers and at least one GigaSpaces agent
Admin admin = new AdminFactory().addGroup("myGroup").create();
admin.getGridServiceAgents().waitForAtLeastOne();
admin.getElasticServiceManagers().waitForAtLeastOne();
GridServiceManager gsm = admin.getGridServiceManagers().waitForAtLeastOne();
{% endhighlight %}

# Maximum Memory Capacity

The EPU deployment requires two important properties:

- `memoryCapacityPerContainer` defines the Java Heap size of the Java Virtual Machine and is the most granular memory allocation deployment property. It is internally translated to:

{% highlight java %}
commandLineArgument("-Xmx"+memory).commandLineArgument("-Xms"+memory)
{% endhighlight %}

.

- `maxMemoryCapacity` provides an estimate for the maximum total Processing Unit memory.

Here is a typical example for a memory capacity Processing Unit deployment. The example also includes a scale trigger that is explained in the following sections of this page.

{% highlight java %}
// Deploy the Elastic Stateful Processing Unit
ProcessingUnit pu = gsm.deploy(
    new ElasticStatefulProcessingUnitDeployment(new File("myPU.jar"))
           .memoryCapacityPerContainer(16,MemoryUnit.GIGABYTES)
           .maxMemoryCapacity(512,MemoryUnit.GIGABYTES)
           //initial scale
           .scale(new ManualCapacityScaleConfigurer()
                  .memoryCapacity(128,MemoryUnit.GIGABYTES)
                  .create()));
);
{% endhighlight %}

Here is again the same example, this time the deployed Processing Unit is a pure Space (no jar files):

{% highlight java %}
// Deploy the Elastic Space
ProcessingUnit pu = gsm.deploy(
	new ElasticSpaceDeployment("mySpace")
           .memoryCapacityPerContainer(16,MemoryUnit.GIGABYTES)
           .maxMemoryCapacity(512,MemoryUnit.GIGABYTES)
           //initial scale
           .scale(
                new ManualCapacityScaleConfigurer()
         	.memoryCapacity(128,MemoryUnit.GIGABYTES)
         	.create())
		);
{% endhighlight %}

The memoryCapacityPerContainer and maxMemoryCapacity properties are used to calculate the number of partitions for the Processing Unit as follows:

{% highlight java %}
minTotalNumberOfInstances
   = ceil(maxMemoryCapacity/memoryCapacityPerContainer)
   = ceil(1024/256)
   = 4

numberOfPartitions
   = ceil(minTotalNumberOfInstances/(1+numberOfBackupsPerPartition))
   = ceil(4/(1+1))
   = 2
{% endhighlight %}

{% note %}
The number of Processing Unit partitions cannot be changed without re-deployment of the PU.
{%endnote%}

# Maximum Number of CPU Cores

In many cases when you should take the number of space operations per second into consideration when scaling the system. The memory utilization will be a secondary factor when calculating the required scale. For example, if the system performs mostly data updates (as opposed to reading data), the CPU resources could be a limiting factor more than the total memory capacity. In these cases use the `maxNumberOfCpuCores` deployment property. Here is a typical deployment example that includes CPU capacity planning:

{% highlight java %}
// Deploy the EPU
ProcessingUnit pu = gsm.deploy(
        new ElasticStatefulProcessingUnitDeployment(new File("myPU.jar"))
           .memoryCapacityPerContainer(16,MemoryUnit.GIGABYTES)
           .maxMemoryCapacity(512,MemoryUnit.GIGABYTES)
           .maxNumberOfCpuCores(32)

           // continously scale as new machines are started
           .scale(new EagerScaleConfig())
);
{% endhighlight %}

The `maxNumberOfCpuCores` property provides an estimate for the maximum total number of **CPU cores** on machines that have one or more primary processing unit instances deployed (instances that are not in backup state). Internally the number of partitions is calculated as follows:

{% highlight java %}
minTotalNumberOfInstances
   = ceil(maxMemoryCapacity/memoryCapacityPerContainer)
   = ceil(1024/256)=4

minNumberOfPrimaryInstances
   = ceil(maxNumberOfCpuCores/minNumberOfCpuCoresPerMachine)
   = ceil(8/2)
   = 4

numberOfPartitions
   = max(minNumberOfPrimaryInstances,
     ceil(minTotalNumberOfInstances/(1+numberOfBackupsPerPartition))
   = max(4, 4/(1+1) )
   = 4
{% endhighlight %}

In order to evaluate the `minNumberOfCpuCoresPerMachine`, the deployment communicates with each discovered GigaSpaces agent and collects the number of CPU cores the operating system reports. In case a machine provisioning plugin (cloud) is used, the plugin provides that estimate instead. The `minNumberOfCpuCoresPerMachine` deployment property can also be explicitly defined.

# Explicit Number of Partitions

The `numberOfPartitions` property allows explicit definition of the number of space partitions. When the `numberOfPartitions` property is defined then `maxMemoryCapacity` and `maxNumberOfCpuCores` should not be defined.

{% highlight java %}
// Deploy the EPU
ProcessingUnit pu = gsm.deploy(
        new ElasticStatefulProcessingUnitDeployment(new File("myPU.jar"))
           .memoryCapacityPerContainer(16,MemoryUnit.GIGABYTES)
           .numberOfPartitions(12)
           .scale(new EagerScaleConfig())
);
{% endhighlight %}

Here is another example, deployment with explicit number of partitions and memory capacity scale trigger:

{% highlight java %}
// Deploy the EPU
ProcessingUnit pu = gsm.deploy(
        new ElasticStatefulProcessingUnitDeployment(new File("myPU.jar"))
           .memoryCapacityPerContainer(16,MemoryUnit.GIGABYTES)
           .numberOfPartitions(12)
           .scale(new ManualCapacityScaleConfigurer()
                  .memoryCapacity(16,MemoryUnit.GIGABYTES)
                  .create())
           )

);

// Application continues
Thread.sleep(10000);

// Scale out to 32GB memory
pu.scale(new ManualCapacityScaleConfigurer()
         .memoryCapacity(32,MemoryUnit.GIGABYTES)
         .create()
);
{% endhighlight %}

{% comment %}

{: .table .table-bordered .table-condensed}
| memoryCapacityPerContainer | Number of partitions | Instances per container | Total memory |
|:---------------------------|:---------------------|:------------------------|:-------------|
| 6GB | 2 | 1 |  (2\* 2/ 1)\*6GB = 24GB |
| 1GB | 12 | 1 | (2\*12/ 1)\*1GB = 24GB |
| 6GB | 2 | 2 |  (2\* 2/ 2)\*6GB = 12GB |
| 1GB | 12 | 2 | (2\*12/ 2)\*1GB = 12GB |
| 1GB | 12 | 3 | (2\*12/ 3)\*1GB =  8GB |
| 1GB | 12 | 4 | (2\*12/ 4)\*1GB =  6GB |
| 1GB | 12 | 6 | (2\*12/ 6)\*1GB =  4GB |
| 1GB | 12 | 8 | (2\*12/ 8)\*1GB =  3GB |
| 1GB | 12 | 12 |(2\*12/12)\*1GB =  2GB |

{% endcomment %}

Specifying number of partitions explicitly is recommended only when fine grained scale triggers are required. The example below illustrating 12 partitions system (12 primaries + 12 backups = 24 instances). See how the system scales to have increased total memory capacity as a function of the number of Containers and `memoryCapacityPerContainer`:

{% inittab memoryCapacityPerContainer|top %}
{% tabcontent memoryCapacityPerContainer 6G %}

{: .table .table-bordered .table-condensed}
|Number of Containers|Number of partitions per container|Total available memory|
|:-------------------|:---------------------------------|:---------------------|
|2|24 / 2 = 12|2 * 6GB = 12GB|
|4|24 / 4 = 6|4 * 6GB = 24GB |
|8|24 / 8 = 3|8 * 6GB = 48GB |
|12|24 / 12 = 2|12 * 6GB = 72GB|

{% endtabcontent %}
{% tabcontent memoryCapacityPerContainer 12G %}

{: .table .table-bordered .table-condensed}
|Number of Containers|Number of partitions per container|Total available memory|
|:-------------------|:---------------------------------|:---------------------|
|2|24 / 2 = 12|2 * 12GB = 24GB|
|4|24 / 4 = 6|4 * 12GB = 48GB|
|8|24 / 8 = 3|8 * 12GB = 96GB|
|12|24 / 12 = 2|12 * 12GB = 144GB|

{% endtabcontent %}
{% tabcontent memoryCapacityPerContainer 24G %}

{: .table .table-bordered .table-condensed}
|Number of Containers|Number of partitions per container|Total available memory|
|:-------------------|:---------------------------------|:---------------------|
|2|24 / 2 = 12|2 * 24GB = 48GB|
|4|24 / 4 = 6|4 * 24GB = 96GB|
|8|24 / 8 = 3|8 * 24GB = 192GB|
|12|24 / 12 = 2|12 * 24GB = 288GB|

{% endtabcontent %}
{% endinittab %}

{% note %}
Having larger number of partitions will provide you better flexibility in terms of having more scaling "check points". Having too many partitions (hundreds) will impact the system performance since in some point this will generate some overhead due to the internal monitoring required for each partition.
{%endnote%}

{% comment %}
A deployment of 12 instances per container may incur some memory and thread context switching overhead. Under the assumption that the system scales-in as the load decreases, then this overhead should not be significant. Real world performance results of-course depend on the specific use case.
{% endcomment %}

# Deployment on a Single Machine (for development purposes)

For development and demonstration purposes, it is very convenient to deploy the EPU on a single machine. By default, the minimum number of machines is two (for high availability concerns). This could be changed using the `singleMachineDeployment` property.

{% highlight java %}
// Deploy the EPU
ProcessingUnit pu = gsm.deploy(
        new ElasticStatefulProcessingUnitDeployment(new File("myPU.jar"))
           .memoryCapacityPerContainer(256,MemoryUnit.MEGABYTES)
           .maxMemoryCapacity(1024,MemoryUnit.MEGABYTES)
           .singleMachineDeployment()  // deploy on a single machine

           // other processes running on machine would have at least 2GB left
           .dedicatedMachineProvisioning(
               new DiscoveredMachineProvisioningConfigurer()
                  .reservedMemoryCapacityPerMachine(2,MemoryUnit.GIGABYTES)
                  .create())

          //initial scale
           .scale(new ManualCapacityScaleConfigurer()
                  .memoryCapacity(512,MemoryUnit.MEGABYTES)
                  .create())
);
{% endhighlight %}

# Stateless / Web Elastic Processing Units

Stateless Processing Units do not include an embedded space, and therefore are not partitioned. Deployment of stateless processing unit is performed by specifying the required total number of CPU cores. This ensures 1 container per machine.

{% highlight java %}
// Deploy the Elastic Stateless Processing Unit
ProcessingUnit pu = gsm.deploy(
	new ElasticStatelessProcessingUnitDeployment("servlet.war")
           .memoryCapacityPerContainer(4,MemoryUnit.GIGABYTES)
           //initial scale
           .scale(
                new ManualCapacityScaleConfigurer()
         	.numberOfCpuCores(10)
         	.create())
);
{% endhighlight %}

# Hot Deploy

{%refer%}
The [XAP Hot Deploy](/sbp/xap-hot-deploy.html) tool allows business logic running as a PU to be refreshed (rolling PU upgrade) without any system downtime and data loss. The tool uses the hot deploy approach , placing new PU code on the GSM PU deploy folder and later restart each PU instance.

{%endrefer%}

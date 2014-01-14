---
layout: post
title:  WAN Gateway Master Slave Replication
categories: SBP
parent: wan-based-deployment.html
weight: 200
---
s

{% tip %}
**Summary:** {% excerpt %}WAN Master-Slave replication example{% endexcerpt %}<br/>
**Author**: Ali Hodroj, Senior Solutions Architect, GigaSpaces<br/>
**Recently tested with GigaSpaces version**: XAP 9.6<br/>

{% toc minLevel=1|maxLevel=1|type=flat|separator=pipe %}

{% endtip %}

# Overview

The WAN Gateway provides a simple way of creating a master-slave topology enabling data from one XAP site to be replicated to one or more remote sites. For instance, given three clusters in New York, London, and Hong Kong, with New York being the master and the remaining two acting as slaves, any updates to the New York space will propagate to both London and Hong Kong asynchronously. The sample processing units and configuration provided below are intended as an example of implementing a single-master/multi-slave topology across three sites: New York (US), London (GB), and Hong Kong (HK) where each site has an independent cluster and a Gateway.
![WAN_masterslave.png](/attachment_files/sbp/WAN_masterslave.png)

The demo is configured to start three space instances across three clusters. While the three clusters run on your local machine, they are demarcated by zones and different lookup service ports as follows:

{: .table .table-bordered}
| Gateway/Space | Zone | Lookup Service Port |
|:--------------|:----:|:-------------------:|
| wan-gateway-HK | HK | 4166 |
| wan-space-HK | HK | 4166 |
| wan-gateway-US | US | 4266 |
| wan-space-US | US | 4266 |
| wan-gateway-GB | GB | 4366 |
| wan-space-GB | GB | 4366 |

The internal architecture of the setup includes a clustered space and a Gateway, such that the master site (US) only configures delegators while the slave sites (GB, HK) only configure sinks (click the thumbnail to enlarge):

[<img src="/attachment_files/sbp/WAN_masterslave_arch.png" width="140" height="100">](/attachment_files/sbp/WAN_masterslave_arch.png)


As a result of this topology setup, the following scenario will take place once updates are written to the New York space:

1.	All updates performed on the New York cluster are sent to local delegators for London and Hong Kong
2.	London and Hong Kong sinks will receive the updates asynchronously
3.	London and Hong Kong sinks apply the updates on their local cluster

# Configuring Master-Slave Replication

The master-slave topology configuration is simply implemented through delegators on the master (New York) and a sink on each slave (London, Hong Kong). In this case, New York's site will be the active site while London and  Hong Kong will be the passive sites. While the slave sites are passive,  this does not necessarily mean that no work is done in these sites. However, in  terms of replication over the WAN, these sites should not replicate to  the other sites and usually should not alter data replicated from other  sites because it may cause conflicts:

{% inittab %}

{% tabcontent New York Space %}

{% highlight xml %}
<?xml version="1.0" encoding="UTF-8"?>
<beans xmlns="http://www.springframework.org/schema/beans"
	xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance" xmlns:context="http://www.springframework.org/schema/context"
	xmlns:os-core="http://www.openspaces.org/schema/core" xmlns:os-events="http://www.openspaces.org/schema/events"
	xmlns:tx="http://www.springframework.org/schema/tx" xmlns:os-remoting="http://www.openspaces.org/schema/remoting"
	xmlns:os-gateway="http://www.openspaces.org/schema/core/gateway"
	xsi:schemaLocation="http://www.springframework.org/schema/beans http://www.springframework.org/schema/beans/spring-beans.xsd
       http://www.springframework.org/schema/context http://www.springframework.org/schema/context/spring-context.xsd
       http://www.openspaces.org/schema/core http://www.openspaces.org/schema/{%latestxaprelease%}/core/openspaces-core.xsd
       http://www.springframework.org/schema/tx http://www.springframework.org/schema/tx/spring-tx-2.5.xsd
       http://www.openspaces.org/schema/events http://www.openspaces.org/schema/events/openspaces-events.xsd
       http://www.openspaces.org/schema/core/gateway http://www.openspaces.org/schema/{%latestxaprelease%}/core/gateway/openspaces-gateway.xsd
       http://www.openspaces.org/schema/remoting http://www.openspaces.org/schema/remoting/openspaces-remoting.xsd">
	<context:annotation-config></context:annotation-config>
	<tx:annotation-driven transaction-manager="transactionManager" />

	<os-core:distributed-tx-manager id="transactionManager" />

	<os-events:annotation-support />

	<os-core:space id="space" url="/./wanSpaceUS" gateway-targets="gatewayTargets" />
	<os-core:giga-space id="gigaSpace" space="space" />

	<os-core:giga-space-context />

	<os-gateway:targets id="gatewayTargets"		local-gateway-name="US">
		<os-gateway:target name="GB" />
		<os-gateway:target name="HK" />
	</os-gateway:targets>
</beans>

{% endhighlight %}

{% endtabcontent %}

 

{% tabcontent New York Gateway %}

{% highlight xml %}

<?xml version="1.0" encoding="UTF-8"?>
<beans xmlns="http://www.springframework.org/schema/beans"
       xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
       xmlns:os-gateway="http://www.openspaces.org/schema/core/gateway"
       xsi:schemaLocation="http://www.springframework.org/schema/beans
       http://www.springframework.org/schema/beans/spring-beans-3.0.xsd
       http://www.openspaces.org/schema/core/gateway
       http://www.openspaces.org/schema/{%currentversion%}/core/gateway/openspaces-gateway.xsd">

    <os-gateway:delegator id="delegator" local-gateway-name="US" gateway-lookups="gatewayLookups">
        <os-gateway:delegation target="GB" />
        <os-gateway:delegation target="HK" />
    </os-gateway:delegator>

    <os-gateway:lookups id="gatewayLookups">
        <os-gateway:lookup gateway-name="US" host="localhost" discovery-port="10768" communication-port="7000"/>
        <os-gateway:lookup gateway-name="GB" host="localhost" discovery-port="10769" communication-port="8000"/>
        <os-gateway:lookup gateway-name="HK" host="localhost" discovery-port="10770" communication-port="9000"/>
    </os-gateway:lookups>

</beans>

{% endhighlight %}

{% endtabcontent %}

{% tabcontent London Space %}

{% highlight xml %}

<?xml version="1.0" encoding="UTF-8"?>
<beans xmlns="http://www.springframework.org/schema/beans"
	xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance" xmlns:context="http://www.springframework.org/schema/context"
	xmlns:os-core="http://www.openspaces.org/schema/core" xmlns:os-events="http://www.openspaces.org/schema/events"
	xmlns:tx="http://www.springframework.org/schema/tx" xmlns:os-remoting="http://www.openspaces.org/schema/remoting"
	xmlns:os-gateway="http://www.openspaces.org/schema/core/gateway"
	xsi:schemaLocation="http://www.springframework.org/schema/beans http://www.springframework.org/schema/beans/spring-beans.xsd
       http://www.springframework.org/schema/context http://www.springframework.org/schema/context/spring-context.xsd
       http://www.openspaces.org/schema/core http://www.openspaces.org/schema/{%latestxaprelease%}/core/openspaces-core.xsd
       http://www.springframework.org/schema/tx http://www.springframework.org/schema/tx/spring-tx-2.5.xsd
       http://www.openspaces.org/schema/events http://www.openspaces.org/schema/events/openspaces-events.xsd
       http://www.openspaces.org/schema/core/gateway http://www.openspaces.org/schema/{%latestxaprelease%}/core/gateway/openspaces-gateway.xsd
       http://www.openspaces.org/schema/remoting http://www.openspaces.org/schema/remoting/openspaces-remoting.xsd">
	<context:annotation-config></context:annotation-config>
	<tx:annotation-driven transaction-manager="transactionManager" />

	<os-core:distributed-tx-manager id="transactionManager" />

	<os-events:annotation-support />

	<os-core:space id="space" url="/./wanSpaceGB" />
	<os-core:giga-space id="gigaSpace" space="space" />

	<os-core:giga-space-context />

</beans>

{% endhighlight %}

{% endtabcontent %}

{% tabcontent London Gateway %}

{% highlight xml %}
<?xml version="1.0" encoding="UTF-8"?>
<beans xmlns="http://www.springframework.org/schema/beans"
       xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
       xmlns:os-gateway="http://www.openspaces.org/schema/core/gateway"
       xsi:schemaLocation="http://www.springframework.org/schema/beans
       http://www.springframework.org/schema/beans/spring-beans-3.0.xsd
       http://www.openspaces.org/schema/core/gateway
       http://www.openspaces.org/schema/{%currentversion%}/core/gateway/openspaces-gateway.xsd">

    <os-gateway:sink id="sink" local-gateway-name="GB" gateway-lookups="gatewayLookups"
                     local-space-url="jini://*/*/wanSpaceGB">
        <os-gateway:sources>
            <os-gateway:source name="US"/>
        </os-gateway:sources>
    </os-gateway:sink>

    <os-gateway:lookups id="gatewayLookups">
        <os-gateway:lookup gateway-name="US" host="localhost" discovery-port="10768" communication-port="7000"/>
        <os-gateway:lookup gateway-name="GB" host="localhost" discovery-port="10769" communication-port="8000"/>
        <os-gateway:lookup gateway-name="HK" host="localhost" discovery-port="10770" communication-port="9000"/>
    </os-gateway:lookups>

</beans>

{% endhighlight %}

{% endtabcontent %}

{% tabcontent Hong Kong Space %}

{% highlight xml %}
<?xml version="1.0" encoding="UTF-8"?>
<beans xmlns="http://www.springframework.org/schema/beans"
	xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance" xmlns:context="http://www.springframework.org/schema/context"
	xmlns:os-core="http://www.openspaces.org/schema/core" xmlns:os-events="http://www.openspaces.org/schema/events"
	xmlns:tx="http://www.springframework.org/schema/tx" xmlns:os-remoting="http://www.openspaces.org/schema/remoting"
	xmlns:os-gateway="http://www.openspaces.org/schema/core/gateway"
	xsi:schemaLocation="http://www.springframework.org/schema/beans http://www.springframework.org/schema/beans/spring-beans.xsd
       http://www.springframework.org/schema/context http://www.springframework.org/schema/context/spring-context.xsd
       http://www.openspaces.org/schema/core http://www.openspaces.org/schema/{%latestxaprelease%}/core/openspaces-core.xsd
       http://www.springframework.org/schema/tx http://www.springframework.org/schema/tx/spring-tx-2.5.xsd
       http://www.openspaces.org/schema/events http://www.openspaces.org/schema/events/openspaces-events.xsd
       http://www.openspaces.org/schema/core/gateway http://www.openspaces.org/schema/{%latestxaprelease%}/core/gateway/openspaces-gateway.xsd
       http://www.openspaces.org/schema/remoting http://www.openspaces.org/schema/remoting/openspaces-remoting.xsd">
	<context:annotation-config></context:annotation-config>
	<tx:annotation-driven transaction-manager="transactionManager" />

	<os-core:distributed-tx-manager id="transactionManager" />

	<os-events:annotation-support />

	<os-core:space id="space" url="/./wanSpaceHK" />
	<os-core:giga-space id="gigaSpace" space="space" />

	<os-core:giga-space-context />
	<os-remoting:annotation-support />

</beans>
{% endhighlight %}

{% endtabcontent %}

{% tabcontent Hong Kong Gateway %}

{% highlight xml %}
<?xml version="1.0" encoding="UTF-8"?>
<beans xmlns="http://www.springframework.org/schema/beans"
       xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
       xmlns:os-gateway="http://www.openspaces.org/schema/core/gateway"
       xsi:schemaLocation="http://www.springframework.org/schema/beans
       http://www.springframework.org/schema/beans/spring-beans-3.0.xsd
       http://www.openspaces.org/schema/core/gateway
       http://www.openspaces.org/schema/{%currentversion%}/core/gateway/openspaces-gateway.xsd">

    <os-gateway:sink id="sink" local-gateway-name="HK" gateway-lookups="gatewayLookups"
                     local-space-url="jini://*/*/wanSpaceHK">
        <os-gateway:sources>
            <os-gateway:source name="US"/>
        </os-gateway:sources>
    </os-gateway:sink>

     <os-gateway:lookups id="gatewayLookups">
        <os-gateway:lookup gateway-name="US" host="localhost" discovery-port="10768" communication-port="7000"/>
        <os-gateway:lookup gateway-name="GB" host="localhost" discovery-port="10769" communication-port="8000"/>
        <os-gateway:lookup gateway-name="HK" host="localhost" discovery-port="10770" communication-port="9000"/>
    </os-gateway:lookups>

</beans>

{% endhighlight %}

{% endtabcontent %}

{% endinittab %}

# Installing and Running the Example

1. Download the [WAN_Replication_MasterSlave.zip](/attachment_files/sbp/WAN_Replication_MasterSlave.zip) archive. It includes two folders: **deploy** and **scripts**.
2. Please extract the file and and copy the content of the **deploy** folder into `\<GIGASPACES_HOME>\deploy` folder.
3. Extract the `scripts` folder to an arbitrary location and edit the `setExampleEnv.bat/sh` script to include correct values for `NIC_ADDR` as the machine IP and `JSHOMEDIR` as the GigaSpaces root folder location.

The `scripts` folder contains the necessary scripts to start the [Grid Service Agent]({%latestjavaurl%}/service-grid.html#gsa) for each cluster, in addition to a deploy script `deployAll.bat/sh` which will be used to automate the deployment of all three gateways and space instances. This will allow you to run the entire setup on one machine to simplify testing. Here are the steps to run the example:

1. Run `startAgent-GB.bat/sh` or to start GB site.
2. Run `startAgent-HK.bat/sh` to start HK site.
3. Run `startAgent-US.bat/sh` to start US site.
4. Run `deployAll.bat/sh` file to deploy all the processing units listed above.

# Viewing the Clusters

- Start the GigaSpaces Management Center and configure the appropriate lookup groups through the "Group Management" dialog.
- Once all clusters are up and running, you will need to enable the relative groups:
![group_management_dialog.jpg](/attachment_files/sbp/group_management_dialog.jpg)

Check to enable all three advertised groups for each site:
![groups_selection_dialog.jpg](/attachment_files/sbp/groups_selection_dialog.jpg)

As a result, you should see the service grid components for each site displayed under the "Hosts" tree as follows:

[<img src="/attachment_files/sbp/masterslave_hosts_view.png" width="140" height="100">](/attachment_files/sbp/masterslave_hosts_view.png)



Once The deployAll.bat/sh script finishes running, you should be able to see all three sites deployed as follows:
[<img src="/attachment_files/sbp/pu_deployments.jpg" width="140" height="100">](/attachment_files/sbp/pu_deployments.jpg)


If you are using the GS-WEBUI, you can also view the site topology through the "Data Grids > Gateways" view as the following:
[<img src="/attachment_files/sbp/webui_gw_topology.png" width="140" height="100">](/attachment_files/sbp/webui_gw_topology.png)

# Testing Master-Slave Replication

You can test the setup by using the [benchmark utility]({%latestjavaurl%}/benchmark-view---gigaspaces-browser.html) comes with the GS-UI. Select the US Benchmark icons and click Start to begin writing objects to the space:
[<img src="/attachment_files/sbp/masterslave_space_write.png" width="140" height="100">](/attachment_files/sbp/masterslave_space_write.png)


Click the Spaces icon on the Space Browser Tab to get a global view of all spaces. As objects are being written, you should see replication occurring across both HK and GB sites until there are 5000 objects in each space:
[<img src="/attachment_files/sbp/masterslave_space_count.png" width="140" height="100">](/attachment_files/sbp/masterslave_space_count.png)



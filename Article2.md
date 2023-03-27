<!-- @format -->

# How to Strangle your Legacy System: A Study Case - Part II

In the first article from this series, we discussed why understanding the business process is important to define the system boundaries and spend money and time modernizing features that stand the company out among its competitors, and most important: avoid dealing with too many things at the same time.

In this second article, we will discuss the current architecture state and how to make the data in the legacy database running in on-premise infrastructure available from a cloud environment.

1 - Given the business needs, the processes, and the rules that belong to the business, we will create a model using Domain-Driven Design.

**2 - With the picture of the actual architecture we will explore how to create a strategy that makes the data recorded inside the legacy system available in cloud.**

3 - In the end, we will explore how to run the services on the cloud using Azure Kubernetes Service

## The Current Architecture State

<img src="./Current Architecture State.png" alt= 'Current Architecture State'>

As usual in many companies with technology stack from the 2000s we will encounter a monolith design with a database responsible for the storage and a set of stored procedures, database triggers, and more recently a number of daemons running based on timers performing table scans and executing some business rules given the tables row's state transitions.

<img src="./Current Architecture State - Inside Server.png" alt= 'Current Architecture State - Inside Server'>

### Issues in a Modernization and Cloud Migration

Let's first discuss another important rule that we should keep in mind. Despite being simple and straightforward, this rule is the most violated in IT. For me at least it is really astonishing how IT leadership always tends to predict what the business will need in the future and develop fancy and complex things that are never used and consume a lot of time and money.

<font size='6'>_[You Aren't Gonna Need It.](https://martinfowler.com/bliki/Yagni.html)_</font>

There are not so many options to design the steps for the modernization and cloud migration project given our sacred rule: to create small-sized steps for the whole project the data available in the on-premise environment needs to be accessible from the cloud.

Given that the company also aims at the end of the project to do a complete cloud migration we will need to figure out how to transfer data from an SQL Server database to the cloud. Also, assuming that small features will be removed from the legacy system to a new cloud-based system a bi-directional sync between the data storage in the cloud and the legacy system should exist.

## What will be mordernized?

<img src="./Current Architecture State - Specialist and Customer Domains.png" alt= 'Current Architecture State - Specialist and Customer Domains'>

The diagram above conveys a shallow overview of the Specialist and Customer Domains. In a scenario like that the DBAs are struggling to vertically scale the SQL Server given a large amount of pooling and table scans happening at the same time. Besides that, we also need to mention the transactions managed by the database.

Typically, the central monolith SQL Server in a small-sized company with at least 20 years of IT development has more than 2TB of data stored. Off course, this isn't a giant database but there aren't so many options for online migration of the complete server. Azure SQL Database wouldn't be an option because the database size is greater than the [Premium Standard Tier limit of 4TB](https://learn.microsoft.com/en-us/azure/azure-sql/database/resource-limits-dtu-single-databases?view=azuresql). [Azure Hyperscale has a limit of 100TB](https://learn.microsoft.com/en-us/azure/azure-sql/database/service-tier-hyperscale-frequently-asked-questions-faq?view=azuresql) and also we should consider [Azure SQL Managed Instance that supports 16TB of data](https://techcommunity.microsoft.com/t5/azure-sql-blog/increased-storage-limit-to-16-tb-for-sql-managed-instance/ba-p/2421443?wt.mc_id=DP-MVP-4015656). The last option would be to use an [Azure SQL Server VM image](https://learn.microsoft.com/en-us/azure/azure-sql/virtual-machines/windows/sql-server-on-azure-vm-iaas-what-is-overview?view=azuresql).

Anyway, in the first moment, a [complete database migration](https://learn.microsoft.com/en-us/azure/dms/tutorial-sql-server-to-virtual-machine-online-ads) would be a big challenge for a company that lacks staff with experience in running workloads in cloud environments. A small migration of the necessary data to the cloud environment would be easier in terms of technical expertise and for sure reduce the blast radius of any possible issue.

## Target Architecture

<img src="./Target Architecture.png" alt= 'Target Architecture'>

In our fictitious scenario, we aim to modernize the Specialist and Customer Domain in terms of some Domain Events only. Off course, even in a small company, there will be a lot of extra business flows dependent upon the tables and APIs which will be migrated to the cloud environment using an asynchronous communication pattern. This is the reason that leads us in the first moment to prefer a strategy where [Change Data Capture for SQL Server](https://learn.microsoft.com/en-us/sql/relational-databases/track-changes/about-change-data-capture-sql-server?view=sql-server-ver16) and an [Azure Service Bus Topic](https://learn.microsoft.com/en-us/azure/service-bus-messaging/service-bus-queues-topics-subscriptions) are both used to send and receive events from the cloud environment respectively. There are some other options like [migrating the SQL Server to the cloud0](https://learn.microsoft.com/en-us/azure/dms/tutorial-sql-server-to-virtual-machine-online-ads) environment and using a tool like [SQL Data Sync](https://www.sqlshack.com/what-is-sql-data-sync/#:~:text=SQL%20Data%20Sync%20is%20a,on%2Dpremises%20SQL%20Server%20databases.&text=To%20synchronize%20data%20and%20the,and%20columns%20should%20be%20defined.) to keep the cloud and on-premises databases synchronized. That is a very good solution but what leads us to use CDC and a message broker to synchronize data is that this kind of architecture doesn't duplicate the SQL Server. And anyway we will need CDC in order to be notified when a change is performed over the observed tables, otherwise the old fashioned tables scan will be the single option. Something that the company wants to avoid at all costs.

## Strategies for Decoupling from the on-premise Databases

Given our previous discussion and the rule for the whole project that states to move forward in small-seized steps let's discuss the step by step guide.

### 1 - On-Premises to Cloud Connectivity

Off course, the first thing that we need to address is the connectivity between the on-premise VPN and the cloud VPN. There are three options available:

- [Virtual Network Point-to-site](https://learn.microsoft.com/en-us/azure/vpn-gateway/point-to-site-about): A Virtual Network Point-to-site connection allows us to create a secure connection between a local windows machine to the cloud VPN. That option is recommended for prototyping and, development, testing, and simulation purposes.
- [Virtual Network Site-to-site](https://learn.microsoft.com/en-us/azure/vpn-gateway/tutorial-site-to-site-portal): A site-to-site VPN allows you to create a secure connection between your on-premises site and your virtual network. The industry standard IPsec VPN is used for encryption of the package exchange between your on-premises device and the cloud gateway VPN.
- [ExpressRoute](https://learn.microsoft.com/en-us/azure/expressroute/): ExpressRoute lets you create private connections between Azure datacenters and infrastructure that’s on your premises or in a co-location environment. ExpressRoute connections do not go over the public Internet, and offer more reliability, faster speeds, lower latencies and higher security than typical connections over the Internet. For sure, this option is the most complex.

This is the first step of the whole migration project. For sure, the team will be learning how to work in a hybrid environment and we don't need to target the highest quality option just for loading a couple of tables from the on-premise SQL Server to the Cosmos DB Databases. Then, the option we choose here is the Site-to-Site VPN. Keep in mind that this is a task that will be performed by an infrastructure team and isn't any kind of rocket science but it may need some time to be completed. Consider the Point-To-Site connection for the prototyping and development phase if the setup of the Site-to-Site connection requires an amount of time that delays the target deadline for the sprint (or detailed project's step).

<img src="./Site-to-Site VPN Connection.png" alt= 'Site-to-Site VPN Connection'>

### 2 - First Load of Data from the on-premises SQL Server to the Cosmos DB Accounts

In the desired system an asynchronous communication architecture will take care of keeping the consistency between the different domains and broadcasting state transitions. But, in the first moment, we will need to transfer the current events, specialists, customers, and other tables present in the on-premises SQL Server to their respective data stores in the cloud environment. To solve that issue we will create a simple [Azure Data Factory pipeline to extract the data from the SQL Server, perform transformations when necessary, and load the data into Cosmos DB Database Containers](https://learn.microsoft.com/en-us/azure/data-factory/).

<img src="./Azure Data Factory Pipeline.png" alt= 'Azure Data Factory Pipeline'>

We could have gone for a more permanent solution like migrating the whole SQL Server Databases to an Azure SQL Server VM. Maybe this is something we will need to do to completely deactivate the on-premises data center. But remember: avoid unnecessary complexity and keep things simple when possible. As a starting point to only migrate some Commands and Domain Events from the Specialist and Customer domains, this is more than enough.

Following are some important documents necessary for this project's step:

- [Create and configure a self-hosted integration runtime](https://learn.microsoft.com/en-us/azure/data-factory/create-self-hosted-integration-runtime?tabs=data-factory)
- [Copy Data From Azure SQL Server](https://learn.microsoft.com/en-us/azure/data-factory/connector-sql-server?tabs=data-factory)
- [Transform data in Azure Data Factory and Azure Synapse Analytics](https://learn.microsoft.com/en-us/azure/data-factory/transform-data)
- [Copy and transform data in Azure Cosmos DB for NoSQL by using Azure Data Factory](https://learn.microsoft.com/en-us/azure/data-factory/connector-azure-cosmos-db?tabs=data-factory)

### 3 - Synchronize the on-premises SQL Server with the Azure Cosmos DB Accounts

After the initial data load from the on-premise SQL Server to the Azure Cosmos DB Databases we need to figure out how the changes executed in SQL Server tables will be notified to the cloud environment. There are many possible solutions and there is no right or wrong answer. One needs to understand the requirements and assumptions and take a best-guess option, avoiding future complex predictions about how the system will need to behave. Take current knowledge, decouple responsibilities using asynchronous communication, and define the system boundaries.

[Incrementally loading data from the database using Azure Data Factory and a watermark column](https://learn.microsoft.com/en-us/azure/data-factory/tutorial-incremental-copy-portal) would be a reasonable option. But we already have a lot of table scans happening on the database and adding new ones is something that we want to avoid. Given the previous statement, the best scenario would be to be notified when a delete, insert, and update operation happens on the tables of interest. [Change Data Capture](https://learn.microsoft.com/en-us/sql/relational-databases/track-changes/about-change-data-capture-sql-server?view=sql-server-ver16) with [Debezium](https://debezium.io/) suits well in a scenario like that.

<img src="./Change Data Capture.png" alt= 'Change Data Capture'>

[This github repo](https://github.com/azure-samples/azure-sql-db-change-stream-debezium/tree/master/) has an excellent step-by-step guid about the procedures to setup Change Data Capture Streaming Using Debezium and Azure Event Hub.

### 4 - How to Send Domain Events from the Cloud Environment to the on-premises systems?

With the steps described above we guarantee that the events created in the on-premises data center will be available asynchronously to the cloud applications. What about the events created in the cloud system? To address that issue Azure Cosmos DB has a feature called [Change Feed](https://learn.microsoft.com/en-us/azure/cosmos-db/change-feed). It works similarly to the Change Data Capture available for many Database platforms like PostgreSQL, Cassandra, etc.

<img src="./Cosmos DB Change Feed.png" alt= 'Cosmos DB Change Feed'>

[Azure functions will be triggered after insert/update operations over the observed containers](https://learn.microsoft.com/en-us/azure/cosmos-db/nosql/change-feed-functions). The functions will also be responsible for sending messages to an Azure Service Bus topic. In the on-premises environment, some applications like a windows service will subscribe to the topic and be notified. With this architecture, we can asynchronously keep the systems running in the data center receiving the state transitions happening in the cloud environment.

## Conclusion

In this second article from our series, we discussed strategies for kicking off the modernization project with a good level of decoupling and also avoiding unnecessary complexities. For sure, this isn't the final target architecture for the whole project. But in my personal experience, this isn't something that the company is able to figure out at the first moment. The beauty of agile methodologies is that the knowledge is constructed over incremental phases, not in a single planning phase that usually takes months or years to be completed. Start small and try to isolate responsibilities using asynchronous communication whenever it is possible.

In the next article, I will discuss why the company chose Azure Kubernetes Service to run their applications in the cloud even with other less complex options available like Azure App Services.

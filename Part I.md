<!-- @format -->

# How to Strangle your Legacy System: A Study Case - Part I

<img src="./strangler_tree.jpg" alt= 'A tree strangling another'  height="500px">

In comparison with other engineering areas Software is a brand-new field. For example, mining is extensively explored and developed by mankind [since approximately 40000 to 20000 thousand years ago](https://www.earthsystems.com/history-mining/#:~:text=The%20earliest%20known%20mine%20for,10%2C000%20to%207%2C000%20years%20ago.). What it brings for us, IT professionals is a lot of opportunities and things to explore, like the vast amount of companies that have grown since the last subprime crisis in 2008 when the interests rates became very low and the money available was invested in monolith systems which do not suit well with the necessity of rapidly release new features in production and implement changes without disrupting other parts of the big picture. With all that a common scenario is a company that whats to strangle its legacy system and migrate to a cloud platform.

In this article, we will explore some of the common mistakes that I have seen during some modernization processes and how to address the contexts that lead to those mistakes. The fictitious scenario used here belongs to an also fictitious company that provides maintenance services for truck drivers linking them with well-suited specialists given the kind of failure the driver is facing.

There are three phases that this article series will follow:

**1 - Given the business needs, the processes, and the rules that belong to the business, we will create a model using Domain-Driven Design.**

2 - With the picture of the actual architecture we will explore how to create a strategy that makes the data recorded inside the legacy system available in cloud.

3 - In the end, we will explore how to run the services on the cloud using Azure Kubernetes Service and Pricing Comparisons for overall architecture

In this first article we will discuss the first topic.

Something important to mention: this is a how-to article, then there will be a lot of code, tutorials to follow, and deliverables here.

About the fictitious scenario: the company has most of its legacy system developed using Microsoft technologies like .NET framework, SQL Server, IIS Server, Windows Server, etc, then the cloud platform already selected by the board of executives is Azure (doesn't mean because your stack is Microsoft based you must use Azure - We can explore the AWS platform in a next article).

# The Business Needs and the Domain-Driven Design model

Despite the next sections about technical solutions for sure will become the ones that people usually are more concerned about, the truth is that most companies are struggling to modernize and migrate to a cloud infrastructure because the modernization process is carried out using the same patterns that created the mess. No attention is given to the definition of system boundaries or a clear definition of the domain events. How the whole system will be decoupled if no one knows what should be decoupled from what? And given our present state of the IT architecture the first mistake will be presented:

<font size='6'>_Don't start by creating a new table schema._</font>

There is no single rule about how we should carry out a complex undertaking like the migration from an on-premise monolithic architecture to a reactive microservice architecture on a cloud infrastructure. Anyway, for sure one should not think that a new table schema is a good starting point.

In this article, we will focus on creating a Domain-Driven Design model. I will not dive into the important steps that for sure people involved with the design of the modernization strategy should take care of, like the business model canvas for the current state and the desired state. Having a clear understanding of the business is critical to a successful domain discovery.

Assuming that we already possess a foundational understanding of the business strategy the target architecture design process can begin.

## The Use Case

We will start by describing one important use case using a UML Behavioral Diagram:

<img src="./Use Cases UML.drawio.png" alt= 'Use case UML Diagram'  height="500px">

As already mentioned, the business of the fictitious company is to assign the best specialist taking into consideration factors like proximity to the failure event, the expertise level of the specialist (e.g. rate within the platform), etc. One important factor here is to be aware that every model is an abstraction. It will never cover all details about the real system being modeled.

If the desire to map the whole business process exists (something that I strongly recomend) there is no better tool available than Business Process Model and Notation, aka BPMN. In next picture we can see a BPMN diagram constructed using [Camunda Modeler](https://camunda.com/download/modeler/). Something we can dicuss in another article is the power of State Machine Automation Tools. Every business process can be modeled as a State Machine. To avoid reiventing the wheel every time the team needs to deal with a complex business process, tools like Camunda became very popular among Financial and Logistics companies.

<img src="./Use Case BPMN.png" alt= 'Use case BPMN Diagram' >

By using a BPMN diagram we can map with a great level of detail the alternative paths that a failure event can pass during the process.

## Event Storming Sessions

Event Storming is an attractive tool when both domain experts and developers need to be engaged. Our event storming board can be seen in the following picture:

Now it's time to derive our Domain-Driven Desing diagram.

### Domain Events

<img src="./EventStorming - 1.png" alt= 'Event Storming: Domain Events' >

In the first step, we derive our domain events. Here I'm using the same convention from [Vaughn Vernon's book Implementing Domain-Driven Design](https://www.amazon.com.br/Implementing-Domain-Driven-Design-Vaughn-Vernon/dp/0321834577/ref=asc_df_0321834577/?tag=googleshopp00-20&linkCode=df0&hvadid=379787788238&hvpos=&hvnetw=g&hvrand=461375848233545553&hvpone=&hvptwo=&hvqmt=&hvdev=c&hvdvcmdl=&hvlocint=&hvlocphy=1001566&hvtargid=pla-451193117745&psc=1) where the Domain Events are represented using orange sticky notes.
Something really important here is to understand that the main focus is to emphasize the business process, not the technical requirements, data structures, and stuff like that. For example, for sure, the specialist needs to login into some application and the operator also has some back office app. But those details don't play a big role in the business process, and then, an event named like SpecialistLogged can be omitted at this phase. Also, remember that Domain Events should be a verb stated in the past tense.
Another point here is that sometimes a Domain Event triggers a process or another important application execution considered as important by the Domain Experts. In those cases, we are using lilac sticky notes to map those processes.

### Create the Commands

<img src="./EventStorming - 2.png" alt= 'Event Storming: Commands' >

After creating the domain events and taking into consideration the Business Process unfolding, it's time to derive the Commands that cause each Domain Event. The Commands are represented in the picture above as blues sticky notes. Don't be afraid to use a different convention. The most important here is to represent in a reasonable model the important things that belong to the process. To give an example, maybe there is a situation where a command is executed by a user or is the outcome of a process (lilac sticky notes). The yellow small sticky notes represent some important roles. If developers or Domain Experts feel the need to map them that is a useful way to represent them on the diagram.

### Aggregates, Entities and Value Objects

<img src="./EventStorming - 3.png" alt= 'Event Storming: Aggregates' >

The goal of the third phase is to map the Aggregates/Entity where the Command is executed and the Domain Event is emitted. As also stated in [Vaughn Vernon's book Domain-Driven Design Distilled](https://www.informit.com/store/domain-driven-design-distilled-9780134434421?ranMID=24808), the term Aggregates can be confusing even for technical people.

To clarify what is an Aggregate we should first understand what is an Entity. An Entity is an object that has a unique identity. Even if there exists another Entity whose property values are equal to another Entity, those Entities will be different because the identity assigned to them is unique.

A Value Object is an immutable type that is distinguishable only by the state of its properties. That means, in contrast with an Entity, which has a unique identifier and remains distinct even if its properties are identical, two Value Objects with the same properties will be considered equal.

And finally, we can explain what an Aggregate is. An Aggregate is a collection of Entities and Values Objects which, given constraints from the Business Process (and please, not technical ones), have a transactional relationship. What that means: the Entities and Value Objects have a relationship. In practice, this doesn't translates to the fact that in an update or delete operation a transaction needs to be managed by the database. It only means that, from the business point of view, those Entities need to be consistent.

As an example, consider an e-commerce domain that has concepts for Orders, which have multiple OrderItems, each of which refers to some quantity of Products being purchased. Adding and removing items to an Order should be controlled by the Order - parts of the application shouldn't be able to reach out and create an individual OrderItem as part of an Order without going through the Order. Deleting an Order should delete all of the OrderItems that are associated with it. So, Order makes sense as an aggregate root for the Order - OrderItem group.

I will not dive into the details of Value Objects, Entities, and Aggregates. Those concepts were first described in [Evans' Domain-Driven Design book](https://www.amazon.com/exec/obidos/ASIN/0321125215/domainlanguag-20). The following articles from Vaughn Vernon are also excellent references describing examples of how Entities and Value Objects can be clustered into Aggregates: [Effective Aggregate Design Part I: Modeling a Single Aggregate](https://www.dddcommunity.org/wp-content/uploads/files/pdf_articles/Vernon_2011_1.pdf), [Effective Aggregate Design Part II: Making Aggregates Work Together](https://www.dddcommunity.org/wp-content/uploads/files/pdf_articles/Vernon_2011_2.pdf), [Effective Aggregate Design Part III: Gaining Insight Through Discovery](https://www.dddcommunity.org/wp-content/uploads/files/pdf_articles/Vernon_2011_3.pdf).

### The First Boundaries

<img src="./EventStorming - 4.png" alt= 'Event Storming: Domains' >

After the discussion about Domain Events, Commands, Processes, and their relationships, it's time to start drafting the first limits between the clusters of elements present in the diagram. This is one the most important steps because here we define what will be present in each system responsible for the business capacity needed, and also the consistency level after the unfolding of some Domain Event. Usually, people tend to at first moment say that everything should have a strong consistency level. What does that mean? I will try to explain with an example: after the Domain Event SpecialistAssigned is emitted, the Specialist must be notified immediately. To smooth things try to say that the Specialist will be notified after 24 hours or one week. It makes people exercise the idea of having a system based on events and asynchronous communication.

Also, remember that the aim here is to define the boundaries between the different phases of the business process modeled. Having those boundaries clearly stated tends to make people understand the steps needed to complete a modernization project and cloud migration and to isolate responsibilities. And again, the modeling of a business process is something that is never over. It should change and evolve as the business and technical expertise evolves. If there is some point fuzzy and clunky while defining the boundaries for the domains don't spend so much time discussing it. Take a deep breath and come back for the sessions next week.

### What will be modernized first?

<img src="./Core Domain Charts.png" alt= 'Event Storming: Core Domain Chart' >

As always in IT design tasks, there is no wrong or correct answer. For sure, some decisions tend to phase things out and lower the risk. But also, modernizing an important domain (Core Domain) will stand out the business among its competitors.

After looking at the [Core Domain Chart](https://github.com/ddd-crew/core-domain-charts) a decision was made: the Specialist and Customer Domains will be modernized first because it is something that will differentiate the business the most, despite their complexity.

## Conclusion

With that said we covered the basics about how we can avoid getting lost in a long project that usually wants to change the airplane engine while still flying. Modernizing a legacy system that supports all the business processes for a company has a lot of pitfalls. Defining clearly what will be the steps is a key factor to avoiding tackling everything at the same time.

As already mentioned, there is no perfect model. Always something will need to be revised and updated. The great achievement is that evolving the software after having clearly defined the boundaries and contracts between the domains won't be a project more similar to the process of evolving a piece of hardware where things are tightly coupled.

In an article like this one, there is no room for moving forward in the process of creating the link between the DDD model and the software, like defining the shape of events, API's, application architecture, etc. For a deep understanding of those phases I strongly recommend [Vaughn Vernon's book IDD](<(https://www.amazon.com.br/Implementing-Domain-Driven-Design-Vaughn-Vernon/dp/0321834577/ref=asc_df_0321834577/?tag=googleshopp00-20&linkCode=df0&hvadid=379787788238&hvpos=&hvnetw=g&hvrand=461375848233545553&hvpone=&hvptwo=&hvqmt=&hvdev=c&hvdvcmdl=&hvlocint=&hvlocphy=1001566&hvtargid=pla-451193117745&psc=1)>). For C# developers [Scott Millet's and Nick Tune's book Patterns, Principles, and Practices of Domain-Driven Design](https://www.amazon.com.br/Patterns-Principles-Practices-Domain-Driven-Design/dp/1118714709/ref=asc_df_1118714709/?tag=googleshopp00-20&linkCode=df0&hvadid=379733272930&hvpos=&hvnetw=g&hvrand=11004744162210465982&hvpone=&hvptwo=&hvqmt=&hvdev=c&hvdvcmdl=&hvlocint=&hvlocphy=1001566&hvtargid=pla-452302156786&psc=1) is the best alternative. And finally, for Java developers Vijay Nair's book [Practical Domain-Driven Design in Enterprise Java](https://www.google.com.br/books/edition/Practical_Domain_Driven_Design_in_Enterp/eq6tDwAAQBAJ?hl=en&gbpv=1&printsec=frontcover) is an excellent reference. Those books will give you a practical standpoint of view on how Domain-Driven Design creates the connection between the Business Process needs and the IT system that supports those needs.

And last but not least:

<font size='6'>_Don't start by creating a new table schema._</font>

You need to first map the business needs and understand the boundaries between the multiple phases.

In the next article, we will explore the current system architecture design and how the data available in the on-premises legacy system can be made available in a cloud Platform.

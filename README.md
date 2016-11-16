# Cloud Cost Notation
v1.0.0

Cloud Cost Notation (CCN) is a method of communicating the relative cost of cloud-based computing.

> *"Big-O notation for computing cost."*

This document focuses on Amazon Web Services (AWS), but these ideas can also be applied to other cloud-computing platforms. The Overview section is intended for a wide audience. The remainder of the document discusses technical details and recommendations.

###### Contents:
* [Overview](#overview)
* [Measuring droplets](#measuring-droplets)
* [AWS recommendations](#aws-recommendations)

## Overview

### Problem
Often, an organization is interested in knowing how much a cloud-based solution will cost them before putting it into production. Estimating costs becomes more difficult as the number of different cloud-computing products in-use increases.

Common causes for inaccuracy in predicting cloud-computing cost:
* The [AWS pricing model](https://aws.amazon.com/pricing/services/) is too complex for a human to easily remember.
* Estimating costs is a slow process that involves [many calculations](http://calculator.s3.amazonaws.com/index.html).
* Exotic cloud services have equally [exotic pricing](https://aws.amazon.com/lambda/pricing/) models.
* [Prices change](https://aws.amazon.com/blogs/aws/category/price-reduction/) over time.
* Prices vary [between regions](https://aws.amazon.com/sqs/pricing/).
* Cost scales with usage for each solution differently.
* [Pre-purchasing reserved cloud services](https://aws.amazon.com/ec2/pricing/reserved-instances/pricing/) can provide a cost savings of up to 75%.

An accurate price estimate in dollars does not necessarily answer all of the implied questions that the person asking for an estimate was hoping to answer. For example:
* *"How does the cost of this new solution compare to our existing solutions?"*
* *"Is the system architecture appropriate for this class of problem?"*
* *"Is it worth iterating on this solution to reduce costs? Which improvements should be made first?"*
* *"Do costs scale well under increased load? How do costs scale across multiple regions?*


### Solution
One solution to these problems is to stop measuring cloud cost in terms of dollars and define a new unit of measurement. A new unit of measurement can be defined with useful properties and is free from the assumptions people have about existing units. The trade-off is some overhead associated with learning a new measurement.

> This approach is similar to that of [big-O notation](https://en.wikipedia.org/wiki/Big_O_notation), which is used to compare algorithms in terms of complexity, instead of the number of seconds they take to run sample data.

#### Droplets
Let us a define a new unit of measurement for *cloud cost*, called a `droplet`.

* `One droplet is the cost of running a single t2.micro EC2 instance in AWS region us-east-1 for one month of the year 2016.`

> *"A droplet is the cost of running a t2.micro instance."*

##### Droplet notation
* 1 droplet = `1d`
* 100 droplets = `100d`
* Triple the amount of 5 droplets = `(3 * 5d = 15d)`

##### Droplet properties
* Droplets measure the **upper bound** of monthly cloud computing cost. (Worst-case scenario.)

* Droplets are a **course unit** of measurement. (How big, not exact size.)

* Droplets are **linearly proportional**. (`300d` is one hundred times more than `3d`.)

* Droplets are computed with a simple **lookup table**.

* Droplets are **constant**. (Droplet cost is the same between regions, when currencies change value, for on-demand and reserved resources, etc.)

### Case study
The technical details on how to measure and use `droplets` can be found in the next section. Without understanding the finer details, droplets allow us to reason about the cost of cloud-computing solutions.

#### Relative cost
Suppose we have two services in production. Even with a billing history, it is difficult to measure exactly how much each one costs because our AWS bill is broken down by AWS product, not by how the AWS products are arranged into our services.

> Note: The numbers in this example are real.

In under 5 minutes, by using the droplets lookup table and elastic beanstalk monitoring pages, the following information can be collected:

* Service A handles `600k requests/day`.
* Service A costs `24d` (`18d` for ELB, `6d` for RDS)
* Service B handles `200k requests/day`.
* Service B costs `23d` (`20d` for ELB, `3d` for RDS)
* From a basic understanding of each service's function we also know that Service A's tasks require more computation.

Based on this information, we can determine that both services cost roughly the same amount. Additionally, we can determine that Service B is likely over-provisioned.

With a quick look at the monitoring history we see that a simple way to reduce costs would be to reduce Service B's four t2.medium instances to four t2.small instances. (A savings of `8d` and a great return on a small time investment.)

We also ask a developer to investigate Service B's code. He determines it would be possible to shrink Service B's instances again to four t2.micro instances for an additional savings of `4d`. This change would be low-risk and require three days to complete. We quickly determine that three days of developer time is more expensive than what the company pays for an entire year of `4 droplets` in the cloud.


## Measuring droplets
The tone of this section is intentionally less conversational.

### Unit conversion
To determine the droplet cost of a cloud product, use the following conversion rates. **Always round up.**

|Cost  |Item                          |Notes |
|------|------------------------------|------|
|`1d`  |100 DynamoDB read capacity    |To calculate, multiply write capacity by 5 and add to read capacity.^ |
|`1d`  |100GB of S3 storage           ||
|`1d`  |10 million SQS messages       ||
|`2d`  |Elastic load balancer         |Often created by elastic beanstalk. |
|`1d`  |micro EC2 instance            ||
|`2d`  |small EC2 instance            ||
|`4d`  |medium EC2 instance           ||
|`8d`  |large EC2 instance            ||
|`16d` |xlarge EC2 instance           |For [n]xlarge instances multiply n*16d. |
|`+50%`|RDS and Elasticache instances |Multiply the size of standard EC2 instances by 1.5.^^ |

> ^For example, 100 DynamoDB read capacity and 60 DynamoDB write capacity would be 4d. (100 reads + 5*60 writes = (100 + 300) reads = 400 reads = 4d)

> ^^For example, an RDS or Elasticache instance of size medium is 6d. (4d * 1.5 = 6d)

The table will be expanded to include other AWS products.

#### Example calculation
Suppose a service has a cost of `11 droplets`, computed as follows.

|Cost |Item |
|-----|-----|
|`2d` |(2 elastic beanstalk load balancers) |
|`4d` |(4 t2.micro instances) |
|`1d` |(100 provisioned DynamoDB reads) |
|`3d` |(1 small elasticache instance) |
|`1d` |(1 million SQS messages) |
|==== |======================== |
|`11d`|droplets total |

Our testing shows that an increase in traffic from 3 requests/second (today) to 33 requests/second (anticipated maximum load a year from now) would require increasing the provisioned DynamoDB capacity to 980 reads (`+9d`). The number of SQS messages would increase to 10 million (`+0d`). No changes to the number of EC2 instances are required (`+0d`).

Therefore, increasing traffic to 1100% would scale the cost from `11d` to `20d`.

### Scaling notation
> Note: The scaling notation section is a work in progress.

Now that we have a unit that is easy to measure and make relative comparisons with, we can begin to reason about how an individual cloud solution will scale. This will enable us to develop expectations around what the cost curve of a highly scalable solution looks like.

#### Definitions
Let's start by defining some useful points of reference.
* **minimal footprint (`0x`):** The number of droplets it takes to run this solution for one user.

* **expected load (`ex`):** The number of droplets it takes to run this solution for the expected number of users.

* **nx load (`nx`):** The number of droplets it takes to run this solution for `n` times the expected number of users.

* **Cloud Cost Notation (`CCN(...)`):** A comma-separated list of elements comprised of a load prefix and droplet cost describing how a solution scales.
  * `CCN(0x:5d, ex:20d, 1000x:12000d)`
  * `CCN(0x:2d, ex:45d)`
  * `CCN(0x:12d, ex:200d, 10x:1800d, 100x:23000d, 1000x:452000d)`

Droplets, as per their definition, are a measure of *upper bound*. Therefore, **CCN() notation describes upper bounds, not exact values.**
* The element `0x:5d` translates to: *"It will cost at most 5 times our droplet price to stand up this solution in a new region."*
* The element `ex:20d` translates to: *"At our expected load, the solution will cost at most 20 times our droplet price."*
* The element `1000x:12000d` translates to: *"At one thousand times our expected load, the solution will cost at most 12,000 times our droplet price."*

#### Additional thoughts
If this style of notation finds an audience, I would not be surprised to see a desire to abbreviate it further by assuming the presence of the minimal footprint and expected load terms. In that case, consider the following modification as an alternative:
  * `CCN(5d|20d, 1000x:12000d)`
  * `CCN(2d|45d)`
  * `CCN(12d|200d, 10x:1800d, 100x:23000d, 1000x:452000d)`

> Ideally, I would like to see scaling costs represented in the form of a function, similar to the way big-O notation represents complexity with O(n<sup>2</sup>). For example, `CCN(4d|8d, O(log n)d)`.
>
> There are challenges associated with that approach that require additional consideration. In particular, it requires generating more measurements than are required to make useful decisions about a solution. Additionally, it requires fitting a series to a curve, which is not an intuitive task for most people.
>
> The majority of cloud solutions would likely scale at a rate of O(n), varying only by their constant factor. The big-O approach feels like overkill, but may evolve into something that focuses on the constant factor and looks like this: `CCN(4d|8d, 80%)`
>
> There may be value in providing architectural guidance to developers by defining a CCN() that describes a desirable microservice minimum footprint and scaling rate.


## AWS recommendations
The following recommendations are based on personal experience. Use them as a starting point, not a set of commandments.

### Minimal footprint
A cloud service's minimal footprint is how much it costs to operate in a region with near-zero traffic. This can be ignored when building an omni-tenant service that handles all requests from a single region.

For many real-world solutions, performance and data-residency concerns lead to a service being deployed in multiple regions. Minimal footprint overhead quickly becomes important as the number of deployed services grows. `(overhead * number of services * number of regions)`

### Instance size
Instance size should be determined by the task that a solution is running. There are generally two types:
* **Bursty** workloads are often driven by users. (Web apps, continuous integration builds.)

* **Scheduled** workloads are often long-running calculations. (Sales reports, protein folding.)

Bursty workloads tend to favor smaller instances. Combined with a load balancer, these smaller instances can offer better burst performance by distributing the workload across multiple machines.

Scheduled workloads are likely to completely drain the CPU credits from t2 instances, and lend themselves to other instance types that do not operate on CPU credits. (m4, c4, x1, r3, etc.)

In terms of cost, a micro instance (`1d`) is half the price of a small instance (`2d`). Using micro instances to service web requests allows twice as many CPUs to be available for the same price. It also makes it easier to lower costs by introducing automated scaling based on dynamic load or time of day.

> *"You can't deploy half a server... but you can deploy a smaller one."*

Other constraints on instance size selection include the number of CPUs and the amount of available RAM.

### CPU credits
It is important to understand CPU credits when using burstable (t2) instances. When a burstable instance exhausts its CPU credits its performance will drastically drop **until the workload is reduced**.

A simplified definition of a CPU credit:
* `The right to use 100% of a CPU for 1 minute.`

A CPU credit can also be consumed by using 50% of a CPU for 2 minutes, or 1% of a CPU for 100 minutes, etc. CPU credits are accrued based on instance size. Unused credits are held in reserve for up to 24 hours before they expire.

Although it is not yet possible to configure Elastic Beanstalk to scale based on remaining CPU credits, it is possible to configure Cloud Watch alarms based on CPU credits. An alarm can be used to trigger another instance to spin up.

### Tiny databases
For relational data, prefer MySQL 5.6 over other RDS alternatives.
* MySQL 5.6 is compatible with Amazon Aurora, which offers a clear upgrade path to a very scalable database.

* Do not start with Aurora. The smallest Aurora instance size is a db.r3.large (`24 droplets`). The smallest MySQL instance is a db.t2.micro (`2 droplets`).

For non-relational data, prefer DynamoDB.
* Operating a DynamoDB table with a provisioning of 1 read and 1 write has a near-zero cost.

* Due to the way reads are billed, a well-designed table can return a page of results for the same price as an individual result.

* As traffic grows, consider caching tables that have a high read/write ratio with Elasticache.

* Before using secondary indices:
  * Understand how they will impact your costs. It may be better to use a relational database instead.
  * Understand the difference between local and global secondary indices.
  * Be aware that adding a local secondary index requires recreating a table; have a data migration strategy.

----

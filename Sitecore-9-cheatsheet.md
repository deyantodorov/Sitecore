# Sitecore 9 - performance optimization tips and tricks

*Platform - [Microsoft Azure](http://portal.azure.com) using Platform as a Service (PaaS)*

*Sitecore version - [Sitecore 9.1](https://dev.sitecore.net/)*

## 1. Should I follow [Sitecore XP 9.1 ARM templates – topologies and tiers](https://kb.sitecore.net/articles/713688)?

As you can see Sitecore recommends **S** services plans or **I** service plans. 

- **S** type service plan means **Standard** service plan
- **I** type service plan means **Isolated** service plan

More information about Azure App Service Plan you could find [here](https://azure.microsoft.com/en-us/pricing/details/app-service/plans/) and about pricing details [here](https://azure.microsoft.com/en-us/pricing/details/app-service/windows/).

If you check the price of **S3** in West Europe it will cost around ~$292/month, but if you would like to check with something similar to S3 you might take **I2** (~$438/month) or **I3** (~$876/month) which is a big price increase.

So I decided to take something in similar price range which is **P2v2** and will cost around ~$292/month.

What is **P**? As you can [see](https://azure.microsoft.com/en-us/pricing/details/app-service/windows/) it means PREMIUM (Enhanced performance and scale).

I'll compare **S3** vs. **P2v2**

| Plans                    | S3            | P2v2          |
| ------------------------ | ------------- | ------------- |
| Web, mobile, or API apps | Unlimited     | Unlimited     |
| Disk space               | 50 GB         | 250 GB        |
| Maximum instances        | Up to 10	   | Up to 20	   |
| Custom domain            | Supported     | Supported     |
| Auto Scale               | Supported     | Supported     |
| VPN hybrid connectivity  | Supported     | Supported     |
| Network Isolation        | No            | No            |
| Price per hour           | $0.10         | $0.20         |
| CORES                    | 4             | 2             |
| RAM                      | 7 GB          | 7 GB          |
| STORAGE                  | 50 GB         | 250 GB        |
| PRICES                   | ~$292/month   | ~$292/month   |

So what you should we take 4 cores vs. 2 cores, 50 GB storage vs. 250 GB? Let's check in more details.

In Azure Portal you could see that:

| Plans               | S3                                    | P2v2                                |
| ------------------- | ------------------------------------- | ----------------------------------- |
| ACU                 | 400 total ACU                         | 420 total ACU                       |
| Memmory             | 7 GB memory                           | 7 GB memory                         |
| Compute equivalent  | **A**-Series compute equivalent       | **Dv2**-Series compute equivalent   |
| Auto scale          | Up to **10** instances                | Up to **20** instances              |
| Staging slots       | Up to **5** staging slots             | Up to **5** staging slots           |
| Daily backups       | Backup your app **10** times daily.   | Backup your app **50** times daily. |

Next question. What is **A** series and **Dv2** series?

As Microsoft [explains](https://azure.microsoft.com/en-us/pricing/details/virtual-machines/series/)

A-Series
Entry-level economical VMs for dev/test
A-series VMs have CPU performance and memory configurations best suited for **entry level workloads like development and test**. They are economical and provide a low-cost option to get started with Azure. Av2 Standard is the latest generation of A-series VMs with similar CPU performance but more RAM per vCPU and faster disks.
Example use cases include development and test servers, low traffic web servers, small to medium databases, servers for proof-of-concepts, and code repositories.

D-Series
General purpose compute
D-series VMs feature fast CPUs, and optimal CPU-to-memory configuration making them suitable for most production workloads. **Dv2-series instances carry more powerful CPUs** and the same memory and disk configurations as the D-series.
D2-64 v3 instances are the latest hyper-threaded generation of general purpose instances and are based on the 2.3 GHz Intel XEON ® E5-2673 v4 (Broadwell) processor. They can achieve 3.5 GHz with Intel Turbo Boost Technology 2.0. The Ds-series supports Azure Premium SSDs.
Example use cases include many **enterprise-grade applications**, relational databases, in-memory caching, and analytics. The latest generations are ideal for applications that demand faster CPUs, better local disk performance or higher memories.

Final test before conclusion. Lets take [JMeter](https://jmeter.apache.org/) and prepare a test with **200** users, ramp-up period **100** seconds and see the results.

I'll run those test against Sitecore envrionment A and Sitecore environment B. For both I have the same Sitecore 9.1 XPSingle toplogy. And the results are:

| Environment   | Error %  | Throughput (how many request could handle per second) | Time to complete all tests |
| --------------| ---------| ----------------------------------------------------- | -------------------------- |
| Environment A | 1.39%    | 3.2/sec                                               | 08.55 minutes              |
| Environment B | 0.00%    | 6.2/sec                                               | 04.50 minutes              |

## 2. Modify Redis Cache configurations and Sitecore ARM templates.

## 3. Optimize cost with SQL Elastic Pool.
- http://blog.baslijten.com/to-elastic-pool-or-not-to-elastic-pool-for-sitecore-on-azure/

## 4. Reduce Azure PaaS Hosting Costs in Non-Prod Environments With Scheduled Vertical Scaling.
- https://sitecorerap.wordpress.com/2019/07/09/drastically-reduce-azure-paas-hosting-costs-in-non-prod-environments-with-scheduled-vertical-scaling/

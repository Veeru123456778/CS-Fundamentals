# Serverless Architecture:

The word serverless implies that we are not managing the server but that does not implies that we did not have the server. Server is required to run the code, but we are not managing the servers.

Suppose if the traffic that is coming to our appliation is generally not much but at some time like for 2-3 minutes the traffic shoots and spikes up and this may lead to take server down, now we may consider some solution for this:

## We monitor the highest peak load that may come to the server 

In this approach we can note down the highest load that may come to our server at any particular time and then according to that we can alloacte and increase the resources like we can increase the RAM, CPU and resources on our Server so that it can handle the peak.

Disadvantages of this:

- Because our load is not continuous and it is only for some minutes so for those minutes we can not increase our resources because it will be very costly because server will run forever and the infra cost shoots up while the request load is not that much. This is the main reason we must not choose this.


## Infrastruture as a Service: Serverless Architecture
Ex: AWS Lambda

We should have something that can be flexible and we can use as infrastructure as a service means we whenever the traffic scales up it can be automatically scales up and whenever the load on our server is not very much then it should be automatically scaled down.

Benefits for this approach:

- Much Flexible
- We will only get billed when we are actually utilizing the resources means only when our traffic goes up, then our infra gets scaled up and we will only get billed at that time only.
- This is something as a serverless architecture in which we have auto scaling or descaling.



# Cloud Infrastructure: Autoscaling vs. Serverless Architecture

## 1. What is Autoscaling?
Autoscaling is a cloud computing feature that automatically adjusts the number of active computing resources (like virtual servers, containers, or memory) in real time based on actual traffic demand.

*   **Horizontal Scaling (Scale-out / Scale-in):** Adds or removes entire instances or nodes (e.g., going from 2 servers to 5 servers).
*   **Vertical Scaling (Scale-up / Scale-down):** Adds or removes power—like CPU, RAM, or storage—to a single existing server.

### Who Provides It?
Major cloud service providers offer native autoscaling capabilities:
*   **Amazon Web Services (AWS):** AWS Auto Scaling / Amazon EC2 Auto Scaling.
*   **Microsoft Azure:** Azure AutoScale.
*   **Google Cloud Platform (GCP):** Managed Instance Groups (MIGs).
*   **Container Orchestrators:** Kubernetes (via the Cluster Autoscaler).

---

## 2. How Does Autoscaling Work? (Is It Automatic?)
**Yes, the execution is automatic**, but **you must configure the rules and boundaries** first. You tell the system how to behave by setting policies:

*   **Thresholds / Triggers:** e.g., *"If CPU utilization goes above 80% for 5 minutes, add 2 more servers."*
*   **Boundaries:** e.g., *"Never drop below 2 servers (Minimum), and never exceed 10 servers (Maximum)."*
*   **Automation:** Once these rules are set, the cloud provider monitors live metrics and spins servers up or down without human intervention.

### Do you pay for the maximum load all the time?
**No.** You only pay for the extra servers during the exact minutes or hours they are active and running. When traffic drops and the system "scales in" (removes the extra servers), billing for them stops immediately. You are never billed for your maximum limit unless your traffic actually forces the system to run at that limit.

---

## 3. How Serverless Architecture is Different
While both technologies scale based on traffic, **they are not the same**. The key differences lie in how they handle idling, billing granularity, and infrastructure management.

*   **True Zero Cost when Idle:** In standard autoscaling, you must keep at least one baseline server running 24/7 to listen for traffic, meaning you always pay a minimum monthly fee. In serverless, if no one visits your app, **you pay absolutely $0**.
*   **Billing Granularity:** Autoscaling charges you by the hour or minute for a whole server, regardless of how many requests it handles. Serverless charges you strictly by the **millisecond of execution time** per single request.
*   **No Pre-allocated Capacity:** With autoscaling, you still rent a "box" (e.g., a 4GB RAM server) that takes a few minutes to boot up. With serverless, the cloud provider instantly runs your code in a fraction of a second the moment a request hits, without you ever configuring virtual machines.

---

## 4. Head-to-Head Comparison

| Feature / Scenario | Traditional Autoscaling (IaaS / VMs) | Serverless Architecture (FaaS / e.g., AWS Lambda) |
| :--- | :--- | :--- |
| **What you manage** | Underlying virtual machines, operating systems, and runtime environments. | **Zero** servers; you only manage the application code. |
| **0 requests / Idle State** | You still pay for your base server(s) running 24/7. | **$0.00** (Completely free while idle). |
| **1 single request** | You pay for the full server for that minute/hour. | You pay only for that **one execution** (e.g., $0.000002). |
| **Sudden Spike (0 to 10k users)** | Takes **minutes** to boot new servers; early users face lag. | Scales **instantly** (milliseconds) to handle all requests. |
| **Granularity of scaling** | Scales at the **instance, node, or container** level. | Scales down to individual **function invocations**. |
| **Billing model** | Paid by provisioned capacity / time server is online. | Pure pay-per-use (millisecond execution time + request count). |


# Serverless Arch Use Cases:

## Cron Jobs: 
Because cron jobs runs at a particular scheduled time so having a sever running all the time is inefficient for this.

## Batch and Stream Processing:
We can use something like AWS SQS along with a lambda function (AWS Lambda) such that only when there is some data in SQS then that triggers and calls the API to process this data on serverless AWS Lambda.

## Chatbots
If we need something like if anyone stypes /command some command on slack then it should fetch something from DB and give response, this can be done on AWS Lambda where that command can call the serverless APIs.

## Scheduled DB backup:
Triggering a DB backup is just an API call if we are using a managed DB.

## Vending machine:
Suppose on every purchase we need to process something and that requires resources and this action is not frequent so we can use a serverless function here that when the actions gets performed (suppose something gets purchased from vending machine) then the serverless process should gets triggered.


# Advantages of Serverless Arch:

## No maintainance and Overhead of servers:
So we dont need to take care of the servers and they are fully managed by the vendors itself.

## Pay-as-you-go pricing:
So there is per request pricing and it will gives the flexibility that only the compute we use we only pay that much amount to the vendor.

## No Capacity planning:
If any server fails then what has to be done, like we need to shift to new server or all the capacity of the traffic is not maintained by us and it is maintained by the vendor and that is the abstraction.


# Disadvantages of Serverless Arch:

## Cold Start Problem:

First Request may take unnecessarily higher time.
Because in serverless the server is not always running so the first request because server is not running (is in sleep), it may take the time to boot the server and the first request may get served in a long amount of time and takes unnecessarily higher time.
After some time the machine again gets in sleep then if request comes in then again cold start problem may occur.

## Not build for long-running process:

Serverless has an execution time limit (amazon provides it as 15 minutes) 

## Testing and debugging is little tricky:
Also if we have the vendor so mostly the logs will go to vendor logs system if we need to debug and we deal with complex executions so debugging and testing that over serverless architecture may be challanging.

## Vendor lock-in:
It is very hard to move from one vendor to another. So it creates vendor lock in.
There is a lot to change in code, as we have implemented the functions and all according to vendor so needs to change those as well.

# When we should not choose serverless:

- Load is almost constant and predictable.

- Long Running process and execution.

- We need multi-tenancy (Means we need the isolation between the users/ requests and new infra for every user/request but serverless does not provide that because serverless is kind of common infra that is running for execution.)

# When we should use Serverless:

- Quick build, prototype, test and deploy.

- Use case is small and lightweight.

- traffic is burstly--- peak to 0 alternates.
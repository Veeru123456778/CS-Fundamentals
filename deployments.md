# Canary Deployment

We have pushed our new code changes to only 1 server and we are running it incrementally firstly for 1% traffic, then for 5% traffic, then for 10% traffic and eventually if everything works fine we roll out the changes to 100% of the traffic. This is what canary deployment is to understand that our new code changes working fine against previous stable version of the code.

So we also monitor the key metrices such as go routines, CPU utilization, RAM etc of new code and previous code if everything works fine then its correct if there is any behaviour change then needs to monitor that.

# Blue Green Deployment:

Blue Green Deployment is a deployment pattern that reduces the downtime by running two identical production environments called Blue and Green.

At one time only one environment is live.
Green is live while the blue env is idle. (So blue is the old version and green is the new version of the deployment)

This is beneficial for :

1. Minimal Downtime : During the time when server gets started and boots up we can minimize the downtime such that our requests will still come in old deployment and once the new deployment is up then our requests will come in new deployment that will ensure minimum downtime

2. Disaster Recovery or Rollback is easy/ super simple : We have our old deployment running if there is something wrong with the new deployment then we can roll back our and switch the traffic to old deployment again from new deployment in minimal time.

3. Deployments are quick : We just need to switch the traffic between new and old deployment and thats it.


## Potential Challanges of a blue green deployment:

1. During deployments the infra cost will be 2X

2. Stateful applications will loss the data if there is a data cached in local memory of the application server.

3. Database migrations : If we are using shared db for the blue green instances then we should need to make them (blue, green) in sync

4. Forward and Backward compatibility for APIs

5. It takes time and efforts to set up blue/green deployment

We should use blue/green deployment when:

1. We need 100% instant switch in deployment

2. We can bear the 2X cost of infra 

3. We need no downtime deployment


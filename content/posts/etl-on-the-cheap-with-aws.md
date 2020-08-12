---
title: "ETL on the Cheap with AWS"
date: 2019-07-15T00:49:17-04:00
author: Ze Xuan Ong
feature_image: /etl-on-the-cheap-with-aws/feature.png
draft: false
tags: ["Dev"]
summary: "Or how to be a an absolute scrooge using AWS as a student who ran out of GitHub Educate funds. Full of funny practices, such as putting long-lived functions in Lambdas, turning off the production database when not in use, and scheduling ETL on the clock without triggers."
---

To keep costs down, convoluted plans are probably sometimes necessary.

### Context

I built an ETL pipeline that takes Internet data from ScrapingHub, does cool stuff to the data and puts it in Metabase. After all that last month, I had a brief look at my AWS bill. I figured the bill couldn't be that bad, even though this last month I spun up my first t2.medium instance to keep Metabase running at a reasonable speed, which is in a funny way quite a financial milestone for me.

{{< figure src="/etl-on-the-cheap-with-aws/bill_chart.png" caption="The bill chart seems to correspond with the development efforts. You can also see the day where I switched from a t2.small to a t2.medium." >}}

Obviously I was wrong about the bill, which ended up being slightly over $100, which is not something I would like to have at this phase of my life (the phase where I scrounge around on LowEndBox because money). Looking at the cost, it seems that the EBS  cost quite a bit. Metabase runs on a t2.medium, backed by an t2.micro RDS for the infrastructure, and another t2.micro for holding the actual data.

Some obvious room for cost savings include, in increasing order of difficulty:

1. Reducing the scaling requirements for EBS which provisions Metabase
2. Turn off the EBS Metabase at scheduled times (non-working hours) since the number of users is spotty at the moment
3. Turn off the RDS instance (a t2.micro) holding the actual data when not using Metabase AND when the ETL is not occurring

Because I'd like to save maximize savings, I should attempt all three of them.

### 1. Reducing the scaling for Metabase on Elastic Beanstalk

I originally provisioned Metabase on EBS because it was a one-click setup option (ok quite a number of clicks but still). Given the budget and scale this project is currently run at, the scaling is unnecessary. Pretty much all the time we only require a single instance. In fact most of the time the single instance is not even required i.e. we can turn it off. In that case, we can make adjustments to the autoscaling of the EBS to shave off some cost.

#### Reducing auto scale

The autoscaling is handled by EBS, so most of the settings to change can be found on the EBS dashboard. This is a relative no-brainer - it is simply flipping a few switches and keying in some values to be as financially conservative as possible (small size and number of instances). The only major issue is that I am already at a maximum of 1 instance and I cannot seem to get it lower than that on the EBS settings i.e. shut it off completely.

This can be fixed by going to the EC2 dashboard, looking for the tab on 'Auto Scaling Groups' (last tab) and manually changing the settings there instead. Namely, we want to set the minimum to 0 instead of 1 to allow us to turn off the entire thing. Once that is done we can go ahead to turn off the EC2 instance that was spun up by EBS in the list of running instances.

#### Removing the load balancer

After doing the above, I noticed I was still being charged for the load balancer (coming to about 60 cents a day). It finally dawned upon me at this point that there is no point having a load balancer if there is nothing to balance it with - there is either one instance, or no instance at any time to route traffic to. Seems like I've failed to take away the basic lesson of distributed systems class.

With that, I deleted the load balancer from the EC2 dashboard. This stopped the EBS endpoint from working, since that points to the load balancer I think. In this case we can still access Metabase by using the endpoint of the instance itself. I have not yet verified whether the endpoint remains the same between turning on and off the instance, since unlike the usual EC2 instance, this particular instance is managed by the auto scaling group.

One minor issue here was that direct access to the instance was not allowed under the security group setting that the auto scaling group was part of. This was fixed by opening up an inbound port for my IP address. Something more general would be needed down the line.

### 2. Turning off/on Metabase at scheduled times

I confess that my use of Metabase is spotty at best, since most of the time I am working on the libraries supporting the ETL, so turning on/off Metabase on a schedule is unnecessary. I am happy to go turn on the server when I need it, and wait for it to boot up, and turn it off responsibly when I am done (I forgot to do so today). This is at some point undesirable, but not so much at this point in my life.

### 3. Turning off/on RDS at scheduled times

AWS relatively [recently allowed RDS instances to be turned off](https://aws.amazon.com/about-aws/whats-new/2017/06/amazon-rds-supports-stopping-and-starting-of-database-instances/) temporarily for up to seven days in a row. Practically for my use case, the RDS is only needed for two things:

1. To supply data to Metabase when Metabase is in use
2. To store transformed data from the ETL process

We can handle the first case as above. The second case requires us to sync the on/off times with the ETL process from ScrapingHub.

This then becomes annoying because similar penny-pinching micro-optimizations have also been applied on ScrapingHub previously. There are at least three different crawls that will occur daily with varying completion durations, run sequentially one after another. The RDS start/stop times are also somewhat variable. This means that hardcoding an RDS start/stop schedule in anticipation of ScrapingHub completion times would probably break more often than not, unless we provide enough buffer time e.g. 1 day before attempting to process and load the data.

We will need some mechanism to start the RDS when we need the transformed data to be loaded in. We would also prefer to only start the RDS once a day, since starting and stopping is slow, and one start-stop cycle could well be longer than a crawl cycle, resulting in more unnecessary error handling etc.

### Making it all Modular - the Plan

The plan is to orchestrate all these functions using AWS CloudWatch. This allows us to run the Lambdas on a schedule, or when certain triggers have been satisfied. There's a number of things we need to take care of before we can set up CloudWatch.

#### i) Decoupling the Transform and Load step

Previously there was a single Lambda function that took care of both transforming the data as it came into S3, as well as loading the transformed data into RDS. A single Lambda function made it rather simple for a project of this scale. With scheduling now coming into play, the convenience of the load step happening directly after the transform step is becomes somewhat of a hindrance. There is no real need for both steps to happen temporally directly after one another, so long as the order of operations is preserved.

With that, I decided to break down the original Lambda into two Lambdas. I've described it to a friend as attempting to organize a shipping container port, where the ship is the RDS that comes into port at some time. It's expensive to keep the ship in port, so we would ideally want to minimize the time spent in port. Hence, we should prepare all the containers for loading while the ship is out of port. In fact, we should do everything short of actually loading the containers themselves, so that we are all ready to go when the boat comes in.

#### ii) Micro-optimizing for the wallet

There is another financial incentive for separating the Lambdas into two. The original Lambda function had a really large amount of memory allocation (2GB) as all the data had to be loaded into memory before being transformed and reduced to something on the scale of 50MB. The 50MB staging data file is then loaded into the table in the database in RDS. That last bit, while not slow, isn't exactly blazing fast. While the Lambda is waiting for the I/O to complete, I'm being charged for the extra memory that is now unused (the bill is the product of the processing time and the amount of memory allocated).

So by decoupling the two transform and load step, we can allocate more RAM to the transform process which needs it, and allocate less RAM to the load process. If we have to do one transform and load step for every data source, then this could provide shave off some cost as well, accounting for the extra 100ms rounding cost incurred per Lambda invocation. Logically, the transform step can occur as soon as the data hits S3, while the load step can only occur after that but also only when the RDS is up. So we will generate the staging data in a .csv file ready for upload as is. The load step can take its time to actually put it in the database at the right time.

There's also a Free Tier for Lambdas, so trying to stay in that keeps costs down too.

#### iii) Accounting for Boot Time

It turns out that turning on and off the RDS is much slower than a regular EC2 instance. A few manual tries puts it in the ballpark of about 10 minutes in my region, though some ridiculous figures do appear online. 

The boot time matters because ideally I'd like to have the Lambda report an error if the RDS fails to start. For this to happen, the Lambda needs to stay alive long enough for the RDS to change state from stopped to available. If that takes 10 minutes on average however... then that's 10 minutes I'm getting charged for a Lambda, multiplied by the expected value of the number of starts required to turn on the instance.

Trading off between cost, simplicity of implementation and error handling, I felt that going with the first two was sufficient here (nothing mission critical), and I shall trust that the RDS will eventually start on time when we attempt to start it.

Turning on and off the RDS will also likely require an extra two Lambda functions, since the RDS console doesn't provide a means to schedule an on/off cycle. I'm of the opinion that AWS doesn't want to encourage people to turn off RDS instances (hence the auto-start after 7 days), so it has no reason to make this any easier than it minimally needs to be.

#### iv) Security

To turn on/off the RDS using Boto requires access to the AWS API. Lambdas in the VPC cannot access the AWS API on the public internet to turn on the RDS, despite being in the same VPC. This can be fixed by setting up a NAT gateway, which again costs money. I'm beginning to appreciate the money making potential of AWS.

In that case, we need to have the Lambdas that turn on and off the RDS sitting outside the VPC, while the Lambda that runs the data loading step can sit within the VPC. 

### Finally Setting up CloudWatch

After doing the above, we will have three triggers on CloudWatch, each of which activates a specific Lambda function to perform a certain task. Here are their descriptions, to be as specific as possible.

1. At scheduled time daily, call the Lambda that calls Boto to turn on the RDS instance
2. When said RDS instance changes its state from 'stopped' to 'available', call the Lambda in the VPC that loads the staging data from the staging S3 bucket into the database in the RDS instance
3. When loading for all staging data is complete, call the Lambda that called Boto to turn off the RDS instance

{{< figure src="/etl-on-the-cheap-with-aws/rds.png" caption="Three CloudWatch rules for chaining the things together. Also yes I cheated I'm just turning off the RDS at a scheduled time. BUT BUT I can justify this by claiming that I do need a cap on the uptime of the RDS, in the rare event that the loading step fails to take place i.e. I should not wait indefinitely for loading to take place, it could have failed silently :)" >}}

#### Daily Schedule

Scheduling a CloudWatch trigger to run daily can be done relatively easily following the syntax and examples provided [here](https://docs.aws.amazon.com/AmazonCloudWatch/latest/events/ScheduledEvents.html). For example to schedule the RDS to start at UTC time 10am daily (1), I have the following expression. A similar expression can be made to stop the RDS at scheduled times (3).

```
0 10 * * ? *
```

AWS seems to run only on UTC time. This is fine so long as it syncs with ScrapingHub, which also runs on UTC time. The date matters because it is used to distinguish new data from old data, so it would be extremely problematic if ScrapingHub provided data in the wrong date to our downstream processes.

You can check that everything is connected by going to the Lambda console. The relevant Lambdas should now have a CloudWatch trigger attached to them where there was previously none.

#### Triggers

Constructing the trigger when the RDS becomes available (2) is slightly more difficult. There is a built-in drop-down menu for EC2 services, where you can construct a trigger that activates when the state of the EC2 instance changes to a desired state. However, the menu for the Relational Database Service doesn't seem to provide a similar option. We can instead construct a custom event pattern as follows:

```
{
  "detail-type": [
    "RDS DB Instance Event"
  ],
  "source": [
    "aws.rds"
  ],
  "resources": [
    "<YOUR_RESOURCE_ARN>"
  ],
  "detail": {
    "EventCategories": [
      "notification"
    ],
    "SourceType": [
      "DB_INSTANCE"
    ],
    "Message": [
      "DB instance started"
    ]
  }
}
```

There isn't too much documentation on how to construct such an event pattern for RDS. I came to this by aggregating information from various sources, including manually turning on and off the RDS and checking the log to see the event notifications. You can do the same using the following command on the AWS CLI, which will fetch information about the last seven days of notifications.

```
$ aws rds describe-events --duration 10080
```

### Checking it all Works

The final flow of data is now as follows:

1. Data comes in from ScrapingHub at various times of day, is immediately transformed and stored in S3
2. At scheduled time daily, RDS is switched on
3. When RDS is switched on, transformed data is loaded into RDS
4. After scheduled amount of time, RDS is switched off

After testing it, I left it running over the weekend to see if it does as it should. Looking at the CloudWatch monitoring this morning, it seems to have turned out great:

1. The RDS is turned off at this time
3. CloudWatch monitoring shows a successful completion of each Lambda over the last three days
4. RDS log shows the RDS being turned on and off at specified times
5. Actually turning on the RDS showed fresh data in the database
6. Billing cost decreased

{{< figure src="/etl-on-the-cheap-with-aws/bill_chart_weekend.png" caption="Cost decreased! Look!" >}}

The labelling of the graphs doesn't give a very good picture of the actual usage. So here's another graph I took a day later labelled by database engines. The RDS instance runs a MySQL engine. We can visibly see the usage of the MySQL instance becoming insignificant over time, which is exactly what we want.

{{< figure src="/etl-on-the-cheap-with-aws/bill_chart_weekend.png" caption="Cost decreased! Look!" >}}



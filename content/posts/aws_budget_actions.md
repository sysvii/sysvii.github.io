+++
title = "Budgeting your way to Admin?"
description = "Exploration of AWS Budget actions as a baddie"
date = 2020-10-20
updated = 2020-10-20

[taxonomies]
categories = ["security"]
tags = ["aws", "budget", "security"]
+++

# How it started

Today I noticed a new AWS feature, you know one of the hundreds, that piqued my interest: [Budget Actions](https://aws.amazon.com/blogs/aws-cost-management/get-started-with-aws-budgets-actions/) and the most interesting part to me is this section:

>AWS Budgets now allows you to configure actions [..] that will be applied [..] once a budget target has been exceeded.
> There are three action types: **Identity and Access Management (IAM) policies, Service Control policies (SCPs)**, or target running instances (EC2 or RDS). 

Or in other words: if you are paying too much for AWS you can
react automagically by slapping an IAM policy or service control policy onto a thing.

# why tho?

On my first read of it I thought:

> This looks to be a new path to privilege escalation!

## When would you even use this?

This lends itself to some cool use cases 

 * stop exporting spendy logs by denying it access to the S3 bucket (That's what I'll be using it for)
 * big corporates might like to cut costs on training accounts with a
   `FinanceSayNo_DenyAll` SCP if you get too spendy
 * got a spendy vendor hosting infrastructure in your account? Just stop their
     instances once they are over budget.
 
# Trying it out

So as anyone would with a free service, you click about for a while.

The interesting parts I found about budgets (& thus budget actions) is that you have them activate
on not just cost but some service usage, like S3 or EC2 CPU credits, which is
a super neat TIL. 

At the end of my clicking I quickly got side tracked by the requirement of an
IAM role to be use.

## The humble default policy

I feel like AWS knows IAM is pretty annoying when it provides you some
some defaults to copy & paste into a new role.

And here they are:

### Default trust policy

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": {
        "Service": "budgets.amazonaws.com"
      },
      "Action": "sts:AssumeRole"
    }
  ]
}
```

### Default permissions policy

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "ec2:DescribeInstanceStatus",
        "ec2:StartInstances",
        "ec2:StopInstances",
        "iam:AttachGroupPolicy",
        "iam:AttachRolePolicy",
        "iam:AttachUserPolicy",
        "iam:DetachGroupPolicy",
        "iam:DetachRolePolicy",
        "iam:DetachUserPolicy",
        "organizations:AttachPolicy",
        "organizations:DetachPolicy",
        "rds:DescribeDBInstances",
        "rds:StartDBInstance",
        "rds:StopDBInstance",
        "ssm:StartAutomationExecution"
      ],
      "Resource": "*"
    }
  ]
}
```

### Thoughts on these policy

The permissions are pretty broad & could cause a lot of damage on their own.
Having the ability to attach policies to any of the three IAM principals is
screaming to be misused with some managed policies.

This policy is generated in the Console UI, so would be cool for them to tailor 
the permissions to resources selected in the UI. Be a nice secure by default
option & leave the full example in the doc pages.

The saving grace is the trust policy only allowing the `budgets` service to use
the role.

That's enough IAM nerd snipping, they have their reasons.

# Spy vs. Spy time

Like all good services in AWS that can mess with permissions, it might be good
to do some *threat modelling*.
Now, for todays session will be pitting against Red (Attackers) & Blue (Defenders) teams.
> *(this is not just me typing with different coloured LED lighting, I swear)*

## Setting the scene

To set the scene for this cyber battle:

* Someone read the aforementioned AWS blog post & decided to give it a go, using the provided default role policies it worked & then went along their day
* The Red team in their wizardy ways have managed to get the AWS credentials
of someone from Finance's Budgeting team that is reasonable scoped to the
[`AWSBudgetsActionsWithAWSResourceControlAccess`](https://github.com/glassechidna/trackiam/blob/master/policies/AWSBudgetsActionsWithAWSResourceControlAccess.json) policy.
* The Blue team are returning from their team lunch.

The Red Team has a few goals it is going for:
 * Elevating their own privilege
 * Don't let any one stop their rampage
 * Taking down *the site*

The Blue team wants none of that, but sometimes the best you can do is to know
something is about.

## Red team: are going in with the attack

With their newly acquired Budget Team permissions they find scope at what they
can even do in the budget service. To their surprise they find this blog post
telling them exactly how to abuse these permissions!

Fortunately the budget service has this great way to apply IAM policies to
roles at events & SCPs to organization units (OU).

### Elevating their own privilege
 
First things first, get more permissions. Red Team wants it **ALL**.

Someone want to try out this new priv-esc (what the kewl h@ck0rz call it) they
saw using `budgets`. So they:

 * Do the cloud `whoami` with the less succinct  `aws sys get-caller-identity`
 * Take stock, enumerate what permissions you got (there are some cool scripts
     for this like [andresriancho/enumerate-iam](https://github.com/andresriancho/enumerate-iam))
    * the team learns they can list few things like IAM roles & EC2
        * most importantly, listing IAM Roles includes trust policies
    * they also learn they got full access to `budgets`
    * they also discover the role setup for budget actions & left unused for years

Putting 1 and 1 together they came up with a plan of attack:
 1. Make a new budget that notifies at 1 S3 API call
 2. Add a budget action to add the `AdministratorAccess` managed policy to their
    IAM principal
 3. ???
 4. Profit

Not long after laying the trap it springs & they now got access to everything!

They are now free to do it all & order the [Snowmobile](https://aws.amazon.com/snowmobile/) they always wanted.

### Continuing the rampage

At this point it is pretty game over, but just to keep going until the Blue
Team notices & enjoying novelty of budget shinigans they keep going.

Next, on the list from the blog post: service control policies

So taking their previous knowledge, listing the SCPs in the account:

 1. Make a new budget that notifies at 1 S3 API call
 2. Add a budget action to add the `DenyAll` SCP that just happened to all the OUs
 3. ???
 4. No AWS services for anyone

> It is worth noting that this will only work in the Organization root account

### Take down the site

I think there is starting to be a pattern here... but this time targeting EC2 &
RDS instances.

### BONUS: Some things I didn't try cause it takes a billing cycle

The docs say it may remove the policy that is added by the alert.

So what happens if you add a policy that was already added?!

For example, targeting the default SCP policy to allow all services. Will that just get removed
at the start of the next billing cycle.
I know from experience that if you remove that one out you just turn AWS control
plane off killing any serverless apps.

## Blue team: you activated my trap card!

Since this service is behaving how it is suppose to & seems pretty legitimate it
is a bit hard to spot as it is happening. So I wouldn't blame them to be
surprised when the Snowmobile shows up. 

I doubt they'll get caught with the same trick twice though, looking back there
are some tells that tell a tall tale.

### Monitoring & Alerts

There are a few places the Blue team can put some monitors & alerts on:
 * events relating to a human permissions
 * events for attaching some juicy managed policies, like `AdministratorAccess` or
     `IAMFullAccess`
 * if budget events are infrequently changed, any event related to their
     change
 * or if you have a CI/CD deploying the actions, monitor for actions not by the
     CI/CD
 * events for old roles suddenly coming back to life

This is not an exhaustive list but some starting points.

Also worth pointing out these do not need to be paged on. No one likes
noisy alerts, they're often worse than nothing. So be careful tuning alerts.

### Cleaning up old IAM roles

The previous role left was the keystone of the Red team's whole operation and
it is the same story with like 90% of lateral movement maneuvers in AWS.

Cleaning up old unused IAM roles is really important, you never really know 

> The other 10% is over permissive IAM policies to make your own roles to
> maneuver through.

## Implicit nice parts

By having to bring your own role (BYOR) for the service to use it does help mitigate
most of the potential of a Red Team's fun with this new feature.

### Playing by your own rules

First of all, your BYOR will be subject to your own SCPs in
your account. So if you already got good guardrails in your SCPs that prevent
silly things like attaching small managed policies like `AdministratorAccess`
to a role then you'll be fine. If not, that might be a thing to look into.

### It doesn't have AWS Organization magic

Currently there is no direct support from AWS Organization.
Which limits the scope that these can be applied on but does mean repetition
across a fleet of AWS accounts.

Though, on the plus side it also means you can't pull an
Organization Cloudformation StackSet & make a service-linked role with
`AdministratorAccess` access all of the accounts with a click of a button
(*that is always a good time*).

If this ever does integrate with Organization, this feature will get added to the
"why billing account access is scary list".

### Blast radius

If the breach is not in the Organization root account, the damage is limited to
just that account. That's with the assumption that accounts in a Organization stay
completed separated, which is pretty generous.

# Conclusion

As with most things AWS, the devil is in the IAM policy and what IAM Roles that
get left around.

There is nothing inherently wrong with this service or how it does about its
merry day, it is just in conjunction with people using AWS it can lead to a bad
time:tm:.

## My Takeaways

Some of my takeaways from writing this:

 * **Unused IAM Roles can be dangerous: so clean up after yourself**
 * **Monitoring helps**
 * **Have guardrails around *juicy* managed IAM policies, it may save you
     later on**
 * **New features in older services can lead to interesting paths to
     `AdministratorAccess`, I would say read their announcement feeds but there
     is too much so see if anyone makes a blog about it**

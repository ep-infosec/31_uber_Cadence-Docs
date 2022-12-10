---
layout: default
title: Workflows
permalink: /docs/concepts/workflows
---

# Fault-oblivious stateful workflow code

## Overview

Cadence core abstraction is a **fault-oblivious stateful :workflow:**. The state of the :workflow: code, including local variables and threads it creates, is immune to process and Cadence service failures.
This is a very powerful concept as it encapsulates state, processing threads, durable timers and :event: handlers.

## Example

Let's look at a use case. A customer signs up for an application with a trial period. After the period, if the customer has not cancelled, he should be charged once a month for the renewal. The customer has to be notified by email about the charges and should be able to cancel the subscription at any time.

The business logic of this use case is not very complicated and can be expressed in a few dozen lines of code. But any practical implementation has to ensure that the business process is fault tolerant and scalable. There are various ways to approach the design of such a system.

One approach is to center it around a database. An application process would periodically scan database tables for customers in specific states, execute necessary actions, and update the state to reflect that. While feasible, this approach has various drawbacks. The most obvious is that the state machine of the customer state quickly becomes extremely complicated. For example, charging a credit card or sending emails can fail due to a downstream system unavailability. The failed calls might need to be retried for a long time, ideally using an exponential retry policy. These calls should be throttled to not overload external systems. There should be support for poison pills to avoid blocking the whole process if a single customer record cannot be processed for whatever reason. The database-based approach also usually has performance problems. Databases are not efficient for scenarios that require constant polling for records in a specific state.

Another commonly employed approach is to use a timer service and queues. Any update is pushed to a queue and then a :worker: that consumes from it updates a database and possibly pushes more messages in downstream queues. For operations that require scheduling, an external timer service can be used. This approach usually scales much better because a database is not constantly polled for changes. But it makes the programming model more complex and error prone as usually there is no transactional update between a queuing system and a database.

With Cadence, the entire logic can be encapsulated in a simple durable function that directly implements the business logic. Because the function is stateful, the implementer doesn't need to employ any additional systems to ensure durability and fault tolerance.

Here is an example :workflow: that implements the subscription management use case. It is in Java, but Go is also supported. The Python and .NET libraries are under active development.

```java
// This SubscriptionWorkflow interface is an example of defining a workflow in Cadence
public interface SubscriptionWorkflow {
    @WorkflowMethod
    void manageSubscription(String customerId);
    @SignalMethod
    void cancelSubscription();
    @SignalMethod    
    void updateBillingPeriodChargeAmount(int billingPeriodChargeAmount);
    @QueryMethod    
    String queryCustomerId();
    @QueryMethod        
    int queryBillingPeriodNumber();
    @QueryMethod        
    int queryBillingPeriodChargeAmount();
}

// Workflow implementation is independent from interface. That way, application that start/signal/query workflows only need to know the interface
public class SubscriptionWorkflowImpl implements SubscriptionWorkflow {

    private int billingPeriodNum;
    private boolean subscriptionCancelled;
    private Customer customer;
    
    private final SubscriptionActivities activities =
            Workflow.newActivityStub(SubscriptionActivities.class);

    // This manageSubscription function is an example of a workflow using Cadence
    @Override
    public void manageSubscription(Customer customer) {
        // Set the Workflow customer to class properties so that it can be used by other methods like Query/Signal
        this.customer = customer;

        // sendWelcomeEmail is an activity in Cadence. It is implemented in user code and Cadence executes this activity on a worker node when needed.
        activities.sendWelcomeEmail(customer);

        // for this example, there are a fixed number of periods in the subscription
        // Cadence supports indefinitely running workflow but some advanced techniques are needed
        while (billingPeriodNum < customer.getSubscription().getPeriodsInSubcription()) {

            // Workflow.await tells Cadence to pause the workflow at this stage (saving it's state to the database)
            // Execution restarts when the billing period time has passed or the subscriptionCancelled event is received , whichever comes first
            Workflow.await(customer.getSubscription().getBillingPeriod(), () -> subscriptionCancelled);

            if (subscriptionCancelled) {
                activities.sendCancellationEmailDuringActiveSubscription(customer);
                break;
            }
            
            // chargeCustomerForBillingPeriod is another activity
            // Cadence will automatically handle issues such as your billing service being unavailable at the time
            // this activity is invoked
            activities.chargeCustomerForBillingPeriod(customer, billingPeriodNum);

            billingPeriodNum++;
        }

        if (!subscriptionCancelled) {
            activities.sendSubscriptionOverEmail(customer);
        }
        
        // the workflow is finished once this function returns
    }

    @Override
    public void cancelSubscription() {
        subscriptionCancelled = true;
    }

    @Override
    public void updateBillingPeriodChargeAmount(int billingPeriodChargeAmount) {
        customer.getSubscription().setBillingPeriodCharge(billingPeriodChargeAmount);
    }

    @Override
    public String queryCustomerId() {
        return customer.getId();
    }

    @Override
    public int queryBillingPeriodNumber() {
        return billingPeriodNum;
    }

    @Override
    public int queryBillingPeriodChargeAmount() {
        return customer.getSubscription().getBillingPeriodCharge();
    }
}

```
Again, note that this code directly implements the business logic. If any of the invoked operations (aka :activity:activities:) takes a long time, the code is not going to change. It is okay to block on `chargeCustomerForBillingPeriod` for a day if the downstream processing service is down that long. The same way that blocking sleep for a billing period like 30 days is a normal operation inside the :workflow: code.

Cadence has practically no scalability limits on the number of open :workflow: instances. So even if your site has hundreds of millions of consumers, the above code is not going to change.

The commonly asked question by developers that learn Cadence is "How do I handle :workflow_worker: process failure/restart in my :workflow:"? The answer is that you do not. **The :workflow: code is completely oblivious to any failures and downtime of :worker:workers: or even the Cadence service itself**. As soon as they are recovered and the :workflow: needs to handle some :event:, like timer or an :activity: completion, the current state of the :workflow: is fully restored and the execution is continued. The only reason for a :workflow: failure is the :workflow: business code throwing an exception, not underlying infrastructure outages.

Another commonly asked question is whether a :worker: can handle more :workflow: instances than its cache size or number of threads it can support. The answer is that a :workflow:, when in a blocked state, can be safely removed from a :worker:.
Later it can be resurrected on a different or the same :worker: when the need (in the form of an external :event:) arises. So a single :worker: can handle millions of open :workflow_execution:workflow_executions:, assuming it can handle the update rate.

## State Recovery and Determinism

The :workflow: state recovery utilizes :event: sourcing which puts a few restrictions on how the code is written. The main restriction is that the :workflow: code must be deterministic which means that it must produce exactly the same result if executed multiple times. This rules out any external API calls from the :workflow: code as external calls can fail intermittently or change its output any time. That is why all communication with the external world should happen through :activity:activities:. For the same reason, :workflow: code must use Cadence APIs to get current time, sleep, and create new threads.

To understand the Cadence execution model as well as the recovery mechanism, watch the following webcast. The animation covering recovery starts at 15:50.

<figure class="video-container">
  <iframe
    src="https://www.youtube.com/embed/qce_AqCkFys?start=960"
    frameborder="0"
    height="315"
    allowfullscreen
    width="560"></iframe>
</figure>

## ID Uniqueness

:workflow_ID:Workflow_ID: is assigned by a client when starting a :workflow:. It is usually a business level ID like customer ID or order ID.

Cadence guarantees that there could be only one :workflow: (across all :workflow: types) with a given ID open per :domain: at any time. An attempt to start a :workflow: with the same ID is going to fail with `WorkflowExecutionAlreadyStarted` error.

An attempt to start a :workflow: if there is a completed :workflow: with the same ID depends on a `WorkflowIdReusePolicy` option:

- `AllowDuplicateFailedOnly` means that it is allowed to start a :workflow: only if a previously executed :workflow: with the same ID failed.
- `AllowDuplicate` means that it is allowed to start independently of the previous :workflow: completion status.
- `RejectDuplicate` means that it is not allowed to start a :workflow_execution: using the same :workflow_ID: at all.
- `TerminateIfRunning` means terminating the current running workflow if one exists, and start a new one.

The default is `AllowDuplicateFailedOnly`.

To distinguish multiple runs of a :workflow: with the same :workflow_ID:, Cadence identifies a :workflow: with two IDs: `Workflow ID` and `Run ID`. `Run ID` is a service-assigned UUID. To be precise, any :workflow: is uniquely identified by a triple: `Domain Name`, `Workflow ID` and `Run ID`.

## Child Workflow

A :workflow: can execute other :workflow:workflows: as `child :workflow:workflows:`. A child :workflow: completion or failure is reported to its parent.

Some reasons to use child :workflow:workflows: are:

- A child :workflow: can be hosted by a separate set of :worker:workers: which don't contain the parent :workflow: code. So it would act as a separate service that can be invoked from multiple other :workflow:workflows:.
- A single :workflow: has a limited size. For example, it cannot execute 100k :activity:activities:. Child :workflow:workflows: can be used to partition the problem into smaller chunks. One parent with 1000 children each executing 1000 :activity:activities: is 1 million executed :activity:activities:.
- A child :workflow: can be used to manage some resource using its ID to guarantee uniqueness. For example, a :workflow: that manages host upgrades can have a child :workflow: per host (host name being a :workflow_ID:) and use them to ensure that all operations on the host are serialized.
- A child :workflow: can be used to execute some periodic logic without blowing up the parent history size. When a parent starts a child, it executes periodic logic calling that continues as many times as needed, then completes. From the parent point if view, it is just a single child :workflow: invocation.

The main limitation of a child :workflow: versus collocating all the application logic in a single :workflow: is lack of the shared state. Parent and child can communicate only through asynchronous :signal:signals:. But if there is a tight coupling between them, it might be simpler to use a single :workflow: and just rely on a shared object state.

We recommended starting from a single :workflow: implementation if your problem has bounded size in terms of number of executed :activity:activities: and processed :signal:signals:. It is more straightforward than multiple asynchronously communicating :workflow:workflows:.

## Workflow Retries

:workflow:Workflow: code is unaffected by infrastructure level downtime and failures. But it still can fail due to business logic level failures. For example, an :activity: can fail due to exceeding the retry interval and the error is not handled by application code, or the :workflow: code having a bug.

Some :workflow:workflows: require a guarantee that they keep running even in presence of such failures. To support such use cases, an optional exponential _retry policy_ can be specified when starting a :workflow:. When it is specified, a :workflow: failure restarts a :workflow: from the beginning after the calculated retry interval. Following are the retry policy parameters:

- `InitialInterval` is a delay before the first retry.
- `BackoffCoefficient`. Retry policies are exponential. The coefficient specifies how fast the retry interval is growing. The coefficient of 1 means that the retry interval is always equal to the `InitialInterval`.
- `MaximumInterval` specifies the maximum interval between retries. Useful for coefficients of more than 1.
- `MaximumAttempts` specifies how many times to attempt to execute a :workflow: in the presence of failures. If this limit is exceeded, the :workflow: fails without retry. Not required if `ExpirationInterval` is specified.
- `ExpirationInterval` specifies for how long to attempt executing a :workflow: in the presence of failures. If this interval is exceeded, the :workflow: fails without retry. Not required if `MaximumAttempts` is specified.
- `NonRetryableErrorReasons` allows to specify errors that shouldn't be retried. For example, retrying invalid arguments error doesn't make sense in some scenarios.

## How does workflow run 
You may wonder how it works. Behind the scenes, workflow decision is driving the whole workflow running. It's the internal entities for client and server to run your workflows. If this is interesting to you, read this [stack Overflow QA](https://stackoverflow.com/questions/62904129/what-exactly-is-a-cadence-decision-task/63964726#63964726).

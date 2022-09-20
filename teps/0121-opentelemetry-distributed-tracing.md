---
status: proposed
title: Instrument Tekton Pipelines code to allow for distributed tracing of Tekton Tasks and Pipelines
creation-date: '2022-09-20'
last-updated: '2022-09-20'
authors:
- '@kmjayadeep'
---

# TEP-0121: Instrument Tekton Pipelines code to allow for distributed tracing of Tekton Tasks and Pipelines

<!-- toc -->
- [Summary](#summary)
  - [Use Cases](#use-cases)
- [Proposal](#proposal)
- [Design Details](#design-details)
  - [taskRunTemplate](#implement)
  - [Implementation Plan](#implementation-plan)
  - [Test Plan](#test-plan)
  

<!-- /toc -->

## Summary
With distributed tracing, we can track the time taken by each action in
the pipeline like reconciling logic, fetching resources, pulling images
etc. This allows the developers to improve the reconciliation logic and
also allow end users to monitor and optimize the pipelines.


### Use Cases
Pipeline and Task User:
* I would like to understand then duration of each step in my pipeline so that I can optimize the slow steps to improve the pipeline execution speed

Developer:
* I would like to understand the duration of each reconciliation step, so that I can optimize the code to improve reconciliation performance
* When the pipelines are failing due to a bug, I would like to understand which reconciliation logic caused the issue so that I can easily fix the problem

## Proposal
Initialize a tracer provider with jaeger as a backend. The jaeger
collector URL can be passed as an argument to the controller.

### PipelineRun controller
A new tracer span will be initialized in the pipelineRun controller when a new PipelineRun CR is created. The span context will be propogated through the reconciliation methods
to instrument the actions and steps included in the reconciliation logic. The span context will be saved back to the PipelineRun CR as an annotation `tekton/span-context`. This span context can
be retrieved during the next reconciliation loop for the same CR. This way, we will have a single parent span for the entire reconciliation logic for a single PipelineRun CR. This makes it easy to visualize
the multiple reconciliation steps involved for each PipelineRun.

When a TaskRun is created by the PipelineRun reconciler, the parent span context is passed as an annotation `tekton/span-context` so that TaskRun reconciler use the same span as its parent.

## TaskRun controller
TaskRun reconciler retrieves the parent span context propogated by PipelineRun controller. If it is not present (TaskRun is created by user in this case), a new span will be created. It will be used to 
instrument the logic similar to PipelineRun controller

## Goals
* Implementation of opentelemetry tracing with Jaeger
* Instrumentation of pipelinerun and taskrun reconciliation logic

## Non-Goals
* Instrumentation of sidecars and initcontainers
* Support for more tracing backends

### Test Plan
There must be unit tests for recording of spans and e2e tests for context propogation through custom resources. 

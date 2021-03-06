---
title: Azure Functions Consumption and App Service Plans | Microsoft Docs
description: Understand how Azure Functions scale to meet the needs of your event-driven workloads.
services: functions
documentationcenter: na
author: lindydonna
manager: erikre
editor: ''
tags: ''
keywords: azure functions, functions, event processing, webhooks, dynamic compute, serverless architecture

ms.assetid: 5b63649c-ec7f-4564-b168-e0a74cb7e0f3
ms.service: functions
ms.devlang: multiple
ms.topic: reference
ms.tgt_pltfrm: multiple
ms.workload: na
ms.date: 05/15/2017
ms.author: donnam, glenga

ms.custom: H1Hack27Feb2017

---
# Azure Functions Consumption and App Service Plans 

## Introduction

Azure Functions can be run in two different modes: Consumption plan and App Service plan. The Consumption plan automatically allocates compute power when your code is running, scales-out as necessary to handle load, and then scales down when code is not running. So, you don't have to pay for idle VMs and don't have to reserve capacity in advance. This article focuses on the Consumption plan. For details about how the App Service plan works, see the [Azure App Service plans in-depth overview](../app-service/azure-web-sites-web-hosting-plans-in-depth-overview.md). 

If you are not yet familiar with Azure Functions, see the [Azure Functions overview](functions-overview.md) article.

When you create a function app, you must configure a hosting plan for functions contained in the app. The available hosting plans are the **Consumption Plan** and the [**App Service plan**](../app-service/azure-web-sites-web-hosting-plans-in-depth-overview.md). In either mode, functions are executed by an instance of the *Azure Functions host*. The type of plan controls: 1) how host instances are scaled out and 2) the resources that are available to each host.

Currently, the choice of plan type must be made during the creation of the function app, and cannot be changed afterwards. You can scale between tiers on the [App Service plan](../app-service/azure-web-sites-web-hosting-plans-in-depth-overview.md). On the Consumption plan, Azure Functions automatically handles all resource allocation.

## Consumption plan

When using a **Consumption plan**, instances of the Azure Functions host are dynamically added and removed based on the number of incoming events. This plan scales automatically and you are charged for compute resources only when your functions are running. On a Consumption plan, a function can run for a maximum of five minutes. 

Billing is based on execution time and memory used and is aggregated across all functions within a function app. For more information, see the [Azure Functions pricing page].

The Consumption plan is the default and offers the following benefits:
- Pay only when your functions are running
- Scale out automatically, even during periods of high load

## App Service plan

In the **App Service plan**, your function apps run on dedicated VMs on Basic, Standard, Premium SKUs, similar to Web Apps. Dedicated VMs are allocated to your App Service apps, which means the functions host is always running.

Consider an App Service plan in the following cases:
- You have existing, under-utilized VMs that are already running other App Service instances.
- You expect your function apps to run continuously, or nearly continuously.
- You need more CPU or memory options than what is provided in the Consumption plan.
- You need to run longer than the maximum execution time allowed on the Consumption plan

A VM decouples cost from both runtime and memory size. As a result, you won't pay more than the cost of the VM instance you allocate. For details about how the App Service plan works, see the [Azure App Service plans in-depth overview](../app-service/azure-web-sites-web-hosting-plans-in-depth-overview.md). 

With an App Service plan, you can manually scale out by adding more VM instances, or you can enable auto-scale. For more information, see [Scale instance count manually or automatically](../monitoring-and-diagnostics/insights-how-to-scale.md?toc=%2fazure%2fapp-service-web%2ftoc.json). You can also scale up by choosing a different App Service plan. For more information, see [Scale up an app in Azure](../app-service-web/web-sites-scale.md). If you are planning to run JavaScript functions on an App Service plan, you should choose a plan with fewer cores. For more information, see the [JavaScript reference for Functions](functions-reference-node.md#choose-single-core-app-service-plans).  

<!-- Note: the portal links to this section via fwlink https://go.microsoft.com/fwlink/?linkid=830855 --> 
<a name="always-on"></a>
### Always On

If you run on an App Service Plan, you should enable the **Always On** setting so that your Function App  runs correctly. On an App Service Plan, the functions runtime will go idle after a few minutes of inactivity, so only HTTP triggers will actually "wake up" your functions. This is similar to how WebJobs must have Always On enabled. 

Always On is only available on an App Service plan. On a Consumption plan, the platform activates Function Apps automatically.

## Storage account requirements

On either a Consumption or App Service plan, a Function App requires an Azure Storage account that supports Blob, Queue, and Table storage. Internally Azure Functions uses Azure Storage for operations such as managing triggers and logging function executions. Some storage accounts do not support queues and tables, such as blob-only storage accounts (including premium storage) and general-purpose storage accounts with ZRS replication. These accounts are filtered from the Storage Account blade when creating a Function App.

To learn more about storage account types, see [Introducing the Azure Storage Services] (../storage/storage-introduction.md#introducing-the-azure-storage-services).

## How the Consumption plan works

The Consumption plan automatically scales CPU and memory resources by adding additional instances of the Functions host, based on the number of events that its functions are triggered on. Each instance of the Functions host is limited to at most 1.5 GB of memory.

When using the Consumption hosting plan, function code files are stored on Azure Files shares on the main storage account. When you delete the main storage account, this content is deleted and cannot be recovered.

> [!NOTE]
> When using a blob trigger on a Consumption plan, there can be up to a 10-minute delay in processing new blobs if a Function App has gone idle. Once the Function App is running, blobs are processed immediately. To avoid this initial delay, consider one of the following options:
> - Use an App Service Plan with Always On enabled.
> - Use another mechanism to trigger the blob processing, such as a queue message that contains the blob  name. For an example, see [queue trigger with blob input binding](functions-bindings-storage-blob.md#input-sample).

### Runtime scaling

Azure Functions uses a component called the *scale controller* to monitor the rate of events and determine whether to scale out or scale down. The scale controller uses heuristics for each trigger type. For example, when using an Azure Queue Storage trigger, it scales based on the queue length and the age of the oldest queue message.

The unit of scale is the function app. When scaled out, more resources are allocated to run multiple instances of the Azure Functions host. Conversely, as compute demand is reduced, function host instances are removed. The number of instances is eventually scaled down to zero when no functions are running within a function app.

![](./media/functions-scale/central-listener.png)

### Billing model

Billing for the Consumption plan is described in detail on the [Azure Functions pricing page]. Usage is aggregated at the function app level and counts only the time that function code is executing. The following are units for billing: 
* **Resource consumption in GB-s (gigabyte-seconds)**. Computed as a combination of memory size and execution time for all functions within a Function App. 
* **Executions**. Counted each time a function is executed in response to an event, triggered by a binding.

[Azure Functions pricing page]: https://azure.microsoft.com/pricing/details/functions
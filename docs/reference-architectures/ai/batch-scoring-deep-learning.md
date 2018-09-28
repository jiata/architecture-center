---
title: Batch scoring on Azure for deep learning models
description: This reference architecture shows how to apply style transfer to a video, using Azure Batch AI
author: jiata
ms.date: 09/28/2018
ms.author: jiata
---

# Batch scoring on Azure for deep learning models

This reference architecture shows how to apply style transfer to a video, using Azure Batch AI. *Style transfer* is a deep learning technique that composes an existing image in the style of another image. This architecture can be generalized for any scenario that uses batch scoring with deep learning. [**Deploy this solution**](#deploy-the-solution).
 
![](./_images/batch-ai-deep-learning.png)

**Scenario**: A media organization has a video whose style they want to change to look like a specific painting. The organization wants to be able to apply this style to all frames of the video in a timely manner and in an automated fashion. For more background about style transfer algoroithms, see [Image Style Transfer Using Convolutional Neural Networks](https://www.cv-foundation.org/openaccess/content_cvpr_2016/papers/Gatys_Image_Style_Transfer_CVPR_2016_paper.pdf) (PDF).

![](./_images/batch-ai-style-transfer.png)

This reference architecture is designed for work that is triggered by the presence of new media to process which can be done manually. 

The end-to-end steps that this reference architecture covers are as follows:

1. Upload a selected style image (like a Van Gogh painting) and a style transfer script to Blob Storage.
2. Create an autoscaling Batch AI cluster that is ready to start taking work.
3. Split the video file into individual frames and upload those frames into Blob Storage.
4. Uploading the frames triggers a Logic App that creates a container running in Azure Container Instance.
5. The container runs a script that creates the Batch AI jobs. Each job applies the style transfer in parallel across the nodes of the Batch AI cluster.
6. Once the images are generated, they are saved back to Blob Storage.
7. Download the generated frames, and stitch back the images into a video.

## Architecture
This architecture consists of the following components.

### Compute

**[Azure Batch AI](/azure/batch-ai/)** is used to run the style transfer algorithm. Batch AI supports deep learning workloads by providing containerized environments that are pre-configured for deep learning frameworks, on GPU-enabled VMs. 

### Storage

**Blob Storage**. [Blob storage](/azure/storage/blobs/storage-blobs-introduction) is used to store all images (input images, style images, and output images) as well as all logs produced from Batch AI. Blob storage integrates with Batch AI via [blobfuse](https://github.com/Azure/azure-storage-fuse), an open-source virtual filesystem that is backed by Blob storage. Blob storage is also very cost-effective for the performance that this workload requires.

### Trigger / Scheduling

**Azure Logic Apps**. [Logic Apps](/azure/logic-apps/) is used to trigger the workflow. When the logic app detects that a blob has been added to the container, it triggers the Batch AI process. Using logic app is a great fit for this reference architecture because it is an easy way to detect change to blob storage and provides an easy process for changing the trigger.

**Azure Container Instances**. [Container Instances](/azure/container-instances/) are used to run the Python scripts that create the AI Batch jobs. Running these scripts inside a Docker container is a convenient way to run them on demand. For this architecture, we use Container Instances because there it has pre-built Logic Apps connector for it, which allows the logic app to trigger the AI Batch job. Container Instances is a convenient way to spin up stateless processes quickly.

**DockerHub**. DockerHub is used to store the Docker image that Container Instances uses to execute the job creation process. DockerHub was chosen for this architecture because it is easy to use and is the default image repository for Docker users. Azure Container Registry can also be used for this architecture.

### Data Preparation

This reference architecture uses video footage of an orangutan in a tree. You can download the footage from [here](https://happypathspublic.blob.core.windows.net/videos/orangutan.mp4) and process it for the workflow by following these steps:

1. Use [AzCopy](/azure/storage/common/storage-use-azcopy-linux) to download the video from the public blob.
2. Use [FFmpeg](https://www.ffmpeg.org/) to extract the audio file, so that the audio file can be stitched back into the output video later.
3. Use FFmpeg to break the video into individual frames. The frames will be processed independently, in parallel.
4. Use AzCopy to copy the individual frames into your blob container.
At this stage, the video footage is in a form that can be used for style transfer.

## Performance considerations

### GPU vs CPU

For deep learning workloads, GPUs will generally out-perform CPUs by a considerable amount, to the extent that a sizeable cluster of CPUs is usually needed to get comparable performance. While it is an option to use CPUs only for this reference architecture, using GPUs will provide a much better cost/performance profile. We recommend using the latest NCv3 series of GPU optimized VMs.

GPUs are not enabled by default in all regions. Make sure to select a region with GPUs enabled. In addition, subscriptions have a default quota of zero cores for GPU-optimized VMs. You can raise this quota by opening a support request. Make sure that your subscription has enough quota to run your workload.

### Parallelizing across VMs vs Cores

When running a style transfer process as a batch job, the jobs that run primarily on GPUs will have to be parallelized across VMs. Two approaches are possible: You can create a larger cluster using VMs that have a single GPU, or create a smaller cluster using VMs with many GPUs. 

For this workload, these two options will be comparable in terms of performance. Using fewer VMs with more GPUs per VM can help to reduce data movement. However, the data volume per job isn't very large for this workload, so you won't observe much throttling in Azure blob.

### Configuring the number of images to process per Batch AI job

Another parameter that must be configured is the number of images to process per Batch AI job. On the one hand, you want to ensure that work is spread broadly across the nodes. That points to having many Batch AI jobs and thus a low number of images to process per job. On the other hand, if too few images are processed per job, the setup/startup time becomes disproportionately large. The sweet spot is to have enough jobs to run on all nodes of the cluster, while also minimizing the total setup time across all jobs. Processing between 50 to 100 images per job tends to work well for shorter videos. 

### File Servers

When using Batch AI, you can choose multiple storage options depending on the throughput needed for your scenario. For workloads with low throughput requirements, using blobfuse should be enough. Alternatively, Batch AI also supports a Batch AI File Server, a managed single-node NFS, which can be automatically mounted on cluster nodes to provide a centrally accessible storage location for jobs. For most cases, only one file server is needed in a workspace, and you can separate data for your training jobs into different directories. If NFS isn't appropriate for your workloads, Batch AI supports other storage options, including Azure Storage or custom solutions such as a Gluster or Lustre file system.

## Security considerations

### Restricting access to Azure blob storage

In this reference architecture, Azure blob storage is the main storage component that needs to be protected. The baseline deployment shown in the GitHub repo uses storage account keys to access the blob storage. For further control and protection, consider using a shared access signature (SAS) instead. This grants limited access to objects in storage, without needing to hard code the account keys or save them in plaintext. This approach is especially useful because account keys are visible in plaintext inside of Logic App's designer interface. Using an SAS also helps to ensure that the storage account has proper governance, and that access is granted only to the people intended to have it.

For scenarios with more sensitive data, make sure that all your storage keys are protected, because these keys grant full access to all input and output data from the workload.

### Data encryption and data movement

This reference architecture uses style transfer as an example of a batch scoring process. For more data-sensitive scenarios, the data in storage should be encrypted at rest. Each time data is moved from one location to the next, use SSL to secure the data transfer. For more information, see [Azure Storage security guide](/azure/storage/common/storage-security-guide). 

### Securing data in a virtual network

When deploying your Batch AI cluster, you can configure your cluster to be provisioned inside a subnet of a Virtual Network (VNET). This allows the compute nodes in the cluster to communicate securely with other virtual machines, or even with an on-premises network. You can also use [service endpoints](/azure/storage/common/storage-network-security?toc=%2fazure%2fvirtual-network%2ftoc.json#grant-access-from-a-virtual-network) with blob storage to grant access from a virtual network or use a single-node NFS inside the VNET with Batch AI to ensure that the data is always protected.

### Protecting against malicious activity

In scenarios where there are multiple users, make sure that sensitive data is protected against malicious activity. If other users are given access to this deployment to customize the input data, the following precautions should be taken:

- Use RBAC to limit users' access to only the resources they need.
- Provision two separate storage accounts. Store input and output data in the first account. External users can be given access to this account. Store executable scripts and output log files in the other account. External users should not have access to this account. This will ensure that external users cannot modify any executable files (to inject malicious code), and don't have access to logfiles, which could hold sensitive content.
- Make sure that appropriate retry policies are set in Batch AI. Otherwise, malicious users can DDOS the job queue or inject malformed poison messages in the job queue, causing the system to lock up or causing dequeuing errors. 

## Monitoring and logging

### Monitoring Batch AI jobs

While running your job, it's important to monitor the progress and make sure that things are working as expected. However, it can be a challenge to monitor across a cluster of active nodes. 

To get a sense of the overall state of the cluster, go to the Batch AI blade of the Azure Portal to inspect the state of the nodes in the cluster. If a node is inactive or a job has failed, the error logs are saved to blob storage, and are also be accessible in the Jobs blade in the Azure Portal. 

Monitoring can be further enriched by connecting logs to Application Insights or by running separate processes to poll for the state of the Batch AI cluster and its jobs.

### Logging in Batch AI

Batch AI will automatically log all stdout/stderr to the associate blob storage account. Using a storage navigation tool such as Storage Explorer will provide a much easier experience for navigating log files. 

![](./_images/batch-ai-logging.png)

The deployment steps for this reference architecture show how to set up a more simple logging system, such that all the logs across the different jobs are saved to the same directory in your blob container, as illustrated above.
Use logs to monitor how long it takes for each job and each image to process. This will give you a better sense of how to optimize the process even further.

## Cost considerations

Compared to the storage and scheduling components, the compute resources used in this reference architecture by far dominate in terms of costs. One of the main challenges is effectively parallelizing the work across a cluster of GPU-enabled machines. Thus, we must take precautions to minimize incurred costs.

The Batch AI cluster size can automatically scale up and down depending on the jobs in the queue. You can enable auto-scale with Batch AI in one of two ways. You can do so programmatically, which can be configured in the `.env` file that is part of the [deployment steps][deployment], or you can change the scale formula directly in the portal after the cluster is created.

For work that doesn't require immediate processing, configure the auto-scale formula so the default state (minimum) is a cluster of zero nodes, and set the maximum cluster size to the maximum number of Batch AI jobs that could occur at any given time. With this configuration, the cluster's default state is zero nodes, and the cluster only scales up when it detects jobs in the queue. This setting enables significant cost savings for scenarios where the batch scoring process only happens a few times a day or less.

Auto-scaling may not be appropriate for batch jobs that happen too close to each other in time. The time that it takes for a cluster to spin up and spin down also incur a cost, so if a batch workload begins only a few minutes after the previous job ends, it might be more cost effective to keep the cluster running between jobs.

## Deploy the solution

To deploy this reference architecture, follow the steps described in the [GitHub repo][deployment].

[deployment]: https://github.com/Azure/batch-scoring-for-dl-models
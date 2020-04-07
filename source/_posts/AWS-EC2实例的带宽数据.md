---
title: AWS EC2实例的带宽数据
date: 2020-04-07 14:35:07
tags: [AWS]
categories: Cloud
---
[原文出处](https://cloudonaut.io/ec2-network-performance-cheat-sheet/)

What is the maximum network throughput of your EC2 instance? The answer to this question is key to choosing the type of an instance or defining monitoring alerts on network throughput.

EC2 Network Performance Cheat Sheet

Unfortunately, you will only find very vague information about the networking capabilities of EC2 instances within AWS’s service description and documentation. That is why I run a network performance benchmark for almost all EC2 instance types within the last few days. The results are compiled into the following cheat sheet.

The data analysis in the following table shows the burst and baseline throughput of each instance type. The burst throughput is defined as the 95th percentile, which means the throughput was at least reached for 3 minutes. The baseline throughput is defined as the 10th percentile, which means the throughput was reached at least for 54 minutes.

<!-- more -->

| Instance Type | Baseline (Gbit/s) | Burst (Gbit/s) |
|---------------|-------------------|----------------|
| c4.large      | 0.62              |                |
| c4.xlarge     | 1.24              |                |
| c4.2xlarge    | 2.48              |                |
| c4.4xlarge    | 4.96              |                |
| c4.8xlarge    | 9.85              |                |
| c5.large      | 0.74              | 10.04          |
| c5.xlarge     | 1.24              | 10.04          |
| c5.2xlarge    | 2.49              | 10.04          |
| c5.4xlarge    | 4.97              | 10.04          |
| c5.9xlarge    | 10.04             |                |
| c5.18xlarge   | 23.88             |                |
| d2.xlarge     | 1.24              |                |
| d2.2xlarge    | 2.48              |                |
| d2.4xlarge    | 4.96              |                |
| d2.8xlarge    | 9.85              |                |
| g3.4xlarge    | 4.99              | 10.09          |
| g3.8xlarge    | 10.09             |                |
| g3.16xlarge   | 22.7              |                |
| h1.2xlarge    | 2.48              | 10.09          |
| h1.4xlarge    | 4.99              | 10.09          |
| h1.8xlarge    | 10.09             |                |
| h1.16xlarge   | 22.07             |                |
| i3.large      | 0.74              | 10.09          |
| i3.xlarge     | 1.24              | 10.09          |
| i3.2xlarge    | 2.48              | 10.09          |
| i3.4xlarge    | 4.99              | 10.09          |
| i3.8xlarge    | 10.09             |                |
| i3.16xlarge   | 19.65             | 22.46          |
| i3.metal      | 22.05             | 24.16          |
| m3.medium     | 0.3               |                |
| m3.large      | 0.69              |                |
| m3.xlarge     | 0.99              |                |
| m3.2xlarge    | 0.99              |                |
| m4.large      | 0.45              |                |
| m4.xlarge     | 0.74              |                |
| m4.2xlarge    | 0.99              |                |
| m4.4xlarge    | 1.99              |                |
| m4.10xlarge   | 9.85              |                |
| m4.16xlarge   | 19.95             |                |
| m5.large      | 0.74              | 10.04          |
| m5.xlarge     | 1.24              | 10.04          |
| m5.2xlarge    | 2.49              | 10.04          |
| m5.4xlarge    | 4.97              | 10.04          |
| m5.12xlarge   | 10.04             |                |
| m5.24xlarge   | 21.49             |                |
| p2.xlarge     | 1.24              |                |
| p2.8xlarge    | 10.09             |                |
| p2.16xlarge   | 21.05             |                |
| p3.2xlarge    | 2.48              | 10.09          |
| p3.8xlarge    | 10.09             |                |
| p3.16xlarge   | 21.3              |                |
| r3.large      | 0.5               |                |
| r3.xlarge     | 0.69              |                |
| r3.2xlarge    | 0.99              |                |
| r3.4xlarge    | 1.98              |                |
| r3.8xlarge    | 4.96              |                |
| r4.large      | 0.74              | 10.09          |
| r4.xlarge     | 1.24              | 10.09          |
| r4.2xlarge    | 2.48              | 10.09          |
| r4.4xlarge    | 4.99              | 10.09          |
| r4.8xlarge    | 10.09             |                |
| r4.16xlarge   | 22.03             |                |
| r5.large      | 0.74              | 10.04          |
| r5.xlarge     | 1.24              | 10.04          |
| r5.2xlarge    | 2.49              | 10.04          |
| r5.4xlarge    | 4.97              | 10.04          |
| r5.12xlarge   | 12.04             |                |
| r5.24xlarge   | 23.51             |                |
| t1.micro      | 0.07              |                |
| t2.nano       | 0.03              | 0.28           |
| t2.micro      | 0.06              | 0.72           |
| t2.small      | 0.13              | 0.59           |
| t2.medium     | 0.25              | 0.65           |
| t2.large      | 0.51              | 0.78           |
| t2.xlarge     | 0.74              | 0.89           |
| t2.2xlarge    | 0.99              |                |
| t3.nano       | 0.03              | 5.06           |
| t3.micro      | 0.06              | 5.09           |
| t3.small      | 0.13              | 5.11           |
| t3.medium     | 0.25              | 4.98           |
| t3.large      | 0.51              | 5.11           |
| t3.xlarge     | 1.02              | 5.11           |
| t3.2xlarge    | 2.04              | 5.11           |
| x1.16xlarge   | 10.09             |                |
| x1.32xlarge   | 20.91             | 23.2           |
| x1e.xlarge    | 0.62              | 10.09          |
| x1e.2xlarge   | 1.24              | 10.09          |
| x1e.4xlarge   | 2.48              | 10.07          |
| x1e.8xlarge   | 4.99              | 10.09          |
| x1e.16xlarge  | 10.09             |                |
| x1e.32xlarge  | 22.13             |                |
| z1d.large     | 0.74              | 10.03          |
| z1d.xlarge    | 1.24              | 10.04          |
| z1d.2xlarge   | 2.49              | 10.04          |
| z1d.3xlarge   | 4.97              | 10.04          |
| z1d.6xlarge   | 12.05             |                |
| z1d.12xlarge  | 23.14             |                |
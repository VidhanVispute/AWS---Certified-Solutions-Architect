

---

# AWS Compute Services – Structured Overview

AWS Compute services are categorized based on workload type and deployment model.

---

## 1️⃣ Instance-Based Compute

Used when you manage virtual machines directly.

* **Amazon EC2 (Elastic Compute Cloud)**
  Resizable virtual servers in the cloud.

* **Amazon EC2 Spot Instances**
  Low-cost instances using unused AWS capacity (can be interrupted).

* **EC2 Auto Scaling**
  Automatically adjusts EC2 instance count based on traffic/load.

* **Amazon Lightsail**
  Simplified VPS solution for small applications.

* **AWS Batch**
  Fully managed batch processing workloads.

---

## 2️⃣ Container-Based Compute

Used for microservices and containerized applications.

* **Amazon ECS (Elastic Container Service)**
  AWS-managed container orchestration service.

* **Amazon ECR (Elastic Container Registry)**
  Docker image storage repository.

* **Amazon EKS (Elastic Kubernetes Service)**
  Managed Kubernetes service.

* **AWS Fargate**
  Serverless compute engine for containers (no server management).

---

## 3️⃣ Serverless Compute

No server management. Pay per execution.

* **AWS Lambda**
  Event-driven serverless compute service.

---

## 4️⃣ Edge & Hybrid Compute

Extends AWS infrastructure closer to users or on-premise environments.

* **AWS Outposts**
  AWS infrastructure deployed in your data center.

* **AWS Snow Family**
  Data transfer and edge computing devices.

* **AWS Wavelength**
  Ultra-low latency applications with 5G integration.

* **VMware Cloud on AWS**
  Run VMware workloads directly on AWS.

* **AWS Local Zones**
  Brings AWS services closer to metro areas for low latency.

---

## 5️⃣ Cost & Capacity Management

Used to optimize cost and infrastructure efficiency.

* **AWS Savings Plans**
  Discount model for predictable compute usage.

* **AWS Compute Optimizer**
  Provides recommendations to reduce cost and improve performance.

* **AWS Elastic Beanstalk**
  PaaS service to deploy applications without managing infrastructure.

* **EC2 Image Builder**
  Automates creation of custom AMIs.

* **Elastic Load Balancing (ELB)**
  Distributes traffic across multiple targets (EC2, containers, etc.).


# Disaster Recovery Plan - Thinkofliving

- [Overview](#overview)
- [System Information](#system-information)
  - [Service Components](#service-components)
- [Prerequisite](#prerequisite)
  - [Access Requirements](#access-requirements)
  - [Infrastructure](#infrastructure)
- [Upstream Dependant Services](#upstream-dependant-services)
- [Downstream Dependant Services](#downstream-dependant-services)
- [Disaster Scenarios and Recovery Procedures](#disaster-scenarios-and-recovery-procedures)
  - [Scenario A: Availability Zone Outage](#1-scenario-a-aws-single-az-outage)
  - [Scenario B: Individual Service Outage](#2-scenario-b-aws-individual-service-outage)
  - [Scenario C: Region Outage](#3-scenario-c-aws-region-outage)
  - [Scenario D: Data Loss](#4-scenario-d-data-loss)
- [Notes](#notes)
- [Appendix](#appendix)
  - [TOL Auto Scaling Configurations](#tol-auto-scaling-configurations)
  - [TOL CDN Distribution Table](#tol-cdn-distribution-table)

## Overview
The purpose of this plan is to list out steps involved in recovering the system components in ***TOL*** application. The system is hosted in Amazon AWS in a high available (multi-AZ) environment in ap-southeast-1 (Singapore) region.

## System Information
Below are the essential information related to TOL application.
 - [Architecture Diagram](https://iproperty.pagerduty.com/escalation_policies#PA1RTXS)
 - [1Pager Document](https://iproperty.pagerduty.com/escalation_policies#PA1RTXS)

#### Service Components

**1. Web Frontend (Container)**
The front end web application accessed by the general public or wide user base. Developed on React and NextJS framework.

**2. API (Container)**
The GraphQL API application that acts as an interface between the Web Frontend and backend data source. The data source comes from Elastic Search (Primary) and Wordpress API (Fallback). Developed on Nodejs.

**3. CMS/Wordpress API (Container)**
The wordpress application used by internal editors to maintain content which is displayed by the web frontend. Also consists of API which interacts with wordpress database. Runs on PHP.

**4. Wordpress Database (RDS)**
The MySQL database for the Wordpress application. This is where data originates and is the source of truth for the TOL system. This database is accessed by only the Wordpress API, not directly by any other application.

**5. Elastic Search (Elasticsearch Service)**
The scalable and responsive data source which indexes the JSON output from the WP API. This data source acts as an offset to the Wordpress API as the Wordpress API is unable to handle high loads effectively.

## Prerequisite

#### Access Requirements
Below are the access required to run the Disater Recovery Procedures.

| System  | Role |
| --------| ---- |
| AWS     | Production: tol-production-Privileged (103426260373) |

#### Infrastructure

The following table provides an overview of infra services running in the backup region.

| Global | Singapore (Primary) | Tokyo (DR) | Remarks |
| --------| ----------- | ----- | ----- |
| Route53| | | thinkofliving.com |
| CloudFront| | |
| Lambda Edge| | |
| WAF | | |
| | EC2 (Bastion Host)| EC2 (Bastion Host)| |
| | Fargate| Fargate| |
| | S3| S3| Assets: https://cdn-assets.prod.thinkofliving.com/ |
| | RDS/Aurora| RDS/Aurora| arn:aws:rds:ap-southeast-2:855953295449:db:thinkofliving |
| | Elasticsearch Service | | |
| | VPC & Subnet | VPC & Subnet| |
| | CloudWatch| CloudWatch| |
| | ALB| ALB| |
| | Lambda & API Gateway | Lambda & API Gateway | |
| | KMS| KMS| |


## Upstream Dependant Services
TOL application is relies on the following upstream services.

| Service | Description | Owner |
| ------ | ------ | ------ |
| Sharpie | Images served on the site | Platform |
| Locke | User login and registration | User Management (UM) |

## Downstream Dependant Services
There are no downstream dependant services in TOL.

## Disaster Scenarios and Recovery Procedures
Below are the procedure and scenarios captured for a disaster.
### Communication:
In case of any of the below disaster scenarios, please follow [incident process](https://garage.rea-group.com/documents/172793/developer-consumer-incident-process) and execute the recovery procedure in following order:
1. Send an **@channel** notification on Slack at **#asia-content-delivery-squad** and **#tol-ongoing**
2. Communicate to all relevant stakeholders
3. Create bunker slack channel for communications for the incident and DR
4. Assemble a team and execute recovery procedure depending on the particular disaster scenario as below:

### Scenarios:
- Scenario A: AZ failure in Singapore (ap-southeast-1a or ap-southeast-1b)
  - All services can be automatically failover to other healthy AZ.
- Scenario B: Individual service failure in Singapore (all availability zones)
  - Depending on which service. If ElasticSearch or RDS, application operate with minor functionality loss.
- Scenario C: Region failure in Singapore
  - Start up all services in Tokyo (ap-northeast-1) and divert live traffic.
- Scenario D: Data Loss (RDS)
  - Restore MySQL dump file from S3.

### 1. Scenario A: AWS Single AZ Outage

#### 1.1 Impact
**Minor** - due to automatic failover capability provided by AWS and handled by Auto Scaling Group settings.

#### 1.2 Estimated Time for Recovery
None required

#### 1.3 Procedure
Automatic. No manual intervention required.

### 2. Scenario B: AWS Individual Service Outage
In the event that a particular service is unavailable in the Singapore region.

#### 2.1 Impact
- **Minor**:
  - **Elastic Search**: No action required. API fail safe will automatically fall back to RDS.
  - **RDS**: No action required. Application can remain operable with certain functionality disabled as long as Elastic Search remains operable. 
  *Impaired functionality include: Comments, Free Text Search, Price Filter Search, Login, Register, User Profile*.
- **Severe**
  - **Other Service(s)**: Will need to be treated similar to Scenario C: AWS Region Outage.

#### 2.2 Estimated Time for Recovery
- **Elastic Search or RDS faillure**: no recovery time required.
- **Elastic Search and RDS faillure**: The recovery time is same as Scenario C.
- **Other service faillure**: The recovery time is same as Scenario C.

#### 2.3 Procedure
- **Elastic Search or RDS faillure**: Automatic. No manual intervention required.
- **Elastic Search and RDS faillure**: Please refer to Scenario C.
- **Other service faillure**: Please refer to Scenario C.

### 3. Scenario C: AWS Region Outage

#### 3.1 Impact
**Severe** - if both the Availability Zones (ap-southeast-1a and ap-southeast-1b) in Singapore region have an outage. All [Service Components](#service-components) have to be up and running in the backup Tokyo region in order to resume availability.

#### 3.2 Recovery Procedure
Estimated Time for Recovery: 30 minutes

**Activate AWS Tokyo Region and Divert Traffic**

  - 3.2.1. Upscale Elastic Search in backup region
  - 3.2.2. Upscale RDS in backup region
  - 3.2.3. Increase Fargate task count for Web Frontend, API, and CMS applications in backup region
  - 3.2.4. Update CloudFront origin to switch live traffic to backup region
  - 3.2.5. Invalidate CloudFront cache
  - 3.2.6. Verify that site is up and running

  **3.2.1. Upscale Elastic Search in backup region**

  a. Login to AWS Console in Production, then switch to Tokyo Region.
  ![RDS](assets/TOL-DR-AWS-RoleSelection.png?raw=true "RDS")

  b. Go to **Services > Elasticsearch Service**, then click on the domain `th-thin-elasti...`.
  ![RDS](assets/TOL-DR-ES.png?raw=true "RDS")

  c. On the top left, click on `Edit Domain`.
  ![RDS](assets/TOL-DR-ES-EditDomain.png?raw=true "RDS")

  d. Scroll down to **Data nodes** section and set the data nodes according to the configuration shown:
  ![RDS](assets/TOL-DR-ES-DataNodes.png?raw=true "RDS")

  e. Scroll down to **Dedicated master nodes** section and set the master nodes according to the configuration shown:
  ![RDS](assets/TOL-DR-ES-MasterNodes.png?raw=true "RDS")

  e. Scroll down to bottom and click **Submit** button.

  f. Complete. Move to the next step.

  **3.2.2. Upscale RDS in backup region**

  a. Login to AWS Console in Production, then switch to Tokyo Region.
  ![RDS](assets/TOL-DR-AWS-RoleSelection.png?raw=true "RDS")

  b. Go to **Services > Amazon RDS**. In Amazon RDS, click on **DB Instances**
  ![RDS](assets/TOL-DR-RDS.png?raw=true "RDS")

  b. Select the DB Identifier **th-thinkofliving-prod-rds** then click the Modify button
  ![RDSModify](assets/TOL-DR-RDS-Modify.png?raw=true "RDSModify")

  c. In Modify Database, scroll to DB Instance size, then select "Memory Optimized classes" and choose from the list **db.r5.large**. Scroll down and click the "Continue" button.
  ![RDSScaleUp](assets/TOL-DR-RDS-ScaleUp.png?raw=true "RDSScaleUp")

  d. Complete. Move to the next step.

  **3.2.3. Increase Fargate task count Web Frontend application**
  
  a. Login to AWS Console in Production, then switch to Tokyo Region.
  ![RDS](assets/TOL-DR-AWS-RoleSelection.png?raw=true "RDS")
  
  b. Go to **Services > Elastic Container Service**. In Clusters, click on 'ecs-cluster-tol-blue' cluster
  ![ECS Clusters](assets/TOL-DR-Clusters.png?raw=true "ECS Clusters")

  c. Under the Services tab, click on **tol-frontend-blue-EcsService...**
  ![ECS Services](assets/TOL-DR-ClusterServices.png?raw=true "ECS Services")

  d. Click the 'Update' button on the top right of services page
  ![Update Service](assets/TOL-DR-UpdateService.png?raw=true "Update Service")

  e. In Step 1 and 2, scroll to the bottom and click 'Next' until you reach the 'Step 3: Set Auto Scaling (optional) page. On this page, set the task counts according to the following table: [TOL Auto Scaling Configurations](#tol-auto-scaling-configurations). When done, scroll to the bottom and ckick 'Next'
  ![Update Scaling](assets/TOL-DR-UpdateServiceScaling.png?raw=true "Update Scaling")

  f. In Step 4, scroll to the bottom and click on 'Update Service'
  ![Confirm Update](assets/TOL-DR-UpdateServiceConfirm.png?raw=true "Confirm Update")

  g. Repeat steps (c) to (f) with the following ECS services: **tol-api-blue-EcsService...** and **tol-wordpress-blue-EcsService...**
    
  **3.2.4. Update CloudFront origin**

  a. Login to AWS Console in Production, then switch to Tokyo Region.
  ![RDS](assets/TOL-DR-AWS-RoleSelection.png?raw=true "RDS")
  
  b. Go to **Services > CloudFront** In CloudFront Distributions, select the checkbox on the same row with Comment **thinkofliving-wordpress**. Then click on `Distribution Settings`
  ![CloudFront](assets/TOL-DR-CF.png?raw=true "CloudFront")

  c. In Distribution Settings, go to the `Origins and Origin Groups` tab, select the checkbox next to the first row then click on `Edit`
  ![CloudFront Behaviors](assets/TOL-DR-CF-Origins.png?raw=true "CloudFront Behaviors")

  d. In Edit Origin, change the Origin Domain Name to **wordpress-active-tk.prod.thinkofliving.com**. Then scroll to the bottom and click on 'Yes, Edit' button.
  ![CloudFront Edit Behavior](assets/TOL-DR-CF-EditOrigins.png?raw=true "CloudFront Edit Behavior")

  e. Repeat step (b) to (d) for each distribution in the [TOL CDN Distribution Table](). Set the origin according to the DR (Tokyo) region origin.

  **3.2.5. Invalidate CloudFront cache**

  a. Login to AWS Console in Production, then switch to Tokyo Region.
  ![RDS](assets/TOL-DR-AWS-RoleSelection.png?raw=true "RDS")

  b. Go to **Services > CloudFront** In CloudFront Distributions, select the checkbox on the same row with Comment **thinkofliving-wordpress**. Then click on 'Distribution Settings'
  ![CloudFront](assets/TOL-DR-CF.png?raw=true "CloudFront")

  c. In Distribution Settings, go to the 'Invalidations' tab, then click on 'Create Invalidation'
  ![CloudFront Create Invalidation](assets/TOL-DR-CF-CreateInvalidation.png?raw=true "CloudFront Create Invalidation")

  d. Enter "*" (without quotes) into the Object Paths textbox. Then click on 'Invalidate'
  ![CloudFront Invalidation](assets/TOL-DR-CF-Invalidate.png?raw=true "CloudFront Invalidation")

  e. Repeat steps (b) to (d) with the following CloudFront Distributions: **thinkofliving** and **thinkofliving-frontend**

  **3.2.6. Verify that site is up and running**
  Visit thinkofliving.com and verify that the site is up and running. Take note that CDN cache invalidation can take time.

#### 3.3 Reverting the Backup Region

When the backup region is no longer required, take the following steps to switch traffic back up the primary region and hibernate the backup region.

  - 3.3.1. Update CloudFront origin to switch live traffic to primary region
  - 3.3.2. Invalidate CloudFront cache
  - 3.3.3. Verify that site is up and running
  - 3.3.4. Downscale RDS in backup region
  - 3.3.4. Downscale Elastic Search in backup region
  - 3.3.5. Decrease Fargate task count for Web Frontend, API, and CMS applications in backup region

### 4. Scenario D: Data Loss

#### 4.1 Recovery Procedure
In the event that some critical data is discovered to be missing from the source of truth, use the following procedure for data recovery.

  - 4.1.1. Restore MySQL backup dump file from S3 into Wordpress DB.

## Notes

#### Application

Web:
- File `.env` values `PROD_ASSETS_URL_PREFIX` and `API_URL`
  No changes required for the following as the endpoint is calling CDN. The origin of the CDN will be updated to DR region, so the endpoint remains the same.

- File `src/common/config/img.js`
  - `imgSharpieUrls` no changes required as the endpoint is calling CDN.
  - `imgOriginUrls` no changes required as the endpoint is calling CDN.

- Ensure that all functionality can operate without Elastic Search

API:
- Turn off ES for the build that is deployed to DR region

#### Deployment Pipeline

The BuildKite pipeline `tol-build-deploy-images` should support deployments to DR region in the `Unblock Production` step.

#### Data Backup
Set up daily MySQL dump at 3AM (Kuala Lumpur Time) for the Wordpress database and store the dump file in S3. Configure auto-cleanup of dump files after 30 days old.

#### Infra Diagram
 ![Infra Diagram](assets/TOL-DR-Diagram.png?raw=true "Infra Diagram")


## Appendix

##### TOL Auto Scaling Configurations
| Application | Min Task | Desired Task | Max Task |
| --------| ----------- | ----- |-----|
| tol-frontend-blue-EcsService | 8| 8| 20|
| tol-api-blue-EcsService | 2| 2| 2|
| tol-wordpress-blue-EcsService | 2| 2| 2|

##### TOL CDN Distribution Table

| Distribution ID | Comment | Origin ID | Origin Domain (SG) | Origin Domain (TK) |
| -------- | -------- | -------- | -------- | -------- |
| E1434GUJYD7YYF | thinkofliving-wordpress | Public-Wordpress | wordpress-active.prod.thinkofliving.com | wordpress-active-tk.prod.thinkofliving.com |
|  | | Public-S3 | tol-public-content-prod.s3.amazonaws.com | tol-public-content-prod-tk.s3.amazonaws.com |
| E3E3UNXKAIAKTT | thinkofliving-frontend | Public-Wordpress | wordpress-active.prod.thinkofliving.com | wordpress-active-tk.prod.thinkofliving.com |
|  | | Public-S3 | tol-public-content-prod.s3.amazonaws.com | tol-public-content-prod-tk.s3.amazonaws.com |
|  | | Public-Frontend | frontend-active.prod.thinkofliving.com | frontend-active-tk.prod.thinkofliving.com |
| E2MZT0T360DFAQ | thinkofliving | Public-S3 | tol-public-content-prod.s3.amazonaws.com | tol-public-content-prod-tk.s3.amazonaws.com |

**Distributions**:
 ![CloudFront Distributions](assets/TOL-DR-CF-Distributions.png?raw=true "CloudFront Distributions")

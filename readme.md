# Problem Statement

The organization currently operates 30+ application workloads across on-premises servers and maintains 40+ pipeline jobs on Databricks with manual code uploads and no automated testing. With 25+ repositories across the engineering organization, we lack:

1. **Application Migration Strategy** - Need a defined solution and roadmap for migrating on-premises servers and schedulers to a cloud-native platform
2. **Automated Deployment Pipeline** - Require an automated CI/CD solution for deploying pipeline code to Databricks to replace manual uploads
3. **Infrastructure Consolidation Plan** - Need a phased migration strategy to standardize infrastructure across mixed environments (container instances, clusters, on-premises servers)

This case study proposes solutions to modernize our infrastructure, improve deployment reliability, and achieve unified workload management across platforms.

## Migration Operating Model

These whiteboard presentations illustrate the operating model for migrating workloads to the cloud, encompassing key phases such as assessment, planning, execution, and optimization. Each phase includes specific activities and deliverables that guide the migration process, ensuring a structured and efficient transition to the new environment.

### Phases of the Migration Operating Model

1. Build Platform Foundation
   - Build Dev/Stg/Prod environments & GitOps Backbone
2. Databricks Automation
   - Automate CI/CD for all 40+ pipeline jobs
3. Workload Migration
   - Migrate workloads in phases based on risk and complexity to AKS


### Comprehensive Migration Plan
- [Major Operating Streams](images/majorstreamoperatingmodel.jpg)
- [Product Roadmap](images/productroadmap.jpg)
- [Generic Deployment Lifecycle for all workloads](images/lifecycle.jpg)
- [Workload Classification & Migration Strategy](images/workloadclassification.jpg)

### Application Workload & On Prem Scheduler Detailed Plan
- [Detailed Build Sequence Plan for Application Workloads & On-Prem Schedulers Part 1](images/aks/detailedbuildsequencepart1.jpg)
- [Detailed Build Sequence Plan for Application Workloads & On-Prem Schedulers Part 2](images/aks/detailedbuildsequencepart2.jpg)
- [AKS Architecture Layer Diagram](images/aks/aksarchitecturenetwork.jpg)
- Separation of Concerns for IAC & CI/CD
   - [IAC vs Service Deployment Strategy](images/aks/iacvsservicedeploymentstrategy.jpg)
   - [IAC vs Service Deployment Pipeline Flow](images/aks/iacvsservicedeploymentpipelineflow.jpg)

### Databricks Migration Detailed Plan
- [Detailed Proposal Plan for Databricks Migration](images/databricks/databricksprojectplan.jpg)
- [Architecture Diagram DatabricksProjectPlan](images/databricks/databricksarchitectureownership.jpg)
- [Separation of Concerns for IAC vs Service Deployment](images/databricks/databricksiacvsservicedeploymentstrategy.jpg)
- [Databricks CI/CD Pipeline for Application Repositories](images/databricks/databricksiacvsservicedeploymentpipelineflow.jpg)
- [Databricks CI/CD Workspace](images/databricks/databrickscicdworkspace.jpg)

### Conclusion
- [Deployment Strategy](images/deploymentstrategy.jpg)
- [Build vs Buy](images/buildvsbuy.jpg)

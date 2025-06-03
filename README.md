# Deploy Kubernetes Resource ไปยัง Amazon EKS โดยใช้ Azure DevOps
Pattern นี้มีจุดประสงค์เพื่อ Guide วิธีการ Deploy App แบบ Container ไปยัง Amazon EKS Cluster จาก Azure DevOps โดยใช้ Helm Chart 
Pattern นี้สามารถขยายเพิ่มเติมได้โดยการปรับแต่ง Pipeline Template นี้ เพื่อใช้ Azure Pipeline Service Connection สำหรับ AWS ในการ Query และใช้ข้อมูลจาก AWS Cloud

## สิ่งที่ต้องเตรียมไว้ก่อน
- AWS Account
- Amazon EKS Cluster พร้อม Node Instance Role เพื่อ Pull Image จาก ECR
- IAM User Account ที่มีสิทธิ์เข้าถึง Amazon EKS Cluster
- Azure DevOps Account
- AWS Toolkit สำหรับ Azure DevOps ที่ติดตั้งใน Azure DevOps หรือบน On-premises Azure DevOps Server
- App ที่จะ Deploy (มีตัวอย่าง Web App ใน Guide นี้)

## เริ่มต้นใช้งาน

## Architecture เป้าหมาย
![Target Architecture](./docs/img/Architecture.png "Target Architecture")
## เกี่ยวกับ Pipeline Template
Pipline Template มีด้วยกัน 3 Template:

1. Template หลัก - [main_template.yaml](./pipeline_templates/main_template.yaml)

    Template นี้ ทำหน้าที่เป็น Template กลางที่อ้างอิงไปยัง Template อื่นๆ นอกจากนี้ยังมี Stage สำหรับการกำหนดค่าตัวแปร และตรวจสอบการมีอยู่ของ AWS resource (Amazon EKS Cluster และ ECR Repo) ตั้งแต่เริ่ม Excute Pipeline

    >นี่เป็น template เดียว ที่อาจใช้อ้างอิงเมื่อใช้ Pipeline Template ใน Solution นี้

2. CI Job Template - [ci_template.yaml](./pipeline_templates/job_templates/ci_template.yaml)

    คือ Template ที่ถูกเรียกใช้โดย `main_template.yaml` สำหรับ:
    - Build Docker Image
    - Push Docker Image
    - Build Helm Chart
    - Push Helm

3. CD Job Template - [cd_template.yaml](./pipeline_templates/job_templates/cd_template.yaml)

    คือ Template อีกตัวนึงที่ถูกเรียกใช้โดย `main_template.yaml` ซึ่งขึ้นอยู่กับการดำเนินการที่สำเร็จของ Stage `CI` และ `Init` ใน `main_template.yaml` 
    
    ใช้สำหรับ:
    - Deploy Helm Chart
    - แสดงผลลัพธ์การ Deploy

Template เหล่านี้ถูกจัดอยู่ใน Directory โครงสร้างแบบนี้:
```
pipeline_templates/
├─ jobs_templates/
│  ├─ ci_template.yaml
│  ├─ cd_template.yaml
├─ main_template.yaml
```

## วิธีการตั้งค่า Pipeline Template
เพื่อใช้ Pipeline Template ทำตามขั้นตอนนี้:
1. **Copy File ต่อไปนี้จาก Repo นี้ ไปยัง Azure DevOps Repo**
    * Copy Pipeline Templates Folder
        - [pipeline_templates](./pipeline_templates/) Folder (รวม File ใน Sub-folder)
    * Copy ตัวอย่าง Pipeline ที่อ้างอิง Pipeline Template
        - [azure_pipeline.yaml](./azure_pipeline.yaml) File
    
    ![image](https://github.com/user-attachments/assets/75877316-29e9-4941-85f2-d92a8940f80b)

2. **ตั้งค่า Pipeline ใหม่ใน Azure DevOps**
    * ไปที่ Pipelines > Create Pipeline
    * ทำตามขั้นตอนของ Wizard โดยเลือก Azure Repos Git (YAML) as the location of your source code.

      ![image](https://github.com/user-attachments/assets/ea41da05-545a-4005-b1a5-66285e50844a)

    * เมื่อเห็น List ของ Repos ให้เลือก Repo ข้อ 1.
    * เลือก Existing Azure pipelines YAML file ในส่วน Configure your pipeline

      ![image](https://github.com/user-attachments/assets/0476cf5c-b73f-4bb8-8aa8-510a5ce73c7a)

    * เลือก Branch master หรือ main and the path as: /azure_pipeline.yaml. Click on the continue button
    * Replace the input parameter values for the mandatory fields in azure_pipeline.yaml to your needs:
        ```
        serviceConnectionName: Azure DevOps Service Connection name.
        awsRegion: Default region for AWS.
        awsEKSClusterName: Name of the Amazon EKS Cluster used for deployment.
        projectName: Name of the project. This should be same as the Helm chart name
        ```
    * Click on the dropdown menu beside the Run button and save the pipeline
3. **Run the pipeline**
    * Go to Pipelines, and then select the pipeline you just created
    * Click on the Run pipeline button
    * Click on the Run button

## Pipeline template parameters
### Input parameters

| Name                       | Description                                                     | Default Value         |
| -------------------------  | ----------------------------------------------------------------| --------------------- |
| `serviceConnectionName`    | Azure DevOps Service Connection Name                            |                       |
| `awsRegion`                | Default region for AWS                                          |                       |
| `awsEKSClusterName`        | Name of the Amazon EKS Cluster                                  |                       |
| `awsEKSRegion`             | Region for EKS Cluster                                          | `${{ parameters.awsRegion }}` |
| `awsECRRegion`             | Region for ECR repository                                       | `${{ parameters.awsRegion }}` |
| `awsECRAccountId`          | Account ID where ECR Repository is created                      | `AccountID of Service Connection` |
| `projectName`              | Name of the project. Used in naming of K8s resources & Helm Chart  | `"webapp"` |
| `K8sNamespace`             | K8s Namespace for deployment                                    | `${{ parameters.projectName }}` |
| `helmVersion`              | Version of Helm                                                 | `"3.8.2"` |
| `imageName`                | Name of the container image                                     | `${{ parameters.projectName }}` |
| `imageTag`                 | Image Tag for the container image to be generated               | `$(Build.BuildNumber)-image` |
| `helmChartVersion`         | Version number for the helm chart to be created                 | `$(Build.BuildNumber)-helm`  |
| `helmChartDirPath`         | Path of the directory containing the chart to be packaged, eg webapp/charts/webapp | `./${{ parameters.projectName }}/charts/${{ parameters.projectName }}` |
| `dockerfilePath`           | Path of the Dockerfile, eg webapp/Dockerfile.                   | `./${{ parameters.projectName }}/Dockerfile` |

## Limitations
- Amazon EKS cluster is publicly available and may not suit all architectures.
    - For Private EKS cluster, please refer to Azure DevOps Self-Hosted Agents: Self-hosted Linux agents
- To use AWS Toolkit for Azure DevOps for accessing AWS services, you need an AWS account and AWS credentials.
- [Sample web app](./webapp/charts/README.md) provided as a part of this pattern is **only for example purpose**. It is a web server (httpd), hosting a HTML file (index.html). The application is exposed via public load balancer with a Kubernetes service of type `Loadbalancer` over **HTTP**.

## Security

See [CONTRIBUTING](CONTRIBUTING.md#security-issue-notifications) for more information.

## License

This library is licensed under the MIT-0 License. See the [LICENSE](./LICENSE) file.

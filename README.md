# AWS Systems Manager

## Lab Overview

## 시작하기전에

1. 본 Hands-on lab은 AWS Seoul region 기준으로 작성되었습니다. Region을 Seoul (ap-northeast-2)로 변경 후 진행 부탁드립니다.
2. [AWS Credit 추가하기](https://aws.amazon.com/ko/premiumsupport/knowledge-center/add-aws-promotional-code/)
3. [Lab 환경 구축](https://ap-northeast-2.console.aws.amazon.com/cloudformation/home?region=ap-northeast-2#/stacks/quickcreate?templateURL=https://saltware-aws-lab.s3.ap-northeast-2.amazonaws.com/ssm/ssm.yaml&stackName=ssm-lab)

## Managed-Instance Activation

### AMI with SSM Agent preinstalled

1. AWS Management Console에서 좌측 상단에 있는 **[Services]** 를 선택하고 검색창에서 IAM를 검색하거나 **[Security, Identity, & Compliance]** 바로 밑에 있는 **[IAM]** 를 선택

2. **[Roles]** &rightarrow; *ssm-lab-EC2Role-xxxx*를 선택 &rightarrow; **[Attach policies]** :white_check_mark: *AmazonEC2RoleForSSM* &rightarrow; **[Attach policy]** &rightarrow;

3. AWS Management Console에서 좌측 상단에 있는 **[Services]** 를 선택하고 검색창에서 Systems Manager를 검색하거나 **[Management & Goverrnance]** 밑에 있는 **[Systems Manager]** 를 선택

4. 왼쪽 패널에서 **Managed Instances** 선택하고 4개의 인스턴스들이 Managed Instance로 등록됬는지 확인

### AMI without SSM pre-configured (On-premise)

1. *Systems Manager* Dashboard 왼쪽 패널에서 **Hybrid Activations** 선택 &rightarrow; **[Create an Activation]**

2. **Activation description** = *Onprem App Server*, **Instance limit** = *1*, **IAM role** = :radio_button: *Select an existing custom IAM role that has the required permissions* &rightarrow; ssm-lab-OnpremRole-xxx 선택 &rightarrow; **Default instance name** = *OnpremApp* &rightarrow; **[Create activation]**

3. **Activation Code** 와 **Activation ID** 를 메모

4. *OnpremApp* 인스턴스에 SSH 접속

5. SSM Agent 설치 및 설정 - [AWS Document](https://docs.aws.amazon.com/systems-manager/latest/userguide/sysman-install-managed-linux.html)

    ```bash
    mkdir /tmp/ssm
    curl https://s3.amazonaws.com/ec2-downloads-windows/SSMAgent/latest/linux_amd64/amazon-ssm-agent.rpm -o /tmp/ssm/amazon-ssm-agent.rpm
    sudo yum install -y /tmp/ssm/amazon-ssm-agent.rpm
    sudo systemctl stop amazon-ssm-agent
    sudo amazon-ssm-agent -register -code "activation-code" -id "activation-id" -region "region"
    sudo systemctl start amazon-ssm-agent
    ```

6. **Managed Instances** 에 *OnpremApp* 선택 &rightarrow; **Tags** 에 아래와 같은 태그 추가
    | Key | Value |
    |-----|-------|
    | Env | Prod  |
    | App | SSM   |

## Inventory

### S3 버킷 구성

1. AWS Management Console에서 좌측 상단에 있는 **[Services]** 를 선택하고 검색창에서 S3를 검색하거나 **[Storage]** 바로 밑에 있는 **[S3]** 를 선택

2. S3 Dashboard에서 **[Create bucket]** 클릭 &rightarrow; **Bucket name** = *ssm-lab-inventory-[임의의 문자 및 숫자]*, **Region** = *Asia Pacific (Seoul)* &rightarrow; **[Create]**

3. IAM Dashboard 에서 *ssm-lab-EC2Role-xxxx* 과 *ssm-lab-OnpremRole-xxxx*에 아래와 같은 Inline policy 추가

    ```json
    {
        "Version": "2012-10-17",
        "Statement": [
            {
                "Effect": "Allow",
                "Action": [
                    "s3:GetObject",
                    "s3:PutObject",
                    "s3:PutObjectAcl"
                ],
                "Resource": "arn:aws:s3:::<S3_BUCKET_NAME>/*"
            }
        ]
    }
    ```

### Inventory 구성

1. *Systems Manager* Dashboard 왼쪽 패널에서 **Inventory** 선택 &rightarrow; **[Setup Inventory]**

2. **Name** = *Inventory-Association-SSM*, **Targets** = :radio_button: *Specifying a tag* &rightarrow; **tag key** = *App*, **tag value** = *SSM* &rightarrow; Collect inventory data every **[ 1 ]** **[ Day(s) ]**

3. :white_check_mark: Sync inventory execution logs to an S3 bucket &rightarrow; **S3 bucket name** = *위에서 생성한 버킷 이름*, **S3 bucket prefix** = *ExecutionLogs* &rightarrow; **[Setup Inventory]**

4. **Inventory** 현황이 업데이트 됬는지 확인

5. **Managed Instances** 에 *ProdApp* 선택 &rightarrow; **Inventory** 를 통해서 어떤 Software 들이 설치되어 있는지 확인

### Inventory Data 저장

1. Inventory execution logs 가 저장되는 S3 버킷에 아래와 같은 Bucket Policy 적용

    ```json
    {
        "Version":"2012-10-17",
        "Statement":[
            {
                "Sid":"SSMBucketPermissionsCheck",
                "Effect":"Allow",
                "Principal":{
                    "Service":"ssm.amazonaws.com"
                },
                "Action":"s3:GetBucketAcl",
                "Resource":"arn:aws:s3:::<BUCKET_NAME>"
            },
            {
                "Sid":" SSMBucketDelivery",
                "Effect":"Allow",
                "Principal":{
                    "Service":"ssm.amazonaws.com"
                },
                "Action":"s3:PutObject",
                "Resource":"arn:aws:s3:::<BUCKET_NAME>/*",
                "Condition":{
                    "StringEquals":{
                    "s3:x-amz-acl":"bucket-owner-full-control"
                    }
                }
            }
        ]
    }
    ```

2. *Systems Manager Inventory* Dashboard 에서 **Resource Data Syncs** 선택 &rightarrow; **[Create resource data sync]** &rightarrow; **Sync name** = *SSM-Inventory*, **Bucket name** = *위에서 생성한 버킷 이름*, **Bucket prefix** = *SSM-Inventory* &rightarrow; **[Create]**

3. S3 버킷으로 이동해서 위에서 명시한 prefix 아래 각 Components 별 데이터가 저장 됬는지 확인

## Session Manager

1. AWS Management Console에서 좌측 상단에 있는 **[Services]** 를 선택하고 검색창에서 EC2를 검색하거나 **[Compute]** 바로 밑에 있는 **[EC2]** 를 선택

2. 왼쪽 패널에서 **Instances** 선택 &rightarrow; :white_check_mark: *TestApp* &rightarrow; **[Connect]** &rightarrow; :radio_button: *Session Manager* &rightarrow; **[Connect]**

3. 아래의 명령어로 ClamAV 설치

    ```bash
    sudo amazon-linux-extras install epel
    sudo yum update
    sudo yum install clamav
    ```

## State Manager

1. *Systems Manager* Dashboard 왼쪽 패널에서 **State Manager** 선택

2. *Inventory-Association-SSM* 을 선택하고 **[Apply association now]**

3. *Systmes Manager Dashboard* 에서 **Managed Instances** 선택 &rightarrow; *TestApp* 선택 &rightarrow; **Inventory** 에 clamav가 포함되어 있는지 확인

## Documents

1. *Systems Manager* Dashboard 왼쪽 패널에서 **Documents** 선택 &rightarrow; **[Create command or session]**

2. **Name** = *Onprem-ClamAVInstall*, **Target type** = */AWS::SSM::ManagedInstance*, **Document type** = *Command document*, :radio_button: YAML 선택후 아래의 스크립트 붙여넣기 &rightarrow; **[Create document]**

    ```yaml
    schemaVersion: "2.2"
    description: "Install ClamAV"
    mainSteps:
      -
        action: "aws:runShellScript"
        name: "clamav_install"
        inputs:
          runCommand:
            - "sudo yum update -y"
            - "sudo yum install epel-release -y"
            - "sudo yum install clamav -y"
    ```

## Run Command

1. *Systems Manager* Dashboard 왼쪽 패널에서 **Run Command** 선택 &rightarrow; **[Run Command]**

2. **Command document** = :radio_button: *Onprem-ClamAVInstall*, **Targets** = :radio_button: *Choose instances manually*, :white_check_mark: *OnpremApp* &rightarrow; **[Run]**

3. **State Manager** 를 통해서 *Inventory* 정보를 업데이트하고 해당 Managed Instance의 Inventory 목록에 *clamav* 가 추가되었는지 확인

## Patch Manager

1. *Systems Manager* Dashboard 왼쪽 패널에서 **Patch Manager** 선택 &rightarrow; **[Configure patching]**

2. **How do you want to select instances?** = :radio_button: *Enter instance tags*
    | Key | Value |
    |-----|-------|
    | Env | Prod  |

    **[Add]**

3. **How do you want to specify a patching schedule?** = :radio_button: *Skip scheduling and patch instances now*

    > Maintenance Windows를 생성해서 특정 시간 또는 특정 간격으로 패치를 자동으로 실행되게 설정할수 있습니다. Hands-on Lab 의 특성상 바로 결과를 확인해야 하므로 따로 스케쥴링 설정을 하지 않습니다. Maintenance Windows 설정은 해당 [문서](https://docs.aws.amazon.com/systems-manager/latest/userguide/sysman-maintenance-working.html)를 참고해 주시기 바랍니다.

4. **Patching operation** = :radio_button: *Scan and install*

5. **[Configure patching]**

6. *Systems Manager* Dashboard 왼쪽 패널에서 **Run Command** 선택 &rightarrow; **Document name** 이 *AWS-RunPatchBaseline* 인 커맨드가 실행되고 있음을 확인 &rightarrow; **Status** 가 :white_check_mark: Success 로 변경되면 각 Managed Instance 에 실행된 Command 에 대한 Output 확인

    > Output 에 보면 **Step name**에서 *PatchWindows* 와 *PatchLinux* 가 있습니다. 해당 Managed Instance 의 타입에 맞는 Command 에 대한 Output 확인

## Create an AMI with Systems Manager Automation

1. 해당 [문서](https://docs.amazonaws.cn/en_us/systems-manager/latest/userguide/automation-cf.html)를 참고해서 Systems Manager Automation 에 필요한 IAM roles 생성

2. *Systems Manager* Dashboard 왼쪽 패널에서 **Automation** 선택 &rightarrow; **[Execute automation]**

3. **Automation document** 에  *AWS-UpdateLinuxAMI* 선택 &rightarrow; **[Next]** &rightarrow; **SourceAmiId** = *ami-0a93a08544874b3b7* &rightarrow; **[Execute]**

4. *Automation Dashboard* 를 통해서 각 Action 들이 실행되는걸 확인하고 마지막 스탭이 (*terminateInstance*) 완료되면 EC2 Dashboard 로 이동해서 AMI가 생성됬는지 확인

## Daily AMI back up with Maintenance Windows

1. *Systems Manager* Dashboard 왼쪽 패널에서 **Maintenance Windows** 선택 &rightarrow; **[Create maintenance window]**

2. **Name** = *daily_ami_backup*, **Windows start** = :radio_button: Every [ **Day** ] at [ **00:00** ], **Duration** = *1*, **Stop initiating tasks** = 0, **Schedule timezone** = *(GMT+09:00 Asia/Seoul)* &rightarrow; **[Createe maintenance window]**

3. 생성된 Maintenance Windows를 선택 &rightarrow; **Tasks** &rightarrow; **[Register tasks]** &rightarrow; **[Register Automation task]** &rightarrow; **Automation document** 에 *AWS-UpdateLinuxAMI* 를 선택하고 위와 동일한 Parameter 값들을 입력 &rightarrow; **[Register Automation task]**

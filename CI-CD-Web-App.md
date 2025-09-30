# AWS CI/CD Web App

![Image Alt](https://github.com/fredcodess/AWS-Projects/blob/main/images/cicd-expanded.png?raw=true)

---

# Project layout

```
.
├── settings.xml              # Maven settings (for CodeArtifact)
├── buildspec.yml             # CodeBuild build spec
├── appspec.yml               # CodeDeploy appspec
├── scripts/                  # install/start/stop/deploy helper scripts for CodeDeploy
│   ├── install_dependencies.sh
│   ├── stop_server.sh
│   └── start_server.sh
├── index.jsp                 
├── pom.xml                   
```

![Image Alt](https://github.com/fredcodess/AWS-Projects/blob/main/images/cicd-project-layout.png?raw=true)

---

# 1) Launch EC2, connect with VS Code Remote-SSH, and connect to GitHub

**Goal:** create a dev instance you use to edit and push code.

### Step 1.1 — Launch an EC2 instance

1. Console → EC2 → Launch instance:

   * AMI: **Amazon Linux 2023**
   * Instance type: `t2.micro` or appropriate
   * Key pair: create or choose `EC2_KEYPAIR_NAME` (download `.pem`)
   * Security group: allow **SSH (22)** from your IP, **HTTP (80)** (and 443 if needed) from 0.0.0.0/0
   * IAM role: if you want the instance to access S3/CodeArtifact directly, attach an instance profile created later.
2. Launch.

![Image Alt](https://github.com/fredcodess/AWS-Projects/blob/main/images/cicd-ec2-create.png?raw=true)
![Image Alt](https://github.com/fredcodess/AWS-Projects/blob/main/images/cicd-pem.png?raw=true)

---

### Step 1.2 — Connect via SSH (terminal)

```bash
# on your laptop
chmod 400 ~/Desktop/EC2_KEYPAIR_NAME.pem
ssh -i ~/Desktop/EC2_KEYPAIR_NAME.pem ec2-user@<EC2_PUBLIC_IP>
```

![Image Alt](https://github.com/fredcodess/AWS-Projects/blob/main/images/cicd-chmod-pem.png?raw=true)

---

### Step 1.3 — Configure VS Code Remote-SSH

1. In VS Code: `F1 → Remote-SSH: Add New SSH Host...`

   * SSH command: `ssh -i ~/path/EC2_KEYPAIR_NAME.pem ec2-user@<EC2_PUBLIC_IP>`
   * Choose config file (usually `~/.ssh/config`).
2. Connect: `Remote-SSH: Connect to Host...` and open your project folder.

---

### Step 1.4 — Prepare the instance (install tools)

SSH into EC2 / VS Code terminal and run:

```bash
# update + basic dev tools (Amazon Linux 2 example)
sudo yum update -y
# install git, java, maven example
sudo yum install -y git java-11-amazon-corretto-devel maven unzip
```

**Screenshot to save:** `./screenshots/instance-setup.png` (terminal showing installed versions).

---

### Step 1.5 — Connect to GitHub

** HTTPS with a personal access token** (less nice for remote builds).

![Image Alt](https://github.com/fredcodess/AWS-Projects/blob/main/images/cicd-github-token.png?raw=true)

---

# 2) Set up CodeArtifact and give your web app access

**Goal:** host private packages/artifacts and let your build & runtime pull dependencies.

### Step 2.1 — Create CodeArtifact domain & repository

Console: **AWS CodeArtifact → Create domain → domain=`CODEARTIFACT_DOMAIN`**.
Then **Create repository → repository=`CODEARTIFACT_REPO`** and link it to the domain.

**Screenshot to save:** `./screenshots/codeartifact-repo.png`.

---

### Step 2.2 — Create an IAM role / policy for access

Create a role that your **web app** (EC2 instance or CodeBuild) will assume, or attach to instance profile. Minimal trust for EC2:

**Trust relationship** (for instance profile):

```json
{
  "Version": "2012-10-17",
  "Statement": [
    { "Effect": "Allow", "Principal": { "Service": "ec2.amazonaws.com" }, "Action": "sts:AssumeRole" }
  ]
}
```

For CodeBuild, trust principal is `codebuild.amazonaws.com`. For CodePipeline, `codepipeline.amazonaws.com`, etc.

**Example (broad) policy for CodeArtifact read** — **adjust to least privilege**:

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "codeartifact:GetAuthorizationToken",
        "codeartifact:GetRepositoryEndpoint",
        "codeartifact:ReadFromRepository",
        "codeartifact:GetPackageVersion"
      ],
      "Resource": "*"
    }
  ]
}
```

Attach to the role used by EC2/CodeBuild/CodePipeline as needed.

**Screenshot to save:** `./screenshots/iam-codeartifact-policy.png`.

---

### Step 2.3 — Configure your build/dev environment to use CodeArtifact

**Option A — Maven example**:

1. Generate an authorization token (or use CLI helper):

```bash
export CODEARTIFACT_AUTH_TOKEN=$(aws codeartifact get-authorization-token \
  --domain CODEARTIFACT_DOMAIN \
  --domain-owner $ACCOUNT_ID \
  --query authorizationToken --output text --region $AWS_REGION)
```

2. Use `aws codeartifact login` for Maven (this updates `~/.m2/settings.xml` automatically):

```bash
aws codeartifact login --tool maven --domain CODEARTIFACT_DOMAIN --domain-owner $ACCOUNT_ID \
  --repository CODEARTIFACT_REPO --region $AWS_REGION
```

3. Example `settings.xml`:

```xml
<settings>
<servers>
  <server>
    <id>fred-devops-cicd</id>
    <username>aws</username>
    <password>${env.CODEARTIFACT_AUTH_TOKEN}</password>
  </server>
</servers>
<profiles>
  <profile>
    <id>fred-devops-cicd</id>
    <activation>
      <activeByDefault>true</activeByDefault>
    </activation>
    <repositories>
      <repository>
        <id>fred-devops-cicd</id>
        <url>https://fred-930579048377.d.codeartifact.us-west-2.amazonaws.com/maven/devops-cicd/</url>
      </repository>
    </repositories>
  </profile>
</profiles>
<mirrors>
  <mirror>
    <id>fred-devops-cicd</id>
    <name>fred-devops-cicd</name>
    <url>https://fred-930579048377.d.codeartifact.us-west-2.amazonaws.com/maven/devops-cicd/</url>
    <mirrorOf>*</mirrorOf>
  </mirror>
</mirrors>
</settings>
```


![Image Alt](https://github.com/fredcodess/AWS-Projects/blob/main/images/cicd-mvn-build-success.png?raw=true)

---

# 3) Create & configure CodeBuild, connect to GitHub, add `buildspec.yml`

**Goal:** build your project and create deployable artifacts (upload to S3 or pass directly to pipeline).

### Step 3.1 — Create CodeBuild service role

1. Console → IAM → Create Role → select **CodeBuild** service.
2. Trust policy will be auto-configured for `codebuild.amazonaws.com`.
3. Attach S3 access (to upload artifacts) and CodeArtifact access (from Step 2). Example attach `AmazonS3FullAccess` for demo (use least privilege in prod).

**Trust policy example** (auto created):

```json
{
  "Version": "2012-10-17",
  "Statement": [
    { "Effect": "Allow", "Principal": { "Service": "codebuild.amazonaws.com" }, "Action": "sts:AssumeRole"}
  ]
}
```

---

### Step 3.2 — Create CodeBuild project and connect GitHub

Console → CodeBuild → Create project:

* **Project name**: `CODEBUILD_PROJECT`
* **Source provider**: `GitHub App / GitHub Enterprise` — use OAuth/token.
* **Repository**: pick your repo and branch.
* **Environment**: Managed image.
* **Service role**: choose the role from Step 3.1.
* **Artifacts**: choose S3 bucket `S3_BUCKET_NAME` (or let pipeline handle artifacts).

---

### Step 3.3 — `buildspec.yml` sample

`buildspec.yml` at repository root — CodeBuild will use it.

```yaml
version: 0.2

phases:
  install:
    runtime-versions:
      java: corretto8
  pre_build:
    commands:
      - echo Initializing environment
      - export CODEARTIFACT_AUTH_TOKEN=`aws codeartifact get-authorization-token --domain fred --domain-owner ACCOUNT_ID --region us-west-2 --query authorizationToken --output text`

  build:
    commands:
      - echo Build started on `date`
      - mvn -s settings.xml compile
  post_build:
    commands:
      - echo Build completed on `date`
      - mvn -s settings.xml package
artifacts:
  files:
    - target/devops-web-project.war
    - appspec.yml
    - scripts/**/*
  discard-paths: no

```

![Image Alt](https://github.com/fredcodess/AWS-Projects/blob/main/images/cicd-codebuild-success.png?raw=true)

---

# 4) Launch a deployment environment with CloudFormation and deploy with CodeDeploy

**Goal:** stand up a web server (EC2) using CloudFormation, use CodeDeploy to deploy artifacts, and see the site live.

### Step 4.1 — CloudFormation template basics

Create `cloudformation-template.yml` (skeleton example below). Use **CAPABILITY\_NAMED\_IAM** when creating stack if creating IAM roles.

**Minimal CFN skeleton (EC2 + Instance Profile + SecurityGroup):**

```yaml
AWSTemplateFormatVersion: 2010-09-09
Parameters:
  AmazonLinuxAMIID:
    Type: AWS::SSM::Parameter::Value<AWS::EC2::Image::Id>
    Default: /aws/service/ami-amazon-linux-latest/amzn2-ami-hvm-x86_64-gp2
  MyIP:
    Type: String
    Description: My IP address e.g. 1.2.3.4/32 for Security Group HTTP access rule. Get your IP from http://checkip.amazonaws.com/.
    AllowedPattern: (\d{1,3})\.(\d{1,3})\.(\d{1,3})\.(\d{1,3})/32
    ConstraintDescription: must be a valid IP address of the form x.x.x.x/32

Resources:
  VPC:
    Type: AWS::EC2::VPC
    Properties:
      CidrBlock: 10.11.0.0/16
      EnableDnsHostnames: true
      EnableDnsSupport: true
      Tags:
        - Key: 'Name'
          Value: !Join ['', [!Ref 'AWS::StackName', '::VPC'] ]

  InternetGateway:
    Type: AWS::EC2::InternetGateway
    Properties:
      Tags:
        - Key: 'Name'
          Value: !Join ['', [!Ref 'AWS::StackName', '::InternetGateway'] ]

  VPCGatewayAttachment:
    Type: AWS::EC2::VPCGatewayAttachment
    Properties:
      VpcId: !Ref VPC
      InternetGatewayId: !Ref InternetGateway

  PublicSubnetA:
    Type: AWS::EC2::Subnet
    Properties:
      AvailabilityZone: !Select 
        - 0
        - Fn::GetAZs: !Ref 'AWS::Region'
      VpcId: !Ref VPC
      CidrBlock: 10.11.0.0/20
      MapPublicIpOnLaunch: true
      Tags:
        - Key: 'Name'
          Value: !Join ['', [!Ref 'AWS::StackName', '::PublicSubnetA'] ]

  PublicRouteTable:
    Type: AWS::EC2::RouteTable
    Properties:
      VpcId: !Ref VPC
      Tags:
        - Key: 'Name'
          Value: !Join ['', [!Ref 'AWS::StackName', '::PublicRouteTable'] ]

  PublicInternetRoute:
    Type: AWS::EC2::Route
    DependsOn: VPCGatewayAttachment
    Properties:
      DestinationCidrBlock: 0.0.0.0/0
      GatewayId: !Ref InternetGateway
      RouteTableId: !Ref PublicRouteTable

  PublicSubnetARouteTableAssociation:
    Type: AWS::EC2::SubnetRouteTableAssociation
    Properties:
      RouteTableId: !Ref PublicRouteTable
      SubnetId: !Ref PublicSubnetA

  PublicSecurityGroup:
    Type: AWS::EC2::SecurityGroup
    Properties:
      VpcId:
        Ref: VPC
      GroupDescription: Access to our Web server
      SecurityGroupIngress:
      - Description: Enable HTTP access via port 80 IPv4
        IpProtocol: tcp
        FromPort: '80'
        ToPort: '80'
        CidrIp: !Ref MyIP
      SecurityGroupEgress:
      - Description: Allow all traffic egress
        IpProtocol: -1
        CidrIp: 0.0.0.0/0
      Tags:
        - Key: 'Name'
          Value: !Join ['', [!Ref 'AWS::StackName', '::PublicSecurityGroup'] ]

  ServerRole: 
    Type: AWS::IAM::Role
    Properties: 
      AssumeRolePolicyDocument: 
        Version: "2012-10-17"
        Statement: 
          - 
            Effect: "Allow"
            Principal: 
              Service: 
                - "ec2.amazonaws.com"
            Action: 
              - "sts:AssumeRole"
      Path: "/"
      ManagedPolicyArns:
        - "arn:aws:iam::aws:policy/AmazonSSMManagedInstanceCore"
        - "arn:aws:iam::aws:policy/AmazonS3ReadOnlyAccess"

  DeployRoleProfile: 
    Type: AWS::IAM::InstanceProfile
    Properties: 
      Path: "/"
      Roles: 
        - 
          Ref: ServerRole

  WebServer:
    Type: AWS::EC2::Instance
    Properties:
      ImageId: !Ref AmazonLinuxAMIID
      InstanceType: t2.micro
      IamInstanceProfile: !Ref DeployRoleProfile
      NetworkInterfaces: 
        - AssociatePublicIpAddress: true
          DeviceIndex: 0
          GroupSet: 
            - Ref: PublicSecurityGroup
          SubnetId: 
            Ref: PublicSubnetA
      Tags:
        - Key: 'Name'
          Value: !Join ['', [!Ref 'AWS::StackName', '::WebServer'] ]
        - Key: 'role'
          Value: 'webserver'

Outputs:
  URL:
    Value:
      Fn::Join:
      - ''
      - - http://
        - Fn::GetAtt:
          - WebServer
          - PublicIp
    Description: CICD web server
```

**Deploy the stack**

---

### Step 4.2 — Prepare CodeDeploy artifacts: `appspec.yml` & scripts

`appspec.yml` (place at project root):

```yaml
version: 0.0
os: linux
files:
  - source: /target/devops-web-project.war
    destination: /usr/share/tomcat/webapps/
hooks:
  BeforeInstall:
    - location: scripts/install_dependencies.sh
      timeout: 300
      runas: root
  ApplicationStart:
    - location: scripts/start_server.sh
      timeout: 300
      runas: root
  ApplicationStop:
    - location: scripts/stop_server.sh
      timeout: 300
      runas: root
```

`scripts/start_server.sh` sample:

```bash
#!/bin/bash
sudo systemctl start tomcat.service
sudo systemctl enable tomcat.service
sudo systemctl start httpd.service
sudo systemctl enable httpd.service
```

Make scripts executable:

```bash
chmod +x scripts/*.sh
```
---

### Step 4.3 — Create CodeDeploy application & deployment group

1. Console → CodeDeploy → Create application

   * Compute platform: **EC2/On-premises**
2. Create deployment group:

   * Select instances by **tag** or **Auto Scaling group** (match the EC2 instance created by CloudFormation).
   * Service role: create/choose IAM role for CodeDeploy (trust `codedeploy.amazonaws.com`) and attach policies to allow S3 reads and EC2/CodeDeploy actions.

---

### Step 4.4 — Create a deployment (upload artifact to S3)

1. Upload the artifact (zip containing `target/myapp.war`, `appspec.yml`, `scripts/`) to `s3://S3_BUCKET_NAME/<path>/myapp.zip`.
2. Console → CodeDeploy → Create deployment:

   * Choose application, deployment group.
   * Specify the S3 artifact location.

Or deploy via CLI:

```bash
aws deploy create-deployment \
  --application-name MyApp \
  --deployment-group-name MyDeploymentGroup \
  --s3-location bucket=S3_BUCKET_NAME,key=artifacts/myapp.zip,bundleType=zip
```

![Image Alt](https://github.com/fredcodess/AWS-Projects/blob/main/images/cicd-codedeploy-ec2-stack-events.png?raw=true)

---

# 5) OPTIONAL — Automate the whole system with CloudFormation (recommended)

**Goal:** write a CFN template to create CodeArtifact, CodeBuild, CodeDeploy, IAM roles, S3 buckets, EC2, and a CodePipeline stack automatically.

### Step 5.1 — Decide structure

* Use **nested stacks** (one stack per major piece: IAM, CodeArtifact, Build, Deploy, Infra) for readability.
* Keep IAM in a separate stack so you can create roles and reference ARNs.

### Step 5.2 — Template pieces to include

* `AWS::CodeArtifact::Domain` + `AWS::CodeArtifact::Repository` (resource names available in CFN).
* `AWS::S3::Bucket` for artifact store with lifecycle policy.
* `AWS::CodeBuild::Project` resources (pointing to your GitHub via CodeStar connection ARN).
* `AWS::CodeDeploy::Application` and `AWS::CodeDeploy::DeploymentGroup`.
* `AWS::CodePipeline::Pipeline` (optionally) to wire source→build→deploy.
* EC2 + Instance Profile + UserData to install CodeDeploy agent.

**Note:** CodeStar connection resources (GitHub) require you to create the connection in the console and then reference the connection ARN in CloudFormation.

### Step 5.3 — Create stack

```bash
aws cloudformation deploy --template-file cloudformation-template.yml --stack-name FULL_STACK_NAME --capabilities CAPABILITY_NAMED_IAM
```

> **Optional flag:** Automating everything is powerful but more complex. If you're new, implement Steps 1–4 manually first, then codify in CloudFormation.

---

# 6) Create a complete CI/CD pipeline with AWS CodePipeline

**Goal:** automatically build and deploy on pushes to GitHub.

### Step 6.1 — Create CodePipeline service role

1. IAM → Create role → choose **CodePipeline** service.
2. Attach policies to allow CodeBuild, CodeDeploy, S3 access. Example trust:

```json
{
  "Version": "2012-10-17",
  "Statement": [
    { "Effect": "Allow", "Principal": { "Service": "codepipeline.amazonaws.com" }, "Action": "sts:AssumeRole" }
  ]
}
```


---

### Step 6.2 — Create a CodeStar connection to GitHub

Console → Developer Tools → CodeStar connections → Create connection → follow onscreen steps to authorize GitHub.

---

### Step 6.3 — Create the pipeline

Console → CodePipeline → Create pipeline:

* **Pipeline name**: `CODEPIPELINE_NAME`
* **Artifact store**: Use `S3_BUCKET_NAME`
* **Add stage: Source**

  * Provider: GitHub (CodeStar Connection)
  * Repo & branch: choose your repo and `main`
* **Add stage: Build**

  * Provider: CodeBuild
  * Project: `CODEBUILD_PROJECT` from earlier
* **Add stage: Deploy**

  * Provider: CodeDeploy
  * Application & deployment group: choose the ones from Step 4

![Image Alt](https://github.com/fredcodess/AWS-Projects/blob/main/images/cicd-codepipeline-create-source-provider.png?raw=true)
![Image Alt](https://github.com/fredcodess/AWS-Projects/blob/main/images/cicd-codepipeline-create.png?raw=true)

---

### Step 6.4 — Test automatic deployments

1. Commit a small change to the GitHub branch (e.g., update README or change `index.jsp`).

```bash
git add .
git commit -m "CI/CD test"
git push origin main
```

2. Watch CodePipeline dashboard → it should pick up the change and run Source → Build → Deploy.

![Image Alt](https://github.com/fredcodess/AWS-Projects/blob/main/images/cicd-codepipeline-success-commit.png?raw=true)

---

# Common troubleshooting & checks

* **CodeBuild fails to access CodeArtifact**: ensure CodeBuild role has `codeartifact:GetAuthorizationToken` and `codeartifact:GetRepositoryEndpoint`. Make sure `aws codeartifact login` is run in your `install` phase.
* **CodeDeploy deployment stuck at `BeforeInstall`**: check instance logs `/var/log/aws/codedeploy-agent/codedeploy-agent.log`.
* **EC2 health / security group**: ensure security group allows HTTP/HTTPS and the instance has a public IP (or ALB/NAT as needed).
* **IAM permission errors**: examine CloudTrail or the service logs; add CloudWatch Logs to CodeBuild for full logs.
* **Artifact not found in S3**: verify `buildspec.yml` uploaded artifact path and CodeDeploy is referencing the same S3 key.


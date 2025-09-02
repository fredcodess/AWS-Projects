# Cloud Monitoring & Auto-Healing

Using a simple Amazon Linux EC2 instance, we simulate high CPU usage, monitor it using **Amazon CloudWatch**, send email notifications through **Amazon SNS**, and take automated healing actions via **AWS Lambda**.

---

## 🎯 Project Goals

- ✅ Deploy a monitored EC2 Linux instance
- ✅ Configure CloudWatch to track CPU usage
- ✅ Create alarms and trigger alerts via SNS
- ✅ Restart the instance using Lambda (auto-healing)

---

## Use Case Scenario

Imagine running a production EC2 instance that suddenly spikes in CPU usage. This system helps:

- Detect unusual resource usage
- Alert operations via email instantly
- Automatically take recovery actions (reboot/start instance)
- Minimize downtime and manual intervention

---

## Architecture Overview

![Image Alt](https://github.com/fredcodess/AWS-Projects/blob/main/images/moni-archi.png?raw=true)


---

## Services Used

| Service       | Purpose                                  |
|---------------|------------------------------------------|
| EC2           | Linux server for CPU simulation          |
| CloudWatch    | Monitor CPU and trigger alarms           |
| SNS           | Alerting via email                       |
| Lambda        | Auto-healing by restarting EC2 instance  |

---

## 🛠️ Setup Instructions

### 1️⃣ Launch EC2 Instance

- AMI: **Amazon Linux 2023**
- Instance type: e.g. `t2.micro`
- Inbound Rules: Allow SSH (`22`) and ICMP (optional)
- Key Pair: Create or use existing
- Tag it: e.g. `Name = monitoring`

---

### 2️⃣ Install `stress` on EC2

SSH into the instance:

```bash
sudo yum install -y stress
````

To simulate CPU stress:

```bash
stress --cpu 2 --timeout 300
```

![Image Alt](https://github.com/fredcodess/AWS-Projects/blob/main/images/moni-linux-test.png?raw=true)
This creates a 3-minute CPU spike, triggering the CloudWatch alarm.

---

### 3️⃣ Create SNS Topic & Email Subscription

Create SNS Topic:

```bash
aws sns create-topic --name ec2-cpu-alert
```

Subscribe via email:

```bash
aws sns subscribe \
  --topic-arn arn:aws:sns:<region>:<account-id>:ec2-cpu-alert \
  --protocol email \
  --notification-endpoint your-email@example.com
```

✅ **Confirm the email** via the link sent to your inbox.

![Image Alt](https://github.com/fredcodess/AWS-Projects/blob/main/images/moni-email-confirm.png?raw=true)

---

### 4️⃣ Set Up CloudWatch Alarm

1. Go to **CloudWatch > Alarms > Create Alarm**
2. Metric: `CPUUtilization` for the EC2 instance
3. Conditions:

   * Threshold: Greater than **10%**
   * Evaluation: 1 out of 1 datapoint (1-minute period)
4. Actions:

   * Choose SNS topic `ec2-cpu-alert`
5. Name: `EC2HighCPUAlarm`

Once stress is applied, alarm should enter `ALARM` state within 60–90 seconds.

![Image Alt](https://github.com/fredcodess/AWS-Projects/blob/main/images/moni-cwatch-config.png?raw=true)

---

### 5️⃣ Email Alert Trigger

After the threshold is crossed, an email alert is sent via SNS.

![Image Alt](https://github.com/fredcodess/AWS-Projects/blob/main/images/moni-email-alert.png?raw=true)

---

### 6️⃣ Auto-Healing with Lambda

You can create a Lambda function that automatically reboots or restarts the EC2 instance when triggered by the alarm.

**Lambda Python Example:**

```python
import boto3

def lambda_handler(event, context):
    ec2 = boto3.client('ec2')
    instance_id = '...'  

    print(f'Restarting instance {instance_id}')
    ec2.reboot_instances(InstanceIds=[instance_id])
    
    return {
        'statusCode': 200,
        'body': f'Instance {instance_id} rebooted.'
    }

```

Grant the function `AmazonEC2FullAccess` or minimal `RebootInstances` IAM permission.

![Image Alt](https://github.com/fredcodess/AWS-Projects/blob/main/images/moni-lambda-reboot.png?raw=true)



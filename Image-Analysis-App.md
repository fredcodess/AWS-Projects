# Image Upload & Analysis App

The Image Upload & Analysis App is a serverless application built using AWS services to demonstrate an event-driven architecture with machine learning capabilities. Users can upload images to an Amazon S3 bucket, which triggers an AWS Lambda function to analyze the image using Amazon Rekognition. The analysis results are published to an SNS topic, delivering a notification via email with the detected labels.

## Objectives
- Automates image analysis using AWS Rekognition.
- Demonstrates event-driven architecture with S3 event triggers and Lambda.
- Publishes results to SNS for user notifications.
- Provides a practical example of integrating AWS services (S3, Lambda, Rekognition, SNS) for real-world applications.

## AWS Services Used
- **Amazon S3**: Stores uploaded images.
- **AWS Lambda**: Processes images and calls Rekognition.
- **Amazon Rekognition**: Analyzes images to detect labels.
- **Amazon SNS**: Sends analysis results via email.
- **IAM**: Manages permissions for secure access.

## Setup Instructions
1. **Create an S3 Bucket**
   - Navigate to the S3 Console and create a bucket named `image-upload-analysis-app`.
   - Keep default settings (uncheck block public access).

![Image Alt](https://github.com/fredcodess/AWS-Projects/blob/main/images/image-upload-s3.png?raw=true)

2. **Create an SNS Topic**
   - Go to the SNS Console and create a Standard topic named `image-analysis-results`.
   - Subscribe an email address to the topic and confirm the subscription via the email received.
   - Note the SNS Topic ARN (e.g., `arn:aws:sns:us-east-1:123456789012:image-analysis-results`).

![Image Alt](https://github.com/fredcodess/AWS-Projects/blob/main/images/image-upload-email-confirmation.png?raw=true)

3. **Create a Lambda Function**
   - In the Lambda Console, create a function named `ImageRekognitionProcessor` with Python 3.12 runtime.
   - Add the environment variable `SNS_TOPIC_ARN` with the value of your SNS Topic ARN.

![Image Alt](https://github.com/fredcodess/AWS-Projects/blob/main/images/image-upload-env-variable.png?raw=true)

4. **Configure IAM Permissions**
   - Attach an IAM role to the Lambda function with permissions for:
     - `rekognition:DetectLabels`
     - `s3:GetObject` for the S3 bucket
     - `sns:Publish` for the SNS topic
   - Example IAM policy:
     ```json
     {
       "Version": "2012-10-17",
       "Statement": [
         {
           "Action": ["rekognition:DetectLabels"],
           "Effect": "Allow",
           "Resource": "*"
         },
         {
           "Action": ["s3:GetObject"],
           "Effect": "Allow",
           "Resource": "arn:aws:s3:::image-upload-analysis-app/*"
         },
         {
           "Action": ["sns:Publish"],
           "Effect": "Allow",
           "Resource": "arn:aws:sns:us-east-1:123456789012:image-analysis-results"
         }
       ]
     }
     ```

![Image Alt](https://github.com/fredcodess/AWS-Projects/blob/main/images/image-upload-role-policy.png?raw=true)

5. **Set Up S3 Event Notification**
   - In the S3 bucket’s Properties tab, create an event notification for `PUT` events to trigger the `ImageRekognitionProcessor` Lambda function.

![Image Alt](https://github.com/fredcodess/AWS-Projects/blob/main/images/image-upload-23-event.png?raw=true)

6. **Deploy Lambda Code**

![Image Alt](https://github.com/fredcodess/AWS-Projects/blob/main/images/imaage-upload-deploy.png?raw=true)

## Usage
1. Upload an image (e.g., `.jpg`, `.png`) to the `image-upload-analysis-app` S3 bucket.
2. The Lambda function will process the image and call Rekognition to detect labels.
3. Check your email for the analysis results sent via SNS.

![Image Alt](https://github.com/fredcodess/AWS-Projects/blob/main/images/image-upload-result.png?raw=true)

## Testing
- **Test Case**: Upload a sample image (e.g., a photo of a dog) to the S3 bucket.
- **Expected Outcome**: Receive an email with labels like "Dog, Animal, Pet" within a few seconds.
- **Verification**: Check CloudWatch Logs for the Lambda function to confirm successful execution.

![Image Alt](https://github.com/fredcodess/AWS-Projects/blob/main/images/image-upload-cloudwatch-logs.png?raw=true)


## Troubleshooting
- **SNS Error (TopicArn None)**: Verify the `SNS_TOPIC_ARN` environment variable in Lambda.
- **No Email Received**: Confirm the SNS subscription and check the spam/junk folder.
- **Lambda Failure**: Check CloudWatch Logs for errors.


# Deploying a Static Website
![Image Alt](https://github.com/fredcodess/AWS-Projects/blob/main/images/static_host_arch.png?raw=true)

## Introduction
This guide outlines the process of deploying a static website using Amazon Web Services (AWS). The deployment leverages AWS S3 for hosting the static website, Route 53 for domain registration and DNS management, AWS Certificate Manager for securing the site with HTTPS, and CloudFront for content delivery and performance optimisation. This setup ensures a scalable, secure, and globally accessible website with minimal maintenance.

The goal is to provide a step-by-step walkthrough for beginners and intermediate users to deploy a personal website (e.g., `fredcodes.click`) with a professional setup. Screenshots of the AWS Management Console are included to clarify key steps.

## Objectives
- **Host a static website**: Use AWS S3 to store and serve static website files (HTML, CSS, JavaScript, images).
- **Register a custom domain**: Purchase and configure a domain using Route 53 for a professional web address.
- **Secure the website**: Obtain and apply an SSL/TLS certificate using AWS Certificate Manager for HTTPS.
- **Optimise performance**: Use CloudFront to distribute content globally with low latency.

## Prerequisites
Before starting, ensure you have:
- An AWS account (sign up at [aws.amazon.com](https://aws.amazon.com)).
- Static website files (e.g., `index.html`, `styles.css`, images) ready for upload.
- Basic familiarity with AWS Management Console navigation.
- A credit card for purchasing a domain via Route 53.

## Services
- **Amazon S3 (Simple Storage Service)**: A scalable object storage service used to store and serve static website files. It’s cost-effective and reliable for hosting HTML, CSS, JavaScript, and media files.
- **Route 53**: AWS’s Domain Name System (DNS) service, used to register domains and manage DNS records to route traffic to your website.
- **AWS Certificate Manager (ACM)**: A service that provides free SSL/TLS certificates to enable HTTPS, ensuring secure communication between users and your website.
- **CloudFront**: A content delivery network (CDN) that caches your website’s content at edge locations worldwide, reducing latency and improving load times.

## Step-by-Step Deployment

### Step 1: Set Up an S3 Bucket for Static Website Hosting
1. **Log in to the AWS Management Console** and navigate to the S3 service.
2. **Create a bucket**:
   - Click “Create bucket.”
   - Set the bucket name to match your domain (e.g., `domain.com`).
   - Choose a region close to you.
   - Uncheck “Block all public access” to allow public access for website hosting.
   - Acknowledge that the bucket will be public.
3. **Enable static website hosting**:
   - Go to the bucket’s “Properties” tab.
   - Scroll to “Static website hosting” and click “Edit.”
   - Enable static website hosting and specify `index.html` as the index document.
   - Note the bucket’s website endpoint (e.g., `http://domain.com.s3-website-us-east-1.amazonaws.com`).
4. **Set bucket permissions**:
   - Go to the “Permissions” tab and edit the bucket policy.
   - Add the following policy to allow public read access:
     ```json
     {
       "Version": "2012-10-17",
       "Statement": [
         {
           "Sid": "PublicReadGetObject",
           "Effect": "Allow",
           "Principal": "*",
           "Action": "s3:GetObject",
           "Resource": "arn:aws:s3:::domain.com/*"
         }
       ]
     }
     ```
5. **Upload website files**:
   - Go to the “Objects” tab and click “Upload.”
   - Upload your `index.html`, `styles.css`, images, and other files (all files from your build folder).
   - Ensure files are publicly readable (set permissions during upload if needed).

![S3 Static Website Hosting](https://github.com/fredcodess/AWS-Projects/blob/main/images/s3-static-website-hosting.png?raw=true)

### Step 2: Register a Domain with Route 53
1. **Navigate to Route 53** in the AWS Management Console.
2. **Register a domain**:
   - Go to “Domains” > “Register Domain.”
   - Search for an available domain (e.g., `domain.com`).
   - Add the domain to your cart, provide contact details, and complete the purchase (domain registration may take a few hours to propagate).
3. **Create a hosted zone**:
   - Route 53 automatically creates a hosted zone for your domain.
   - Note the hosted zone’s name servers (NS records) for later use.

![Route 53 Domain](https://github.com/user-attachments/assets/50b61d29-7c9d-40db-a2fd-a876994617be)

### Step 3: Request an SSL Certificate with AWS Certificate Manager
1. **Navigate to ACM** in the AWS Management Console (ensure you’re in `us-east-1` for CloudFront compatibility).
2. **Request a certificate**:
   - Click “Request a certificate” and choose “Request a public certificate.”
   - Add your domain name (e.g., `domain.com` and `*.domain.com` for wildcard subdomains).
   - Select “DNS validation” for verification.
3. **Validate the certificate**:
   - ACM provides a CNAME record to add to your Route 53 hosted zone.
   - In Route 53, go to your domain’s hosted zone, click “Create record,” and add the CNAME record provided by ACM.
   - Validation typically completes within a few minutes.
4. **Note the certificate ARN**: Once validated, copy the certificate’s ARN for use in CloudFront.

![ACM Certificate](https://github.com/fredcodess/AWS-Projects/blob/main/images/acm-certificate.png?raw=true)

### Step 4: Set Up CloudFront for Content Delivery
1. **Navigate to CloudFront** in the AWS Management Console.
2. **Create a distribution**:
   - Click “Create Distribution” and select “Web” as the delivery method.
   - Set the “Origin domain” to your S3 bucket’s website endpoint (e.g., `yourdomain.com.s3-website-us-east-1.amazonaws.com`).
   - Under “Viewer Protocol Policy,” select “Redirect HTTP to HTTPS.”
   - Add your domain (e.g., `domain.com`) under “Alternate domain names (CNAMEs).”
   - Select the ACM certificate created in Step 3.
   - Set “Default root object” to `index.html`.
   - Leave other settings as default unless specific needs arise.
3. **Wait for deployment**: CloudFront distributions take 5–20 minutes to deploy. Note the distribution’s domain name (e.g., `d1234567890.cloudfront.net`).

![CloudFront Distribution](https://github.com/fredcodess/AWS-Projects/blob/main/images/cloudfront-distribution.png?raw=true)

### Step 5: Configure Route 53 to Route Traffic to CloudFront
1. **Go to Route 53** and open your domain’s hosted zone.
2. **Create an A record**:
   - Click “Create record.”
   - Set the record type to “A – IPv4 address.”
   - Enable “Alias” and select “Alias to CloudFront distribution.”
   - Choose your CloudFront distribution from the dropdown.
   - Set the record name to your domain (e.g., `domain.com`).
3. **Create a www record (optional)**:
   - Repeat the above steps to create an A record for `www.domain.com` aliased to the same CloudFront distribution.
4. **Test the domain**: After DNS propagation (up to 48 hours, typically faster), visit `https://domain.com` to verify the website loads securely.

![Route 53 A Record](https://github.com/fredcodess/AWS-Projects/blob/main/images/route53-a-record.png?raw=true)

## Testing and Validation
- Open a browser and visit `https://domain.com` to ensure the website loads correctly.
- Verify HTTPS is active (look for the padlock icon in the browser).
- Test navigation to ensure all pages and assets (images, CSS, JavaScript) load properly.
- If using `www.domain.com`, test that it works or redirects appropriately.

## Troubleshooting
- **S3 access issues**: Ensure the bucket policy allows public read access and files are uploaded correctly.
- **Certificate errors**: Confirm the ACM certificate is issued in `us-east-1` and validated via DNS.
- **CloudFront errors**: Check that the origin domain is the S3 website endpoint, not the REST API endpoint.
- **DNS issues**: Verify Route 53 records are correct and DNS has propagated (use `dig` or `nslookup`).


## Conclusion
By following these steps, you'd be able to deployed a secure, scalable, and high-performance static website using AWS. The combination of S3, Route 53, ACM, and CloudFront ensures your site is accessible globally, secured with HTTPS, and optimised for speed. Update your website by uploading new files to the S3 bucket, and changes will propagate through CloudFront automatically.

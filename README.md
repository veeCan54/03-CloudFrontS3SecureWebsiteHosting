# CloudFront S3 Secure Website Hosting
# Objectives:
1. CDNs are not cheap. Explore primary design considerations to determine if a CDN is needed for the system we are designing. 
2. Explore CloudFront features.
2. Host a secure static website on S3. Explore the default certificate provided by CloudFront.
3. Provision a custom SSL certificate using ACM and use it with a custom domain name integrating with Route 53.
3. Use Origin Access Control to restrict access to the S3 bucket so that only CloudFront can access it.

# Important considerations for deciding if a content delivery network is needed for the system we are designing: 
1. Does the system have user presence across the globe?
2. What type of delivery is needed for the workflow?  Only dynamic? Only static, or a combination of both?
3. What are the approximate sizes of objects delivered?
4. Is any object transformation needed?
5. What are the security and access considerations for the content being served?  
<p>These are some primary design considerations, however there could be many more.  
<strong>If our requirements justify the need for a CDN, then AWS CloudFront is a possible solution.</strong>

# CloudFront Overview:  
CloudFront is a content delivery network. It's job is to improve the delivery of that content from it's original location to the viewers of that data.
It does that by using caching and by using a global delivery network of edge locations. Content is cached in these edge locations until their TTL expires or until a new origin fetch is done as a result of a new deployment or a cache invalidation.  

**Key components:**
<ol> <li> <strong>Origin</strong> is where the original, definitive version of our content is stored. It can be S3 or a custom origin like a webserver that has a publicly routable IPV2 IP address. You could have one or more origins for a CloudFront distribution, upto 25 currently.</li>
<li> <strong>Distribution</strong> is the configurable unit of CloudFront. To use CloudFront, we create a distribution and this gets deployed to the network. </li>
<li> <strong>Edge location</strong> is where our information is cached. Edge locations are part of CloudFront's global network and are located in major AWS Regions and major country and state capitals. Edge locations are closer to our end users. </li>
<li> <strong>Regional Edge Cache</strong> provides another layer of caching. They are much bigger than Edge locations and are fewer in number. They are used to cache large amounts of less frequently accessed data, in larger more global CloudFornt deployments. </li>
<li> <strong>Cache behavior settings</strong> allow us to configure CloudFront functionality based on our application needs. A distribution comes with a default cache behavior, we can create additional cache behaviors that define how CloudFront responds when it receives a request for objects that match for e.g a particular path pattern. </li> </ol>  

# Features and integrations:
<ol> <li> APIs or applications can be delivered over HTTPS using the latest version of TLS to encrypt and secure communication between the viewers and CloudFront. AWS Certificate Manager (ACM) can be used to create a custom SSL certificate and deploy to an CloudFront distribution for free. </li>
<li>Protection against network and application layer attacks is provided by CloudFront by seamlessly integrating with AWS Shield, AWS Web Application Firewall (WAF), and Amazon Route 53. This creates a flexible, layered security perimeter against multiple types of attacks including network and application layer DDoS attacks. </li>
<li>Access to content can be restricted using a number of mechanisms like Signed URLs and Signed Cookies. It is possible to restrict access to only authenticated viewers using Token Authentication. Users in certain geographic regions can be restricted access to content through CloudFront's geo-restriction capability. Securing an S3 website origin, making it only accessible from CloudFront is possible using using Origin Access Identity, now referred to as Origin Access Control. </li>
<li>CloudFront infrastructure and processes are compliant with industry standards like PCI-DSS Level 1, HIPAA, and ISO 9001 FEDRAMP etc. </li>
<li>CloudFront is a pay-per-use offering, requiring no long term commitments or minimum fees. After the free tier, cost varies depending on the amout of data transfered to the internet and to the origin per GB. </li>
<li>Lambda@Edge makes it possible to run Node.js and Python Lambda functions to customize content that CloudFront delivers. Here the functions are executed closer to the viewer, in response to CloudFront events without the need for provisioning or managing servers. </li>
</ol>  

> **Note:** The focus of this exercise is to host a secure static S3 website, configure a custom domain name and to restrict access to the content to CloudFront. Lamda@Edge will be the focus of a future hands on exercise. 

# Steps:
1. Deploy static website using the cloud formation template. [Details](#Step1) 
2. Test the S3 website. Notice that if the protocol in the url is modified to https, it will keep spinning and will not load. [Details](#Step2)   
3. <a name="Step3Back"></a>Create a new CloudFront distribution using the S3 website as Origin. [Details](#Step3)  
4. Load the distribution endpoint on a browser, notice the padlock which indicates it is using https, view the certificate. [Details](#Step4)  
5. Modify the distribution by adding a custom DNS name. For this step and subsequent steps we need a public hosted zone in Route 53. [Details](#Step5)  
6. Request a custom SSL certificate using ACM and attach it to the CloudFront distribution. [Details](#Step6)  
7. In  Route 53 create an A record pointing the custom domain name to the CloudFront distribution. [Details](#Step7)  
8. Load the custom domain name on a browser, notice the padlock and view the name on the certificate matching the custom domain name. [Details](#Step8)  
9. Modify the distribution by restricting public access. [Details](#Step9) 
10. Update bucket policy so that only the CloudFront distribution can access the bucket.  [Details](#Step10) 
11. Explore the new bucket policy and understand what it means. [Details](#Step11) 
12. Load the S3 static website URL on a browser and notice the 403 HTTP Status code. It can only be accessed by the CloudFront distribution. [Details](#Step12) 
13. Clean up resources.


# Step 1:<a name="Step1"></a>  
Deploy static S3 website using the CloudFormation template [here](https://github.com/veeCan54/03-CloudFrontS3SecureWebsiteHosting/blob/main/files/S3Website.yaml).  
[Back to Summary](#Step1Back) 
# Step 2:<a name="Step2"></a> 
After the Stack has been created, copy the static website url from CloudFormation Outputs section.  
Load it in a browser window to make sure it is working.  
![Alt text](https://github.com/veeCan54/03-CloudFrontS3SecureWebsiteHosting/blob/main/images/00StackComplete.png)
Notice that it is unsecure.
![Alt text](https://github.com/veeCan54/03-CloudFrontS3SecureWebsiteHosting/blob/main/images/01UnsecureStaticWebsite.png)
If we try https, it will not load - it will simply spin.
![Alt text](https://github.com/veeCan54/03-CloudFrontS3SecureWebsiteHosting/blob/main/images/01UnsecureStaticWebsiteWillNotLoad.png)
We are going to change this.  
[Back to Summary](#Step2Back)
# Step 3:<a name="Step3"></a>  
On the AWS Admin console, go to CloudFormation, click on Create New Distribution.  
Select the Static S3 website endpoint from the drop down as the origin.  
![Alt text](https://github.com/veeCan54/03-CloudFrontS3SecureWebsiteHosting/blob/main/images/03Distribution1.png)  
For Viewer protocol policy, select **HTTP & HTTPS**  
Restrict Viewer access -  **no**  
Caching Policy as **CacheOptimized**  
Under Security settings, select **Do not enable security protections.**  
There is no need for an SSL certificate because CloudFront will assign a default certificate. 
![Alt text](https://github.com/veeCan54/03-CloudFrontS3SecureWebsiteHosting/blob/main/images/03Distribution4.png)  
Click on Create Distribution.  
![Alt text](https://github.com/veeCan54/03-CloudFrontS3SecureWebsiteHosting/blob/main/images/03DistributionDeploying.png) 
Wait until the distribution is created and has completed deployment.  
[Back to Summary](#Step3Back)
# Step 4:<a name="Step4"></a> 
Copy the URL of the distribution.  
![Alt text](https://github.com/veeCan54/03-CloudFrontS3SecureWebsiteHosting/blob/main/images/03DistributionCreated0.png) 
Load the url on a browser and notice the padlock on the URL. Observe the default SSL certificate provided by cloudfront.
![Alt text](https://github.com/veeCan54/03-CloudFrontS3SecureWebsiteHosting/blob/main/images/03DistributionWithCert.png) 
# Step 5:<a name="Step5"></a>   
Inspect the Route 53 public hosted zone. In the public zone we only have the NS and SOA records.  
To add a custom DNS name to the distribution, edit the distribution.
![Alt text](https://github.com/veeCan54/03-CloudFrontS3SecureWebsiteHosting/blob/main/images/05UpdateDNSName.png) 

# Step 6:<a name="Step6"></a>  
Enter the chosen DNS name.  
Click on Request certificate.   
![Alt text](https://github.com/veeCan54/03-CloudFrontS3SecureWebsiteHosting/blob/main/images/05UpdateDNSName1.png) 

In the Certificate Manager screen enter the fully qualified domain name. Make sure DNS validation is selected.  
![Alt text](https://github.com/veeCan54/03-CloudFrontS3SecureWebsiteHosting/blob/main/images/05RequestCert1.png)
![Alt text](https://github.com/veeCan54/03-CloudFrontS3SecureWebsiteHosting/blob/main/images/05RequestCert2.png)
Click on Create records in Route 53
![Alt text](https://github.com/veeCan54/03-CloudFrontS3SecureWebsiteHosting/blob/main/images/05RequestCert3.png)
Route 53 will create a new CNAME record in our public hosted zone. This is how Route 53 validates that we own the domain. 
![Alt text](https://github.com/veeCan54/03-CloudFrontS3SecureWebsiteHosting/blob/main/images/05CreateR53Records.png)
Now check the Route 53 hosted zone. The new record has been created by Route 53. DNS validation has been completed successfully. 
![Alt text](https://github.com/veeCan54/03-CloudFrontS3SecureWebsiteHosting/blob/main/images/05Route53PublicZoneRecordNew.png)
Return to CloudFront to edit the distribution. Now the new SSL certificate shows up in the drop down. Select it and save the changes.  
![Alt text](https://github.com/veeCan54/03-CloudFrontS3SecureWebsiteHosting/blob/main/images/06CertShowsUp.png)
Wait for the distribution to be updated and deployed. Notice that the custom DNS name and the cert name are displayed.
![Alt text](https://github.com/veeCan54/03-CloudFrontS3SecureWebsiteHosting/blob/main/images/06DistroUpdated.png)
# Step 7:<a name="Step7"></a> 
In  Route 53 create an A record pointing the custom domain name to the CloudFront distribution. 
![Alt text](https://github.com/veeCan54/03-CloudFrontS3SecureWebsiteHosting/blob/main/images/07Route53ARecord0.png)
![Alt text](https://github.com/veeCan54/03-CloudFrontS3SecureWebsiteHosting/blob/main/images/07Route53ARecord1.png)
# Step 8:<a name="Step8"></a> 
Load the custom DNS name. Inspect the padlock and Observe the custom SSL cert we created.
![Alt text](https://github.com/veeCan54/03-CloudFrontS3SecureWebsiteHosting/blob/main/images/07SecureSiteDeployed.png)

# Step 9:<a name="Step9"></a>  
Now we need to restrict access to the S3 site so that only CloudFront can access it. This ensures security of our application.
In the distribution, click on Origin. Select the origin and click on Edit.  
Now we are going to modify the  Origin access.  
From Public, change it to Origin access control settings (recommended) 
![Alt text](https://github.com/veeCan54/03-CloudFrontS3SecureWebsiteHosting/blob/main/images/08OACSetting1.png)
Click on Create Origin Access Setting and then Create control setting.  
Keep the defaults and click on Create.
![Alt text](https://github.com/veeCan54/03-CloudFrontS3SecureWebsiteHosting/blob/main/images/08ACCreateControlSetting.png)
Now the control setting that was just created appears on the drop down.  
We need to now modify the Bucket policy on our S3 bucket so that only CloudFront can access it.  
![Alt text](https://github.com/veeCan54/03-CloudFrontS3SecureWebsiteHosting/blob/main/images/08OACSetting3.png)
# Step 10:<a name="Step10"></a>  
In S3, paste the copied bucket policy.
![Alt text](https://github.com/veeCan54/03-CloudFrontS3SecureWebsiteHosting/blob/main/images/08OACSettingNewBucketPolicy.png)

# Step 11:<a name="Step11"></a>  
Understand what the policy means.  
A GetObject operation is allowed on our bucket by the CloudFront service, as long as the source arn of the distribution matches ours.  
Save the policy.
![Alt text](https://github.com/veeCan54/03-CloudFrontS3SecureWebsiteHosting/blob/main/images/08OACSettingNewBucketPolicySaved.png)
# Step 12:<a name="Step12"></a>  
After setting up Origin Access Control, if we try our original S3 website URL we see that it can't be accessed. 
![Alt text](https://github.com/veeCan54/03-CloudFrontS3SecureWebsiteHosting/blob/main/images/09OACInEffect.png)  
It can only be accessed using CloudFront.

![Alt text](https://github.com/veeCan54/03-CloudFrontS3SecureWebsiteHosting/blob/main/images/09OACInEffect1.png)
# Step 13:<a name="Step13"></a>  
Clean up resources.  
Delete the bucket, delete the Distribution. delete the Record in Route 53. 





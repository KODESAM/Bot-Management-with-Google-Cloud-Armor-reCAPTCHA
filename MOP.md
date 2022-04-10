
# Bot-Management-with-Google-Cloud-Armor-reCAPTCHA on GCP
Before you begin
Inside Cloud Shell, make sure that your project id is set up

```
gcloud config list project
gcloud config set project [YOUR-PROJECT-NAME]
PROJECT_ID=[YOUR-PROJECT-NAME]
echo $PROJECT_ID
```

Enable APIs
Enable all necessary services
```
gcloud services enable compute.googleapis.com
gcloud services enable logging.googleapis.com
gcloud services enable monitoring.googleapis.com
gcloud services enable recaptchaenterprise.googleapis.com
```
Click Create Firewall Rule.
Set the following values, leave all other values at their defaults:
Property

Value (type value or select option as specified)

Name default-allow-health-check

Network default

Targets Specified target tags

Target tags allow-health-check

Source filter IP Ranges

Source IP ranges 130.211.0.0/22, 35.191.0.0/16

Protocols and ports Specified protocols and ports, and then check tcp. Type 80 for port number
```
cloudshell:~ (wide-retina-346520)$ gcloud compute firewall-rules create default-allow-health-check --direction=INGRESS --priority=1000 /
--network=default --action=ALLOW --rules=tcp:80 --source-ranges=130.211.0.0/22,35.191.0.0/16 --target-tags=allow-health-check
```
Similarly, create a Firewall rule to allow SSH-ing into the instances -
```
cloudshell:~ (wide-retina-346520)$ gcloud compute firewall-rules create allow-ssh --direction=INGRESS --priority=1000 /
--network=default --action=ALLOW --rules=tcp:22 --source-ranges=0.0.0.0/0 --target-tags=allow-health-check
```
# Create an instance template
```
gcloud compute instance-templates create lb-backend-template --project=wide-retina-346520 --machine-type=n1-standard-1 /
--network-interface=network-tier=PREMIUM,subnet=default /
--metadata=startup-script=\#\!\ /bin/bash$'\n'sudo\ apt-get\ update$'\n'sudo\ apt-get\ install\ apache2\ -y$'\n'sudo\ a2ensite\ default-ssl$'\n'sudo\ a2enmod\ ssl$'\n'sudo\ vm_hostname=\"\$\(curl\ -H\ \"Metadata-Flavor:Google\"\ \\$'\n'http://169.254.169.254/computeMetadata/v1/instance/name\)\"$'\n'sudo\ echo\ \"Page\ served\ from:\ \$vm_hostname\"\ \|\ \\$'\n'tee\ /var/www/html/index.html /
--maintenance-policy=MIGRATE --service-account=294619223509-compute@developer.gserviceaccount.com /
--scopes=https://www.googleapis.com/auth/devstorage.read_only,https://www.googleapis.com/auth/logging.write,https://www.googleapis.com/auth/monitoring.write,https://www.googleapis.com/auth/servicecontrol,https://www.googleapis.com/auth/service.management.readonly,https://www.googleapis.com/auth/trace.append /
--region=us-east1 --tags=allow-health-check --create-disk=auto-delete=yes,boot=yes,device-name=lb-backend-template,image=projects/debian-cloud/global/images/debian-10-buster-v20220406,mode=rw,size=10,type=pd-balanced --no-shielded-secure-boot --shielded-vtpm --shielded-integrity-monitoring --reservation-affinity=any
```

#Create the managed instance group
```
gcloud compute instance-groups managed create lb-backend--example --project=wide-retina-346520 --base-instance-name=lb-backend--example --size=1 --template=lb-backend-template --zone=us-central1-a
```
```
gcloud beta compute instance-groups managed set-autoscaling lb-backend--example --project=wide-retina-346520 --zone=us-central1-a --cool-down-period=60 --max-num-replicas=10 --min-num-replicas=1 --mode=off --target-cpu-utilization=0.6
```
# Add a named port to the instance group

For your instance group, define an HTTP service and map a port name to the relevant port. The load balancing service forwards traffic to the named port.

```
gcloud compute instance-groups set-named-ports lb-backend-example \
    --named-ports http:80 \
    --zone us-central1-a
    
```
# Configure the HTTP Load Balancer

Configure the backend
Backend services direct incoming traffic to one or more attached backends. Each backend is composed of an instance group and additional serving capacity metadata.

Click on Backend configuration.
For Backend services & backend buckets, click Create a backend service.
Set the following values, leave all other values at their defaults:
Property

Value (select option as specified)

Name http-backend

Protocol HTTP

Named Port http

Instance group lb-backend-example

Port numbers 80

Configure the frontend
The host and path rules determine how your traffic will be directed. For example, you could direct video traffic to one backend and static traffic to another backend. However, you are not configuring the Host and path rules in this lab.

Click on Frontend configuration.
Specify the following, leaving all other values at their defaults:
Property

Value (type value or select option as specified)

Protocol HTTP

IP version IPv4

IP address Ephemeral

Port 80

# Create reCAPTCHA session token and WAF challenge-page site key
Create the reCAPTCHA session token site key and enable the WAF feature for the key. We will also be setting the WAF service to Cloud Armor to enable the Cloud Armor integration.
```
gcloud recaptcha keys create --display-name=test-key-name \
   --web --allow-all-domains --integration-type=score --testing-score=0.5 \
   --waf-feature=session-token --waf-service=ca
```
Output of the above command, gives you the key created. Make a note of it as we will add it to your web site in the next step
Created [6LfKE2EfAAAAAB7P5zWGsPUYUYXG9B56qrMvgvLp].

Create the reCAPTCHA WAF challenge-page site key and enable the WAF feature for the key. You can use the reCAPTCHA challenge page feature to redirect incoming requests to reCAPTCHA Enterprise to determine whether each request is potentially fraudulent or legitimate. We will later associate this key with the Cloud Armor security policy to enable the manual challenge. We will refer to this key as CHALLENGE-PAGE-KEY in the later steps.
```
gcloud recaptcha keys create --display-name=challenge-page-key \
   --web --allow-all-domains --integration-type=INVISIBLE \
   --waf-feature=challenge-page --waf-service=ca
   
```
Createdf[6LemAGEUUUUAANSO6MBSTslb9RdfHqzlK1lStt79]

Implement reCAPTCHA session token site key
Navigate to Navigation menu ( mainmenu.png) > Compute Engine > VM Instances. Locate the VM in your instance group and SSH to it.

Go to the webserver root directory and and change user to root -
```
@lb-backend-example-4wmn:~$ cd /var/www/html/
@lb-backend-example-4wmn:/var/www/html$ sudo su
```
```
Update the landing index.html page and embed the reCAPTCHA session token site key. The session token site key is set in the head section of your landing page as below -
<script src="https://www.google.com/recaptcha/enterprise.js?render=<REPLACE_TOKEN_HERE>&waf=session" async defer></script>
```
Create three other sample pages to test out the bot management policies -
good-score.html
```
root@lb-backend-example-4wmn:/var/www/html# echo '<!DOCTYPE html><html><head><meta http-equiv="Content-Type" content="text/html; charset=windows-1252"></head><body><h1>Congrats! You have a good score!!</h1></body></html>' > good-score.html
bad-score.html
```
```
root@lb-backend-example-4wmn:/var/www/html# echo '<!DOCTYPE html><html><head><meta http-equiv="Content-Type" content="text/html; charset=windows-1252"></head><body><h1>Sorry, You have a bad score!</h1></body></html>' > bad-score.html
median-score.html
```
```
root@lb-backend-example-4wmn:/var/www/html# echo '<!DOCTYPE html><html><head><meta http-equiv="Content-Type" content="text/html; charset=windows-1252"></head><body><h1>You have a median score that we need a second verification.</h1></body></html>' > median-score.html
```
Validate that you are able to access all the webpages by opening them in your browser. Make sure to replace [LB_IP_v4] with the IPv4 address of the load balancer.
Open http://[LB_IP_v4]/index.html. You will be able to verify that the reCAPTCHA implementation is working when you see "protected by reCAPTCHA" at the bottom right corner of the page -

# Create Cloud Armor security policy rules for Bot Management

n this section, you will use Cloud Armor bot management rules to allow, deny and redirect requests based on the reCAPTCHA score. Remember that when you created the session token site key, you set a testing score of 0.5.

In Cloud Shell(refer to "Start Cloud Shell" under "Setup and Requirements" for instructions on how to use Cloud Shell), create security policy via gcloud:
```
gcloud compute security-policies create recaptcha-policy \
    --description "policy for bot management"
```
To use reCAPTCHA Enterprise manual challenge to distinguish between human and automated clients, associate the reCAPTCHA WAF challenge site key we created for manual challenge with the security policy. Replace "CHALLENGE-PAGE-KEY" with the key we created -
```
gcloud compute security-policies update recaptcha-policy \
   --recaptcha-redirect-site-key "CHALLENGE-PAGE-KEY"
```
Add a bot management rule to allow traffic if the url path matches good-score.html and has a score greater than 0.4.
```
gcloud compute security-policies rules create 2000 \
     --security-policy recaptcha-policy\
     --expression "request.path.matches('good-score.html') &&    token.recaptcha_session.score > 0.4"\
     --action allow
```
Add a bot management rule to deny traffic if the url path matches bad-score.html and has a score less than 0.6.
```
  gcloud compute security-policies rules create 3000 \
     --security-policy recaptcha-policy\
     --expression "request.path.matches('bad-score.html') && token.recaptcha_session.score < 0.6"\
     --action "deny-403"
```
Add a bot management rule to redirect traffic to Google reCAPTCHA if the url path matches median-score.html and has a score equal to 0.5
```
  gcloud compute security-policies rules create 1000 \
     --security-policy recaptcha-policy\
     --expression "request.path.matches('median-score.html') && token.recaptcha_session.score == 0.5"\
     --action redirect \
     --redirect-type google-recaptcha
```
Attach the security policy to the backend service http-backend:
```
gcloud compute backend-services update http-backend \
    --security-policy recaptcha-policy –-global
```
Validate Bot Management with Cloud Armor
Open up a browser and enter the url http://[LB_IP_v4]/index.html. Navigate to "Visit allow link"






















































































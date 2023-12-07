ESTABLISHING A LOAD BALANCER IN AN ALREADY EXISTING PROJECT
***
<h3>NOTES</h3>
/// You may have to update your permissions by creating a new role with role/compute.InstanceAdmin for authorization to create a new instance. For example

	gcloud projects add-iam-policy-binding sysadmin-418-cnwa 
    	--member=user:peachesnredvelvet@gmail.com
    	--role=roles/compute.instanceAdmin

/// You'll need to go through the Google Cloud console if Linux is having authentication issues.
/// The startup script should automatically install Apache2. If not, you will have to install Apache2 and configure it on each VM.
// You will need to use PowerShell.

***

When you're establishing a load balancer, it's important to adapt according to the OS you're using. It's also good to use the commands from the command line of your operating system to establish the same configurations for all the virtual instances that exist on the project.

1. Verify the project that you're working on using: gcloud config list project
	Your project name should come up, but you should now make a variable for the project name. For windows: FOR /F "tokens=*" %i IN ('gcloud config get-value project') DO SET PROJECT_ID%i
	Or you can use: SET PROJECT_ID=sysadmin-418-cnwa

	Test the variable by using echo %PROJECT_ID%. 
2. Verify the zone that you're working in: gcloud config list zone
	Use SET ZONE=us-central1-a

	Test the variable by using %ZONE%.
3. After verifying that you've set the variables and updated permissions, then you want to create the 1st instance: 
	gcloud compute instances create www1 --zone=%zone% --machine-type=e2-small --image-family=debian-11 --image-project=debian-cloud

	Proceed to creating the 2nd instance:
	gcloud compute instances create www2 --zone=%zone% --machine-type=e2-small --image-family=debian-11 --image-project=debian-cloud

	Proceed to creating the 3rd instance:
	gcloud compute instances create www3 --zone=%zone% --tags=www3 --machine-type=e2-small --image-family=debian-11 --image-project=debian-cloud


	I added tags separately by using: gcloud compute add-tags INSTANCE --tags=tagname

	I had to go through the google console to add metadata for each instance since Linux was experiencing authentication issues. For example:
((///WWW1///
	'#!/bin/bash
      apt-get update
      apt-get install apache2 -y
      service apache2 restart
      echo "
<h3>Web Server: www1</h3>" | tee /var/www/html/index.html'))

///WWW2/

(('#!/bin/bash
      apt-get update
      apt-get install apache2 -y
      service apache2 restart
      echo "
<h3>Web Server: www2</h3>" | tee /var/www/html/index.html'))

///WWW3///

(('#!/bin/bash
      apt-get update
      apt-get install apache2 -y
      service apache2 restart
      echo "
<h3>Web Server: www3</h3>" | tee /var/www/html/index.html'))



    I proceeded to complete this for every instance by clicking Edit > Custom Metadata > the script into the Startup Script section.
    
    4. Next, we create the firewall to allow traffic between the VM instance. Through the command prompt (due to owner privileges) we paste:
        gcloud compute firewall-rules create www1-firewall-rule --target-tags www1 --allow tcp:80
        
    5. Then I listed the instances through the command prompt and then verified that the instances are running with 'curl'
        curl http://34.42.177.9
        curl http://35.202.53.152
        curl http://34.30.219.37
            They all output default data, meaning that the instances are running with curl.
        
    6. Create a static IP:
        gcloud compute addresses create network-lb-ip-1 --region %region%
    7. Create the target pools. Target pools are used for load balancing, you may have to switch between %region% or replacing %region% with the actual region that you're in. For example, mine looked like this:
        gcloud compute target-pools add-instances www-pool-us-central1-a --instances www1,www2,www3 --region=us-central1 --zone=us-central1-a
    8. After creating the target pool, added a forwarding rule:
        gcloud compute forwarding-rules create www-rule --region us-central1 --ports 80 --address network-lb-ip-1 --target-pool www-pool-us-central1-a

    9. Press Win + X. Open Windows PowerShell (Admin), and run this command:
        $IPADDRESS = gcloud compute forwarding-rules describe www-rule --region us-central1 --format="json" | ConvertFrom-Json | Select-Object -ExpandProperty IPAddress
            Run echo $IPADDRESS to verify the address, you should have no problems here.
    10. Time to start sending traffic. Run this command in PowerShell so that HTTP requests will run indefinitely:
        while ($true) { Invoke-WebRequest -Uri "http://$IPADDRESS"; Start-Sleep -Seconds 1 }
    11. Created an instance template through PowerShell:
        gcloud compute instance-templates create lb-backend-template --region=us-central1 --network=default --subnet=default --tags=allow-health-check --machine-type=e2-medium --image-family=debian-11 --image-project=debian-cloud --metadata=startup-script='@echo off
            echo Installing Apache...
            choco install apache2 -y
            echo Configuring Apache...
            httpd -k install
            httpd -k start
            echo Getting instance name...
            set vm_hostname=%COMPUTERNAME%
            echo Page served from: %vm_hostname% > C:\var\www\html\index.html
            echo Restarting Apache...
            httpd -k restart'
    12. Created an Instance Group from the Instance Templates:
            gcloud compute instance-groups managed create lb-backend-group --template=lb-backend-template --size=2 --zone=us-central1-a
    13. Create a firewall to allow health checks:
        gcloud compute firewall-rules create fw-allow-health-check --network=default --action=allow --direction=ingress --source-ranges=130.211.0.0/22,35.191.0.0/16 --target-tags=allow-health-check --rules=tcp:80
    14. Then you need a global static IPv4, so run this command:
        gcloud compute addresses create lb-ipv4-1 --ip-version=IPV4 --global
    15. Created health check for balancer: gcloud compute health-checks create http http-basic-check --port 80
    16. Created a backend service:
        gcloud compute backend-services create web-backend-service --protocol=HTTP --port-name=http --health-checks=http-basic-check --global
    17. Made an URL map by running this command:
        gcloud compute url-maps create web-map-http --default-service web-backend-service
    18. This will create the HTTP proxy (reverse proxy) that routes IPs to my address:
        gcloud compute target-http-proxies create http-lb-proxy --url-map web-map-http
    19. Created the global forwarding rule:
        gcloud compute forwarding-rules create http-content-rule --address=lb-ipv4-1 --global --target-http-proxy=http-lb-proxy --ports=80
    ||TESTING
        To verify that my load balancers were healthy, I went to Google Console > Network Services > web-map-http > and checked the status.
        

Note: If you are doing lab in F5 UDF - NGINX Workshop Sandpit blueprint, please follow this guide instead https://github.com/mcheo/nms-demo-kit/blob/main/README_UDF.md

Note: As of this writing, only NIM is supported in ARM processor but not the other modules (ADM, ACM, SM)

## Credit
This repo leveraged the work done by fantastic [Fabrizio](https://github.com/fabriziofiorucci) at https://github.com/nginxinc/NGINX-Demos/tree/master/nginx-nms-docker

## Description
This demo kit can be used to easily step up NGINX Management Suites-NMS with various modules (API Connectivity Manager-ACM, Instance Manager-NIM, Security Monitoring-SM), NGINX Plus as load balancer and API Gateway, NGINX App Protect as API Security Protection.

For demonstration, we will build a demo setup of 1 LB fronting API gateway clusters which connect to the API endpoints

![alt text](assets/demo-setup-part-1.png)

## Prerequisite: 
- Install docker and docker-compose in your machine
- In [NGINX website](https://www.nginx.com/pricing/) request for NGINX Plus and NGINX Management Suite trial licenses


## Getting started
1. **Start NMS**
```
#Clone the repo
git clone https://github.com/mcheo/nms-demo-kit

cd nms-demo-kit

#Make the script executable
chmod 755 ./scripts/*

#Download NGINX Plus trial license and put nginx-repo.crt and nginx-repo.key in nginx-plus folder
cp nginx-repo.* nginx-plus/
chmod 755 nginx-plus/*

#Build NMS (NGINX Instance Manager, API Connectivity Manager) container image
#Example build with the latest release
./scripts/buildNMS.sh -t nginx-nms -i -C nginx-plus/nginx-repo.crt -K nginx-plus/nginx-repo.key -A -W

#Or build with specific version
#You may edit Dockerfile to specify the .deb package version for the required modules
#./scripts/buildNMS.sh -t nginx-nms:2.6 -C nginx-plus/nginx-repo.crt -K nginx-plus/nginx-repo.key -A -W

#To deploy NMS container
#in MacOS might encounter port 5000 being used by AirPlay sharing process. 
#You may change the port to other than 5000 in docker-compose.yaml eg: - "6000:5000"

docker-compose -f docker-compose.yaml up -d
```

2. **Access NMS GUI**

Use browser to visit https://localhost, login with admin/admin credentials or whatever you specify in the docker-compose.yaml
![alt text](assets/nms_prelogin_landing_page.png)
![alt text](assets/nms_login_prompt.png)


Click on Settings and upload the NMS trial license.
![alt text](assets/nms-license.png)

Click on the browser Refresh button for the page to display availalble modules.

3. **Start NGINX Plus**
```
#Build NGINX Plus image with nginx-agent
#Specify NMS IP address (your laptop IP address), DO NOT use localhost or 127.0.0.1
./scripts/buildNPlusWithAgent.sh -t npluswithagent -n https://192.168.1.3

#Build NGINX Plus (ACM Dev-Portal) image with nginx-agent
#Specify NMS IP address (your laptop IP address), DO NOT use localhost or 127.0.0.1
./scripts/buildNPlusWithAgent.sh -t npluswithagent:devportal -D -n https://192.168.1.3

#Uncomment nginx-lb, nginx-gw, httpbin-app, acm.nginx-devportal section in docker-compose.yaml section
docker-compose -f docker-compose.yaml up -d
```
You should have these number of containers running
![alt text](assets/running-containers-new.png)

For the first time, click browser refresh. On NMS Instance Manager dashboard, you should see these instances
![alt text](assets/nim-managed-instances.png)


4. **Configure Load Balancer using NIM**

Enable NGINX API for NGINX LB

- In Instance Manager section, Instances tab, click on nginx-lb and Edit Config, add a new file as /etc/nginx/conf.d/nplusapi.conf  You may copy the content from repo misc/nplusapi.conf
![alt text](assets/edit-nginx-plus-api-conf.png)

- Replace the existing default.conf config with misc/lb.conf config
![alt text](assets/edit-nginx-plus-default-conf.png)

- Click Publish

Test the LB setting
```
curl -I http://localhost/
curl -I http://localhost/
```
We have configured NGINX load balancer to return backend server's IP address. There are multiple backend server (NGINX Gateway), the NGINX load balancer will round robin to different backend server and return respective IP addresses. Your containers IP addresses might be different from what is shown below.

![alt text](assets/lb-curl-testing.png)


5. **Configure API Gateway using ACM**

Tips: For step 5, you may choose to use the script to automate ACM configuration. However, you are encouraged to manually execute the steps below to be familiar with the ACM workflow.
```
#sh misc/end2end_deploy.sh
#sh misc/end2end_delete.sh
```

Enable API Gateway Cluster

- In API Connectivity Manager, Infrastructure, create a Workspace, an Environment. In the Environment tab, add API Gateway Cluster. You may fill in any mock details, but the Name of the API Gateway Clusters must be same with the instance group name you specify for the nginx-gw in docker-compose file, in this case "gwcluster"

![alt text](assets/infra-workspace.png)
![alt text](assets/infra-env.png)

- Once done, click into the API Gateway section, you will see there are 2 nginx-gw in the Instances section.
![alt text](assets/infra-creation-complete.png)

Optional:
By scaling the nginx-gw instances, the newly spin up instance will auto register as part of the API Gateway Cluster instance group. You will notice additional instance in the Instances section.
```
# Edit docker-compose file, under nginx-gw section, change the replicas to 3
docker-compose -f docker-compose.yaml up -d
```

- Under API Connectivity Manager, Services, create a Workspace, Publish API Proxy
```    
Name: <anything>
Service Target Hostname: httpbin-app
Use an OpenAPI spec: Yes
Upload repo misc/httpbin-oas.yaml
Gateway Proxy Hostname: Select the gateway proxy that you have created
```

![alt text](assets/service-workspace.png)
![alt text](assets/service-upload-api-spec.png)
![alt text](assets/service-api-proxy.png)


- In Advanced Configuration, inisde Ingress section change the following fields and then Click Save and Publish
```
Append Rule: None
Strip Base Path and Version before proxying the request: Yes
```
![alt text](assets/service-edit-ingress.png)


Test the ACM setting
```
#You may refer to the misc/httpbin-oas.yml for other possible httpbin endpoints
curl -v localhost/get
curl -v localhost/headers
```

You may try to configure different policies in ACM and see how it work.

6. **Configure Dev Portal using ACM**

Enable Dev Portal Cluster
- In the previously created API Connectivity Manager->Infrastructure->Workspace->Environment. Inside the environment, Create Developer Portal Cluster.
The Name of the Dev Portal Clusters must be same with the instance group name you specify for the acm.nginx-devportal, in this case "devportal".

![alt text](assets/create-devportal-cluster.png)

- In the previously created API Connectivity Manager->Services-API Proxies, Tick "Also publish API to developer portal"

![alt text](assets/publish-devportal.png)

- Use browser to visit http://localhost:90 which is the Dev Portal container IP/Port

![alt text](assets/sample-devportal.png)

## Bonus
Instead of using NGINX Plus as LB, you may use NGINX App Protect (NAP) as LB + WAF to protect the API endpoints.
At the time of this writing, it is possible to manage NAP policies via NMS in VM setup but not in docker container. Hence, we will just manually create NAP policies in NGINX NAP instance and then manage the nginx.conf via NMS.

![alt text](assets/demo-setup-part-2.png)

```
#Build NGINX App Protect image with nginx-agent
#Specify NMS IP address (your laptop IP address), DO NOT use localhost or 127.0.0.1
./scripts/buildNAPWithAgent.sh -t napwithagent -n https://192.168.1.3

#Uncomment nginx-nap section in docker-compose.yaml section
docker-compose -f docker-compose.yaml up -d
```

Configure NAP in Instance Manager

- In Instance Manager section, Instances tab, click on nginx-nap and Edit Config, replace nginx.conf with repo misc/nap_lb.conf
![alt text](assets/edit-nap-conf.png)

- Click Publish

```
#Send traffic to NAP
curl localhost:83/get

#Violation
curl "localhost:83/get?<script>"
curl "localhost:83/get?username=1'%20or%20'1'%20=%20'1'))%20LIMIT%201/*&amp;password=foo"
```

After generating some violations, head over to NMS Security Monitoring dashboard to view the report.
![alt text](assets/nms-security-monitoring.png)


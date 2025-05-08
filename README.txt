Kubernetes Capstone Project:

Region: us-east-1

Description:
Have use the events-api app for the deployment. The front end service is being deployed as 2 replica set and exposed via a load balancer service. The back end service is deployed with 3 replicas and is exposed via a cluster Ip Service. 
The user can connect to Load balancer dns to connec to the app. The front end app is pointing to the cluster ip service of the backned. 
Job is being used to initialize the database like creating table, inserting random records. 
The DB details as being passed as env variables to backend api. 
Both frontend and backend layer is attacked to auto scaler.

Tried to segragate the layers using kustomization. Will have to run one one command to trigger deployment, service and hpa for repective namespace. 

Steps:
1. Create a new EC2 instance with basic Linux AMI in default subnet with internet connectivity, region = us-east-1 and enter below user-data in advanced details section while creating ec2. This will install terraform.
#!/bin/bash 
sudo yum install -y yum-utils
cd /home/ec2-user/
git clone <new url>
chown -R ec2-user:ec2-user /home/ec2-user/SD-AWS-ROI-Capstone_Project
2. Allow it some time to execute user data. It will take git checkout. So no need to checkout manually.
3. Create and download the secret access key for your account, if not already downloaded
4. Connect the ec2 instance using ec2 Instance connect
5. Configure the aws with access keys using 'aws configure' command. Enter region as us-east-1
6. Go inside the git checkout folder using 'cd <Newpath>'
7. Execute setup.sh using command ''
8. The above setup will install docker, eksctl, kubectl, helm.
9. Docker images are already created for the app and is pushed to docker hub. 
		[* Optional: The images can be run using below commands: ]
10. There will be cluster.config file used to create a cluster. Execute below command to start executing cluster:
   eksctl create cluster -f cluster.config
11. Once cluster is created, it can be validated using below commands:
	<commands>
12. Now do inside folder resources to create namespace, jobs, deployments, services, hpa
13. Have tried to keep separate layers for namespaces. Let create the resources for dev namespace.
14. Run below command to create dev namespace
	<command>
15. Lets mark gpu as the default storage classe which is a pre-requisite for database which we will be installing.
16. Run below command to install helm mariadb for dev namespace.
	helm install database-server oci://registry-1.docker.io/bitnamicharts/mariadb --namespace dev
17. Once mariadb is installed, we can run a job which will just initialize the database with required table and dummy data.
18. After db setup, we can now deploy the application using below command.
	<command>
19. We can use the Load balancer url of the url which we can fetch using:

20. Post validation, we can now try to deploy a new version of events-web (front-end layer).
	We already have created a green deployment as we have 2 deploymentfiles kept for front-end layer pointing to different versions
	Lets change manually the service file to point to green deployment using below command:
		

21. Once deploymeny is done, we can delete the kube infra using below commands.



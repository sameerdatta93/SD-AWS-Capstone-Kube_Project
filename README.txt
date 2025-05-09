Kubernetes Capstone Project:

Region: us-east-1

Description:
Events-app application is used here for the deployment. The front end service is being deployed as 2 replica sets and exposed via a load balancer service. The back end service is also deployed as 2 replicas and is exposed via a cluster Ip Service. 
The user can connect to Load balancer dns to connect to the app. The front end app is pointing to the cluster ip service of the backend. 
Job is being used to initialize the database like creating table, inserting random records. 
The DB details are being passed as env variables to backend api. As we have used namespace in this project, so had to create configMap inside kustomization.yaml to pass the corresponding env variable(based on namespace for dbhost) to events-api deployment.
Both frontend and backend layer is attached to auto scaler which has min as 2 and max as 6 configured and have added stress container on events-api deployments to simulate hpa. Once deployed the cpu utilization will be increased for events-api and it will be autoscaled automatically.
Made an attempt here to segragate the layers using kustomization.yaml and using one command we can trigger the creation of deployment, service and hpa for repective namespace. 

Steps:
1. Create a new EC2 instance with basic Linux AMI in default subnet with internet connectivity, region = us-east-1 and enter below user-data in advanced details section while creating ec2. This will install git and take a checkout.
#!/bin/bash 
sudo yum install git -y
cd /home/ec2-user/
git clone https://github.com/sameerdatta93/SD-AWS-Capstone-Kube_Project.git
chown -R ec2-user:ec2-user /home/ec2-user/SD-AWS-Capstone-Kube_Project
2. Allow it a few seconds to execute user data. So no need to take git checkout manually.
3. Create and download the secret access key for your account, if not already downloaded
4. Connect the ec2 instance using ec2 Instance connect and configure the aws with access keys using 'aws configure' command. Enter region as us-east-1
5. Go inside the git checkout folder using 'cd SD-AWS-Capstone-Kube_Project'
6. Execute setup.sh using command 'sh ./setup.sh'
7. The above setup will install docker, eksctl, kubectl and helm.
8. Docker images are already created for the app and is pushed to docker hub. 
	[*Additional info: Below are the commands used to create and push images:
				sudo docker build -t samdatta93/events-website-cp:<tag> .
				sudo docker push samdatta93/events-website-cp:<tag>]
9. There will be cluster.yaml file that will be used to create a cluster. Execute below command to start creating cluster:
   eksctl create cluster -f cluster.yaml
10. Once cluster is created, it can be validated using below few commands:
	kubectl cluster-info
	kubectl get nodes
11. Now lets create namespace, jobs, deployments, services, hpa. (Built separate layers for the namespaces. Lets create the resources for dev namespace now.)
12. Run below command to create dev namespace. This will create dev namespace
	kubectl apply -f resources/overlays/dev/app/namespace.yaml
13. Lets mark gpu as the default storage class which is a required for database which we will be installing.
	kubectl patch storageclass gp2 -p '{"metadata": {"annotations":{"storageclass.kubernetes.io/is-default-class":"true"}}}'
14. Run below command to install helm mariadb for dev namespace.
	helm install database-server oci://registry-1.docker.io/bitnamicharts/mariadb --namespace dev
    The installation can be verified using commands 'helm list -an dev'
    Ensure database-server pod is running using command 'kubectl get pods -n dev'
15. Once mariadb is installed, we can run a job which will just initialize the database with required table and dummy details. [The job will be fetched from docker repo.]
	kubectl apply -k resources/overlays/dev/init/
   We can check if the pods is running or not using 'kubectl get pods -n dev'	
   Logs for the job can be checked using command 'kubectl logs <pod-name> -n dev'
16. After db setup, we can now deploy the application using below command.
	kubectl apply -k resources/overlays/dev/app/
     This will create deployments for events-api and events-website along with their respective service (LB for events-api and clusterIP for events-website) and HPA. Use below commands to verify: [The apps will be fetched from docker repo]
	kubectl get deploy -n dev
	kubectl get service -n dev
	kubectl get hpa -n dev
    Lets wait for few minutes and then try to access the load balancer dns. [Since stress container was already included in events-api deployment, pods of events-api will get scaled automatically as cpu utilization will be > 30%]
17. We can use the Load balancer dns and try to access it in the browser on port 80:
		kubectl get service -n dev
18. Post validation, we can now try to deploy a new version of events-website (front-end layer).
	We already have created a green deployment (with name deployment_green.yaml). So there are two deployment files present inside events-website pointing to different versions. Currently you can see v3 on ui page and after deployment it will be changed to v4.
	[The green deployment got already created when we executed the command 'kubectl apply -k resources/overlays/dev/app/'; as two deployments were already created inside events-website directory]
	Now lets change manually the service.yaml to point to green deployment using below command:
	     'vim resources/base/events-website/service.yaml' -> change version to v2.0 from v1.0 under selector and save it (Esc -> :wq!) [v2.0 versions is for the deploment_green.yaml]
	Once updated we can apply and check if we are able to get the new version in the browser. We can apply using below command:
	kubectl apply -k resources/overlays/dev/app/
19. Once deployment is done, we can delete the kube infra using below commands.
	kubectl delete -k resources/overlays/dev/app/
	kubectl delete -k resources/overlays/dev/init/
	kubectl delete -f resources/overlays/dev/app/namespace.yaml

	Cluster can be destroyed using below command:
		eksctl delete cluster -f cluster.yaml 

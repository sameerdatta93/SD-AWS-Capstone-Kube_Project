Kubernetes Capstone Project:

Region: us-east-1

Description:
Events-app application is used here for the deployment. The front end service is being deployed as 2 replica set and exposed via a load balancer service. The back end service is deployed with 2 replicas and is exposed via a cluster Ip Service. 
The user can connect to Load balancer dns to connect to the app. The front end app is pointing to the cluster ip service of the backend. 
Job is being used to initialize the database like creating table, inserting random records. 
The DB details as being passed as env variables to backend api. As we have used namespace, so had to create configMapGeneratoe inside kustomization.yaml to pass the env variable to deployment.
Both frontend and backend layer is attached to auto scaler which has min as 2 and max as 6 configured Have added stress container on events-api deployments to simulate hpa. Once deployed the cpu utilization will be increase for events-api and it will be autoscaled automatically.

Tried to segragate the layers using kustomization.yaml and using one command we can trigger the creation of deployment, service and hpa for repective namespace. 

Steps:
1. Create a new EC2 instance with basic Linux AMI in default subnet with internet connectivity, region = us-east-1 and enter below user-data in advanced details section while creating ec2. This will install terraform.
#!/bin/bash 
sudo yum install git -y
cd /home/ec2-user/
git clone https://github.com/sameerdatta93/SD-AWS-Capstone-Kube_Project.git
chown -R ec2-user:ec2-user /home/ec2-user/SD-AWS-Capstone-Kube_Project
2. Allow it few second to execute user data. It will take git checkout. So no need to checkout manually.
3. Create and download the secret access key for your account, if not already downloaded
4. Connect the ec2 instance using ec2 Instance connect and configure the aws with access keys using 'aws configure' command. Enter region as us-east-1
5. Go inside the git checkout folder using 'cd SD-AWS-Capstone-Kube_Project'
6. Execute setup.sh using command 'sh ./setup.sh'
7. The above setup will install docker, eksctl, kubectl and helm.
8. Docker images are already created for the app and is pushed to docker hub. 
	[* Additional info: Below are the commands used to create and push images:
				sudo docker build -t samdatta93/events-website-cp:<tag> .
				sudo docker push samdatta93/events-website-cp:<tag>]
9. There will be cluster.config file used to create a cluster. Execute below command to start creating cluster:
   eksctl create cluster -f cluster.yaml
10. Once cluster is created, it can be validated using below few commands:
	kubectl cluster-info
	kubectl get nodes
11. Now go inside folder named resources to create namespace, jobs, deployments, services, hpa (Have tried to keep separate layers for the namespaces. Let create the resources for dev namespace now. )
12. Run below command to create dev namespace. This will create dev namespace
		kubectl apply -f resources/overlays/dev/app/namespace.yaml
13. Lets mark gpu as the default storage class which is a required for database which we will be installing.
		kubectl patch storageclass gp2 -p '{"metadata": {"annotations":{"storageclass.kubernetes.io/is-default-class":"true"}}}'
14. Run below command to install helm mariadb for dev namespace.
		helm install database-server oci://registry-1.docker.io/bitnamicharts/mariadb --namespace dev
	The installtion can be verified using commands 'helm list -an dev'
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
17. We can use the Load balancer dns and try accessin it in the browser for port 80:
		kubectl get service -n dev
18. Post validation, we can now try to deploy a new version of events-web (front-end layer).
	We already have created a green deployment (with name deployment_green). So there are two deployment files present inside events-website pointing to different versions. Currently you can see v3 in ui and after dpeloyment it will be v4 on ui.
	[The green deployment got already created when we executed the command 'kubectl apply -k resources/overlays/dev/app/'; as teo deployments are already created]
	Now lets change manually the service.yaml to point to green deployment using below command:
	     'vim resources/base/events-website/service.yaml' -> change version to v2.0 from v1.0 under selector and save it (Esc -> :wq!)
	Once updated we can apply and check if we are able to get the new version in the browser.
19. Once deployment is done, we can delete the kube infra using below commands.
		kubectl delete -k resources/overlays/dev/app/
		kubectl delete -k resources/overlays/dev/init/
		kubectl delete -f resources/overlays/dev/app/namespace.yaml

	Cluster can be destroyed using below command:
		eksctl delete cluster -f cluster.yaml


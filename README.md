# Milestone 4 - Special #

| Member                 | Contribution |
| :---                   | :---         |
| Dian Ding(dding3)      | iTrust auto scaling & Nginx traffic monitoring & linear regression algorithm |
| Kai Lu(klu2)           | ESLint for checkbox.io |
| Xiangqing Ding(xding3) | Presentation |
| Fuxing Luan(fluan)     | Test & Report |

## ESlint ##
### Rationale
ESlint is a powerful tool to check you coding style which can ensure your code quality and readability. However, it's not reasonable for every developer to follow the same coding style since every person has their own preference. So this tool can be used as a restriction for everyone to follow the same style by simply running one command.

### Instruction
Copy the [pre-commit](./Deployment/checkboxio/pre-commit) to your *project_root/.git/hooks* directory, then every time you commit code changes to the repository, the pre-commit hook will start to check if your changes have follow the coding style that we set, if not, the commit will fail and ask you to fix your code with executing one command.

## Auto Scaling ##
### Structure
	Jenkins
		- iTrust-Deployment 
		- iTrust-Rolling-Update
		- iTrust-Scale-Up
		- iTrust-Scale-Down

### Rationale
* iTrust-Deployment [Config](./Deployment/Jenkins/roles/iTrust/templates/iTrust.config.xml.j2) 
	1. Compile the application to make sure it's bug free
	2. Pack up the application to a war package for deployment
	
* iTrust-Rolling-Update [Config](./Deployment/Jenkins/roles/iTrust/templates/iTrust_Deployment.config.xml.j2)
	1. Create and provision EC2 instances with [Create Instance](./Deployment/iTrustPostBuild/get_instance.yml) script
	2. Setup environment for all instances   
	3. Deploy the application to all instances using rolling update policy with the [deploy](./Deployment/iTrustPostBuild/deploy.yml) script 
	4. Configure Nginx as a reverse proxy and load balancer for all instances with this [Config Nginx](./Deployment/iTrustPostBuild/config_nginx.yml) script
	
* iTrust-Scale-Up [Config](./Deployment/Jenkins/roles/iTrust/templates/iTrust_Scale_Up.config.xml.j2)    
	1. Create and provision new EC2 instance for scaling up
	2. Setup environment for the new instance
	3. Deploy the application to the new instance
	4. Add new host ip to the fleet and Nginx config file to bring it online
	
* iTrust-Scale-Down [Config](./Deployment/Jenkins/roles/iTrust/templates/iTrust_Scale_Down.config.xml.j2)    
	1. Check if hosts number in our fleet is greater than 1
	2. Pick the last host in our fleet
	3. Terminate the host from EC2 with [Terminate Instance](./Deployment/iTrustPostBuild/terminate_instance.yml)
	4. Remove it from our fleet and remove it from Nginx config to bring it offline

* Monitor Nginx Traffic and Scale  
	With this [monitor.sh](./Deployment/Jenkins/roles/iTrust/templates/monitor.sh.j2) shell script, a cron job will be setup to gather traffic rate every minutes and store the traffic data into a local file. 
	Another cron job with [linear_regression.py](./Deployment/Jenkins/roles/iTrust/files/linear_regression.py) script will be setup to run every hour. The job will take the traffic rate from the result that is generated from above job and do a linear regerssion on the data from past 60 minutes and make a prediction for the next 60 minutes, if the prediction result tends to pass the up threshold that we set, it will trigger the iTrust-Scale-Up job to create more instance to our cluster. Otherwise, if the prediction result tends to go below the down threshold, it will trigger the iTrust-Scale-Down job to terminate a instance. 

### Instruction
1. Navigate to [Jenkins Playbook](./Deployment/Jenkins) directory and Run: `ansible-playbook playbook.yml -i inventory`  
After the script finished executing, Jenkins will be installed and four jobs will be setup properly. And also Nginx will be setup and two crontab jobs will be created in the same server where Nginx is hosted.
2. Whenever you commit changes to *production* branch of the iTrust project, the iTrust-Deployment job will start building. After it's done, iTrust-Rolling-Update job will start to build automatically and this job will do all the provisioning and configuring if it's the first time the job is running, otherwise it will only update the production servers using the rolling update policy.
3. The Nginx monitoring job will gather Nginx traffic data continuously every minute. And every hour, the linear regression model will make prediction for the next hour, if the traffic rate tends to burst over the burst threshold, it will trigger the iTrust-Scale-Up job to add new instance to our cluster. On the contrary, if the the traffic rate tends to go down below the bottom threshold, it will trigger iTrust-Scale-Down job to terminate one instance (if we still have more than one host in our cluster) in order to save budget and resources. 

	
## Presentation Video ##

[https://www.youtube.com/watch?v=pDOzTCIJXRY](https://www.youtube.com/watch?v=pDOzTCIJXRY)



#Prerequisites 

	Linux machine
	
	awscli with valid keys
	https://docs.aws.amazon.com/cli/latest/userguide/getting-started-install.html
	
	eksctl installed 
	https://www.hackerxone.com/2021/08/20/steps-to-install-kubectl-eksctl-on-ubuntu-20-04/
	
	Where ever you see ${{aws_acct_num}} in this document replace with your account number

#Git clone the repo

	git clone https://github.com/apexontop/hablab.git
	
	cd hablab

#Edit eks-cluster.yaml with correct subnets and region

#Create cluster based on eks-cluster.yaml
    
	cat eks-cluster.yaml
    eksctl create cluster -f eks-cluster.yaml


#get config amend paths for your shell 

    aws eks --region eu-west-1 update-kubeconfig --name k8s-hablab-cluster
    
    cd /mnt/c/users/james.calleja/.kube
    
    cp ~/.kube/config  ./config
	
	cd /mnt/c/opscode/hablab


#enable OIDC

    eksctl  utils associate-iam-oidc-provider --cluster=k8s-hablab-cluster --approve
	
	aws eks describe-cluster --name k8s-hablab-cluster --query "cluster.identity.oidc.issuer" --output text
        

#install load balancer controller 
   	
	aws iam create-policy --policy-name AWSLoadBalancerControllerIAMPolicy --policy-document file://iam_policy.json 
	
	
#Create a service account and map the policy to it  

	eksctl create iamserviceaccount \
	  --cluster=k8s-hablab-cluster \
	  --namespace=kube-system \
	  --name=aws-load-balancer-controller \
	  --attach-policy-arn=arn:aws:iam::394990067255:policy/AWSLoadBalancerControllerIAMPolicy \
	  --override-existing-serviceaccounts \
	  --approve
	
	kubectl apply -f v2_3_1_full.yaml
	
	kubectl get deployment -n kube-system aws-load-balancer-controller
	
	
#Set up Iam role 

	TRUST="{ \"Version\": \"2012-10-17\", \"Statement\": [ { \"Effect\": \"Allow\", \"Principal\": { \"AWS\": \"arn:aws:iam::394990067255:root\" }, \"Action\": \"sts:AssumeRole\" } ] }"

	echo '{ "Version": "2012-10-17", "Statement": [ { "Effect": "Allow", "Action": "eks:Describe*", "Resource": "*" } ] }' > /tmp/iam-role-policy

	aws iam create-role --role-name HablabCodeBuildKubectlRole --assume-role-policy-document "$TRUST" --output text --query 'Role.Arn'

	aws iam put-role-policy --role-name HablabCodeBuildKubectlRole --policy-name eks-describe --policy-document file:///tmp/iam-role-policy
	
#patch aws-auth

	ROLE="    - rolearn: arn:aws:iam::394990067255:role/HablabCodeBuildKubectlRole\n      username: build\n      groups:\n        - system:masters"

	kubectl get -n kube-system configmap/aws-auth -o yaml | awk "/mapRoles: \|/{print;print \"$ROLE\";next}1" > /tmp/aws-auth-patch.yml

	kubectl patch configmap/aws-auth -n kube-system --patch "$(cat /tmp/aws-auth-patch.yml)"
	
	kubectl describe  configmap aws-auth -n kube-system
	
	
#Deploy app 	

#Edit line 38 of ci-cd-codepipeline.cfn.yml with the key provided 
	
	aws cloudformation create-stack --stack-name hablab-cicd-pipeline --template-body file://ci-cd-codepipeline.cfn.yml --capabilities CAPABILITY_NAMED_IAM
	
	kubectl get service hello-k8s -o wide
	
	
#Clean up 
	
	Delete all the cloudformation stacks 
	goto ECR and delete the repo
	goto S3 empty and delete the bucket 
	goto IAM delete HablabCodeBuildKubectlRole and AWSLoadBalancerControllerIAMPolicy
	goto cloudwatch and delete log groups
	
	
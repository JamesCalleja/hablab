#Git clone the repo

	git clone https://github.com/apexontop/hablab.git
	
	cd /hablab


#Create cluster based on eks-cluster.yaml
    
	cat eks-cluster.yaml
    eksctl create cluster -f eks-cluster.yaml


#get config 

    aws eks --region eu-west-1 update-kubeconfig --name k8s-hablab-cluster
    
    cd /mnt/c/users/james.calleja/.kube
    
    cp ~/.kube/config  ./config


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
	  --attach-policy-arn=arn:aws:iam::${{aws_acct_num}}:policy/AWSLoadBalancerControllerIAMPolicy \
	  --override-existing-serviceaccounts \
	  --approve
	
	kubectl apply -f v2_3_1_full.yaml
	
	kubectl get deployment -n kube-system aws-load-balancer-controller
	
	
#Set up Iam role 

	TRUST="{ \"Version\": \"2012-10-17\", \"Statement\": [ { \"Effect\": \"Allow\", \"Principal\": { \"AWS\": \"arn:aws:iam::${{aws_acct_num}}:root\" }, \"Action\": \"sts:AssumeRole\" } ] }"

	echo '{ "Version": "2012-10-17", "Statement": [ { "Effect": "Allow", "Action": "eks:Describe*", "Resource": "*" } ] }' > /tmp/iam-role-policy

	aws iam create-role --role-name HablabCodeBuildKubectlRole --assume-role-policy-document "$TRUST" --output text --query 'Role.Arn'

	aws iam put-role-policy --role-name HablabCodeBuildKubectlRole --policy-name eks-describe --policy-document file:///tmp/iam-role-policy
	
#patch aws-auth

	ROLE="    - rolearn: arn:aws:iam::${{aws_acct_num}}:role/HablabCodeBuildKubectlRole\n      username: build\n      groups:\n        - system:masters"

	kubectl get -n kube-system configmap/aws-auth -o yaml | awk "/mapRoles: \|/{print;print \"$ROLE\";next}1" > /tmp/aws-auth-patch.yml

	kubectl patch configmap/aws-auth -n kube-system --patch "$(cat /tmp/aws-auth-patch.yml)"
	
	kubectl describe  configmap aws-auth -n kube-system
	
	
#Deploy app 	

	aws cloudformation create-stack --stack-name hablab-cicd-pipeline --template-body file://ci-cd-codepipeline.cfn.yml --capabilities CAPABILITY_NAMED_IAM
	
	kubectl get service hello-k8s -o wide
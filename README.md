# kubernetes

# install AWS CLI
# 1. configure aws cli 
    curl "https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip" -o "awscliv2.zip"
    unzip awscliv2.zip
    sudo ./aws/install
    aws configure 


# 2. Install IAM-authenticator 
   curl -Lo aws-iam-authenticator https://github.com/kubernetes-sigs/aws-iam-authenticator/releases/download/v0.5.9/aws-iam-authenticator_0.5.9_linux_amd64
    chmod +x ./aws-iam-authenticator
    mkdir -p $HOME/bin && cp ./aws-iam-authenticator $HOME/bin/aws-iam-authenticator && export PATH=$PATH:$HOME/bin
    echo 'export PATH=$PATH:$HOME/bin' >> ~/.bashrc
    aws-iam-authenticator help

# 3. Install kubectl 
    curl -O https://s3.us-west-2.amazonaws.com/amazon-eks/1.27.1/2023-04-19/bin/linux/amd64/kubectl
    chmod +x ./kubectl
    mkdir -p $HOME/bin && cp ./kubectl $HOME/bin/kubectl && export PATH=$PATH:$HOME/bin
    echo 'export PATH=$PATH:$HOME/bin' >> ~/.bashrc
    kubectl version --short --client


# 5. Install eksctl 
    curl --silent --location "https://github.com/weaveworks/eksctl/releases/latest/download/eksctl_$(uname -s)_amd64.tar.gz" | tar xz -C /tmp
    sudo mv /tmp/eksctl /usr/local/bin
    eksctl version

# 6. create EKS cluster using eksctl
    $ eksctl create cluster --name <cluster-name> --node-type t2.medium --nodes 2 --nodes-min 1 --nodes-max 3 --region us-east-1
  # Launch EKS Cluster with asg access.
    While launching the EKS cluster, you have to enable the asg access so that the worker nodes will launch by asg that is Auto Scaling Group.
    $ eksctl create cluster  --profile user1 --asg-access --name my-cluster-1 --asg-access --nodes-max 10 --nodes-min 2 --nodes 3 --node-type t2.small --nodegroup-name Node-group-A --ssh-access --enable-ssm --version 1.25 --region us-west-1

  # NOTE - you can remove ssh-access,enable-ssm ,verion ,nodegroup , profile- aws user ,

# 7. kubectl update kube-config.
    $ aws sts get-caller-identity
    $ aws --version
    $ aws eks update-kubeconfig --region us-east-1 --name <cluster name>

    it will create $/home/user/.kube/config file. 

# 8. delete cluster 
    $ eksctl delete cluster --name <cluster-name> --region <region>

------------------------------------------------------------------------------------------------------------------



# Why Use OIDC?
    Enabling OIDC and using IRSA is the recommended way to manage IAM permissions because it:

    Improves security by limiting access per service account.
    Avoids granting broad permissions to all nodes in the cluster.

# create eks cluster using kubectl:
 $  eksctl create cluster --name sharma --node-type t2.medium --nodes 2 --nodes-min 1 --nodes-max 3 --region us-west-2

# verify
 $ kubectl get pods -n kube-system
 $ kubectl get node
 $ eksctl get cluster --name sharma --region us-west-2

# Note:
 the Amazon EKS cluster you created does not have OIDC (OpenID Connect) identity provider enabled. Without OIDC, IAM permissions for the VPC CNI add-on cannot be automatically configured using IAM roles for service accounts (IRSA). Instead, you must configure these permissions manually.

# Enable OIDC for the Cluster
   OIDC allows your EKS cluster to associate IAM roles with Kubernetes service accounts for secure permission management.
   $ eksctl utils associate-iam-oidc-provider --region <region> --cluster <cluster-name> --approve

# Verify that OIDC is associated:
 $ eksctl get cluster --name <cluster-name> --region <region>


# Create an IAM Role with Recommended Policie
 . AWS provides a set of recommended IAM policies for the VPC CNI add-on.
   # You can create the role with these policies using eksctl:
      
      $ eksctl create iamserviceaccount \
  --region <region> \
  --name aws-node \
  --namespace kube-system \
  --cluster <cluster-name> \
  --attach-policy-arn arn:aws:iam::aws:policy/AmazonEKS_CNI_Policy \
  --approve \
  --override-existing-serviceaccounts

# Update the Add-on with Pod Identity Associations:
 . After creating the IAM role, update the VPC CNI add-on configuration to use the new role. Update your eksctl configuration file (cluster.yaml):
  
 cluster. ymal file 
  addons:
  - name: vpc-cni
    version: latest
    PodIdentityAssociations:
      - namespace: kube-system
        serviceAccount: aws-node
        identity:
          - arn:aws:iam::<account-id>:role/<vpc-cni-role-name>

# Then, run:
 $ eksctl update addon --name vpc-cni --cluster <cluster-name> --force

# if you donot find cluster.yaml file : Generate a New cluster.yaml File

        # Run the following command to create a cluster.yaml file from your existing cluster configuration:

        $ eksctl get cluster --name <cluster-name> --region <region> -o yaml > cluster.yaml
        
        # Open the generated cluster.yaml file for editing:
            $ nano cluster.yaml
        
        # Add or update the addons section to include the VPC CNI configuration:

            addons:
        - name: vpc-cni
            version: latest
            PodIdentityAssociations:
            - namespace: kube-system
                serviceAccount: aws-node
                identity:
                - arn:aws:iam::<account-id>:role/<vpc-cni-role-name>

        # OR  Manually Create the cluster.yaml File
        apiVersion: eksctl.io/v1alpha5
        kind: ClusterConfig

        metadata:
        name: <cluster-name>          # Replace with your cluster name
        region: <region>              # Replace with your AWS region

        addons:
        - name: vpc-cni
            version: latest
            PodIdentityAssociations:
            - namespace: kube-system
                serviceAccount: aws-node
                identity:
                - arn:aws:iam::<account-id>:role/<vpc-cni-role-name>


        # Update the VPC CNI Add-on:
        $ eksctl update addon --config-file cluster.yaml --name vpc-cni --force

        # Verify the Update: Check if the VPC CNI add-on is updated successfully:
        $ kubectl describe daemonset aws-node -n kube-system
        $ kubectl get daemonset aws-node -n kube-system
        $ kubectl logs -n kube-system -l k8s-app=aws-node


# Manual Workaround Without OIDC  :

If you cannot enable OIDC, you need to manually attach the recommended IAM policies to the node IAM role. Here's how:

# Find the Node IAM Role:
 
 $ aws eks describe-nodegroup --cluster-name <cluster-name> --nodegroup-name <nodegroup-name> --query "nodegroup.iam.roleArn" --region <region>

# Attach Policies to the Role: Use the AmazonEKS_CNI_Policy ARN to attach the policy to your node IAM role:

  $ aws iam attach-role-policy --role-name <node-role-name> --policy-arn arn:aws:iam::aws:policy/AmazonEKS_CNI_Policy

# . Validate the Configuration
  $ kubectl get pods -n kube-system

  $ kubectl get daemonset aws-node -n kube-system

----------------------------------------------------------------------------------------------------------------------

# Installing Helm 

# From Script
   $ curl -fsSL -o get_helm.sh https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3
   $ chmod 700 get_helm.sh
   $ ./get_helm.sh
   $ helm version

# From Apt (Debian/Ubuntu)
  $ curl https://baltocdn.com/helm/signing.asc | gpg --dearmor | sudo tee /usr/share/keyrings/helm.gpg > /dev/null

  $ sudo apt-get install apt-transport-https --yes

  $  echo "deb [arch=$(dpkg --print-architecture) signed-by=/usr/share/keyrings/helm.gpg] https://baltocdn.com/helm/stable/debian/  all main" 

  $ sudo apt-get update

  $ sudo apt-get install helm

#





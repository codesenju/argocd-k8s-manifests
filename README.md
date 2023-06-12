### Pre-requisite check
- Ingress Controller
- External DNS

### Create ECR Repository for your Application Docker Images
```bash
ECR_REPO_NAME=***

aws ecr create-repository --repository-name $ECR_REPO_NAME --image-tag-mutability IMMUTABLE --image-scanning-configuration scanOnPush=true

ECR_REPO_URL=$(aws ecr describe-repositories --repository-names $ECR_REPO_NAME --query 'repositories[0].repositoryUri' --output text)

587878432697.dkr.ecr.us-east-1.amazonaws.com/skillsets-api
587878432697.dkr.ecr.us-east-1.amazonaws.com/skillsets-ui
```

### Create CodeCommit Repository and push local code to repo
```bash
REPO_NAME="argocd-k8s-manifests"
aws codecommit create-repository --repository-name $REPO_NAME
REPO_URL=$(aws codecommit get-repository --repository-name $REPO_NAME --query 'repositoryMetadata.cloneUrlHttp' --output text)
echo $REPO_URL


# -- NEW REPO
cd <your-directory>
git init
git add .
git commit -m "Initial commit"
git remote add codecommit $REPO_URL
git push codecommit master
# --- EXISTING REPO
cd <your-directory>
git remote add codecommit $REPO_URL
git push codecommit main
# --
```
### Create STS Assume IAM Role for CodeBuild to interact with AWS EKS
```bash
# Export your Account ID
export AWS_ACCOUNT_ID=$(aws sts get-caller-identity --query Account --output text)

# Set Trust Policy
TRUST="{ \"Version\": \"2012-10-17\", \"Statement\": [ { \"Effect\": \"Allow\", \"Principal\": { \"AWS\": \"arn:aws:iam::${AWS_ACCOUNT_ID}:root\" }, \"Action\": \"sts:AssumeRole\" } ] }"

# Verify inside Trust policy, your account id got replacd
echo $TRUST

# Create IAM Role for CodeBuild to Interact with EKS
aws iam create-role --role-name EksCodeBuildKubectlRole --assume-role-policy-document "$TRUST" --output text --query 'Role.Arn's

# Define Inline Policy with eks Describe permission in a file iam-eks-describe-policy
echo '{ "Version": "2012-10-17", "Statement": [ { "Effect": "Allow", "Action": "eks:Describe*", "Resource": "*" } ] }' > /tmp/iam-eks-describe-policy

# Associate Inline Policy to our newly created IAM Role
aws iam put-role-policy --role-name EksCodeBuildKubectlRole --policy-name eks-describe --policy-document file:///tmp/iam-eks-describe-policy

# Verify the same on Management Console
```

### Update EKS Cluster aws-auth ConfigMap with new role created in previous step
```bash
# Verify what is present in aws-auth configmap before change
kubectl get configmap aws-auth -o yaml -n kube-system

# Export your Account ID
export AWS_ACCOUNT_ID=$(aws sts get-caller-identity --query Account --output text)

# Set ROLE value
ROLE="    - rolearn: arn:aws:iam::${AWS_ACCOUNT_ID}:role/EksCodeBuildKubectlRole\n      username: build\n      groups:\n        - system:masters"

# Get current aws-auth configMap data and attach new role info to it
kubectl get -n kube-system configmap/aws-auth -o yaml | awk "/mapRoles: \|/{print;print \"$ROLE\";next}1" > /tmp/aws-auth-patch.yml

# Patch the aws-auth configmap with new role
kubectl patch configmap/aws-auth -n kube-system --patch "$(cat /tmp/aws-auth-patch.yml)"

# Alternatively you can use eksctl
EKS_CLUSTER=**
eksctl create iamidentitymapping --cluster $EKS_CLUSTER --arn  arn:aws:iam::${AWS_ACCOUNT_ID}:role/EksCodeBuildKubectlRole --username build --group system:masters

# Verify what is updated in aws-auth configmap after change
kubectl get configmap aws-auth -o yaml -n kube-system
```

## CodeBuild

###
```bash
AWS_REGIOIN=$(aws configure get region)
# 587878432697.dkr.ecr.us-east-1.amazonaws.com/skillsets-api
# 587878432697.dkr.ecr.us-east-1.amazonaws.com/skillsets-ui
# Environment Variables for CodeBuild
echo "REPOSITORY_URI = ${AWS_ACCOUNT_ID}.dkr.ecr.${AWS_REGION}.amazonaws.com/skillsets"
echo "EKS_KUBECTL_ROLE_ARN = arn:aws:iam::${AWS_ACCOUNT_ID}:role/EksCodeBuildKubectlRole"
echo "EKS_CLUSTER_NAME = $EKS_CLUSTER"
```

## CodePipeline
```bash
aws codepipeline create-pipeline --pipeline-name eks-devops-pipe --pipeline file://pipeline.json
```

```bash
aws codepipeline get-pipeline --name eks-devops-pipeline --query 'pipeline' --output yaml > pipeline.yaml

aws codepipeline create-pipeline --cli-input-yaml file://pipeline.yaml
```

## Codebuild
```bash
aws codebuild batch-get-projects --names eks-devops-cb-for-pipeline --query 'projects[0]' --output yaml > project.yaml
```

# Argocd-k8s-manifests

## Add repo
```bash
  # Add a private Git repository via HTTPS using username/password:
  argocd repo add $REPO_URL --username git --password secret --name argocd-k8s-manifests
```
## Create argocd applications
```bash
REPO_NAME="argocd-k8s-manifests"
REPO_URL=$(aws codecommit get-repository --repository-name $REPO_NAME --query 'repositoryMetadata.cloneUrlHttp' --output text)
```
```bash
# Only deployable on the cluster where argocd server is installed!
cat <<EOF | kubectl apply -f -
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: $ENV-skillsets-api
  namespace: argocd
spec:
  destination:
    server: 'https://0807E0011E71891914718E9F1BC052A3.gr7.us-east-1.eks.amazonaws.com'
  source:
    repoURL: $REPO_URL
    path: killsets/kustomize.api/$ENV
    targetRevision: argocd
    kustomize:
      images:
      - SKILLSETS_API_IMAGE_NAME=codesenju/skillsets-api:latest
  project: uat
EOF

# Deployable anywhere!
ENV=uat
DEST_CLUSTER=https://0807E0011E71891914718E9F1BC052A3.gr7.us-east-1.eks.amazonaws.com
argocd app create $ENV-skillsets-api \
    --repo $REPO_URL \
    --dest-server $DEST_CLUSTER \
    --path skillsets/kustomize.api/$ENV \
    --revision HEAD \
    --project uat

#   --kustomize-image 'SKILLSETS_API_IMAGE_NAME=codesenju/skillsets-api:latest'

```
### skillsets-ui
```bash
# Only deployable on the cluster where argocd server is installed!

argocd app create $ENV-skillsets-ui \
    --repo  $REPO_URL \
    --dest-server 'https://0807E0011E71891914718E9F1BC052A3.gr7.us-east-1.eks.amazonaws.com' \
    --path skillsets/kustomize.ui/$ENV \
    --revision HEAD \
    --project uat

#  --kustomize-image 'SKILLSETS_UI_IMAGE_NAME=codesenju/skillsets-ui:v1'

argocd app delete $ENV-skillsets-ui -y
```
## Clean up
```bash
DEST_SERVER=***
# Option 1
cat <<EOF | kubectl apply -f -
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: $ENV-skillsets-api
  namespace: argocd
spec:
  destination:
    server: $DEST_SERVER
  source:
    repoURL: $REPO_URL
    path: skillsets/kustomize.api/$ENV
    targetRevision: HEAD
    kustomize:
      images:
      - SKILLSETS_API_IMAGE_NAME=codesenju/skillsets-api:latest
  project: uat
EOF
# Option 2
argocd app delete $ENV-skillsets-ui -y
```

## Deploy new image
```bash
ENV=***
REPOSITORY_URI=codesenju/skillsets-api
TAG=v1
cd skillsets/kustomize.ui/$ENV
kustomize edit set image SKILLSETS_UI_IMAGE_NAME=$REPOSITORY_URI:$TAG
```
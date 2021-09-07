# APP MESH in AWS EKS service #

**Check if your cluster can be upgraded if you are using a pree-released version of EKS**

`curl -o pre_upgrade_check.sh https://raw.githubusercontent.com/aws/eks-charts/master/stable/appmesh-controller/upgrade/pre_upgrade_check.sh
./pre_upgrade_check.sh`

> should say something like "your cluster is ready for upgrade". then proceed further

**Add the eks-charts repository to Helm**

`helm repo add eks https://aws.github.io/eks-charts`

**Add custom resource definition**

`kubectl apply -k "https://github.com/aws/eks-charts/stable/appmesh-controller/crds?ref=master"`

**Create a namespace**

`kubectl create ns appmesh-system`

**Create a Open Identity provider for the cluster using eksctl**

`eksctl utils associate-iam-oidc-provider \
    --region=$AWS_REGION \
    --cluster $CLUSTER_NAME \
    --approve`

**Create an IAM role, attach the AWSAppMeshFullAccess and AWSCloudMapFullAccess AWS managed policies to it, and bind it to the appmesh-controller Kubernetes service account. The role enables the controller to add, remove, and change App Mesh resources.**

`eksctl create iamserviceaccount \
    --cluster $CLUSTER_NAME \
    --namespace appmesh-system \
    --name appmesh-controller \
    --attach-policy-arn  arn:aws:iam::aws:policy/AWSCloudMapFullAccess,arn:aws:iam::aws:policy/AWSAppMeshFullAccess \
    --override-existing-serviceaccounts \
    --approve`

**Deploy the App Mesh Controller**

`helm upgrade -i appmesh-controller eks/appmesh-controller \
    --namespace appmesh-system \
    --set region=$AWS_REGION \
    --set serviceAccount.create=false \
    --set serviceAccount.name=appmesh-controller`

**Check the Controller Version**

`kubectl get deployment appmesh-controller \
    -n appmesh-system \
    -o json  | jq -r ".spec.template.spec.containers[].image" | cut -f2 -d ':'`

**Create namespace for App Mesh Resources**

`kubectl apply -f namespace.yaml`

**Create an App Mesh service mesh**

`kubectl apply -f mesh.yaml`

`kubectl describe mesh my-mesh`

`aws appmesh describe-mesh --mesh-name my-mesh`

**Create an App Mesh virtual node**

`kubectl apply -f virtual-node.yaml`

`kubectl describe virtualnode my-service-a -n my-apps`

`aws appmesh describe-virtual-node --mesh-name my-mesh --virtual-node-name my-service-a_my-apps`

**Create an App Mesh virtual router**

`kubectl apply -f virtual-router.yaml`

`kubectl describe virtualrouter my-service-a-virtual-router -n my-apps`

`aws appmesh describe-virtual-router --virtual-router-name my-service-a-virtual-router_my-apps --mesh-name my-mesh`

`aws appmesh describe-route \
    --route-name my-service-a-route \
    --virtual-router-name my-service-a-virtual-router_my-apps \
    --mesh-name my-mesh`

**Create an App Mesh virtual service** 

`kubectl apply -f virtual-service.yaml`

`kubectl describe virtualservice my-service-a -n my-apps`

`aws appmesh describe-virtual-service --virtual-service-name my-service-a.my-apps.svc.cluster.local --mesh-name my-mesh`

**Enable proxy authorization. We recommend that you enable each Kubernetes deployment to stream only the configuration for its own App Mesh virtual node**

`aws iam create-policy --policy-name my-policy --policy-document file://proxy-auth.json`

**Create an IAM role, attach the policy you created in the previous step to it, create a Kubernetes service account and bind the policy to the Kubernetes service account**

`eksctl create iamserviceaccount \
    --name saawsxray \
    --namespace appmesh-system \
    --cluster $CLUSTER_NAME \
    --region ap-northeast-1 \
    --attach-policy-arn arn:aws:iam::aws:policy/AWSXRayDaemonWriteAccess arn:aws:iam::611783125133:policy/my-policy-appmesh \
    --approve \
    --override-existing-serviceaccounts`

**Deploy the example service**

`kubectl apply -f example-service.yaml`

`kubectl -n my-apps get pods`

**should display pods as running**













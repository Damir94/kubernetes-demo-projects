# Deploying the Cluster Autoscaler on AWS EKS

  <img width="834" height="369" alt="Screenshot 2026-06-15 at 10 39 31 AM" src="https://github.com/user-attachments/assets/c46db163-0f1a-4969-a048-a5a3ff72d40e" />

### Plan: What we’ll build and why
- We will configure CA on an Amazon EKS cluster, using IRSA for secure, password-free authentication. This setup ensures that when Pods cannot be scheduled due to resource shortages, the
  CA will securely request new nodes from AWS. Conversely, when nodes are underutilized for a defined period, it will safely remove them to optimize costs. 
- Here are the high-level milestones we will cover, which you can use as a narrative guide in an interview:
  - Foundation and guardrails: Define our EKS cluster configuration, including the essential tags the CA needs for auto-discovery and the minimum/maximum size to prevent unexpected costs or outages.
  - Establishing trust (IRSA): Enable the OIDC provider for our cluster in AWS IAM, creating the foundation for secure communication between our cluster and the AWS APIs.
  - Principle of least privilege: Craft a fine-grained IAM policy that grants the CA only the permissions it needs, strictly scoped to the resources it’s supposed to manage.
  - Creating the identity: Generate the specific IAM role and Kubernetes ServiceAccount, linking them together through the power of IRSA.
  - Deployment: Install the CA using its official Helm chart, configuring it to use our IRSA setup and setting safe, production-sensible arguments.
  - Proof of work: Trigger a scale-up event by deploying a workload that exceeds our cluster’s capacity and watch the logs to confirm the CA is making intelligent decisions.
  - Troubleshooting: Discuss common failure modes and how to quickly diagnose them — a key skill for any operational role.


  <img width="833" height="450" alt="Screenshot 2026-06-15 at 10 38 36 AM" src="https://github.com/user-attachments/assets/e4f977b1-a8fb-46d4-81c8-e53fce0fe322" />

### Step 1: Prepping the environment and cluster
- Before we create the cluster, we’ll set up some environment variables to make our commandsclean and reusable. This is also a good practice to demonstrate in an interview, as it shows you value consistency and avoid manual errors.
- First, we define our cluster’s name, region, and AWS account ID. We then programmatically detect the latest supported Kubernetes version for EKS in our target region. This is a robust approach that avoids hardcoding versions, though we include a safe fallback.
```bash
set -Eeuo pipefail
# Basic context for our cluster
export CLUSTER_NAME="interview-ca-cluster"
export AWS_REGION="us-east-1"
export AWS_ACCOUNT_ID="$(aws sts get-caller-identity --query Account --output text)"
```
```bash
# Discover the latest supported Kubernetes version for EKS
export EKS_LATEST_VERSION="$( aws eks describe-addon-versions --region "$AWS_REGION" --query 'addons[].compatibilities[].clusterVersion' --output text | tr '\t' '\n' | sort -uV | tail -1 || true)"
```
```bash
# Fallback to a recent version if discovery fails
: "${EKS_LATEST_VERSION:=1.33}"
echo "Using: Account=$AWS_ACCOUNT_ID Region=$AWS_REGION Cluster=$CLUSTER_
NAME K8s=$EKS_LATEST_VERSION"
```

### Step 2: Creating the EKS cluster with auto-discovery tags
- The CA doesn’t automatically manage every node group in your cloud account. You must explicitly tell it which ones to control. It discovers these managed node groups using a specific set of AWS tags. The presence of these tags is non-negotiable; without them, the autoscaler will not detect or manage the Auto Scaling group.

- There are two mandatory tags:
  - k8s.io/cluster-autoscaler/enabled: A true value on this tag signals, “This Auto Scaling Group is available for management.”
  - k8s.io/cluster-autoscaler/<cluster-name>: This tag, with a value of owned, associates the node group with a specific EKS cluster, preventing a single Cluster Autoscaler from trying to manage nodes belonging to another cluster.

- We’ll embed these tags directly into our cluster configuration file (demo-cluster.yaml):
```yaml
apiVersion: eksctl.io/v1alpha5
kind: ClusterConfig
metadata:
name: ${CLUSTER_NAME}
region: ${AWS_REGION}
version: "${EKS_LATEST_VERSION}"
managedNodeGroups:
- name: managed-ng-1
minSize: 1
desiredCapacity: 1
maxSize: 5
instanceType: t3.medium
volumeSize: 20
labels: { role: worker }
# Required tags for Cluster Autoscaler autodiscovery
tags:
k8s.io/cluster-autoscaler/enabled: "true"
k8s.io/cluster-autoscaler/${CLUSTER_NAME}: "owned"
```

- Let’s break down the key fields in this managedNodeGroups definition:
  - minSize: 1 and maxSize: 5: These are our safety rails. The CA will never scale the node count below 1 or above 5 for this group. This is crucial for controlling costs and ensuring a baseline capacity.
  - tags: Here, we apply the two essential discovery tags. When eksctl creates the underlying AWS ASG, it will attach these tags, making the node group visible to the CA.

- Now, let’s create the cluster. This process can take 15-20 minutes as AWS provisions the control plane and worker nodes.
```bash
eksctl create cluster -f ./demo-cluster.yaml
```

<img width="1140" height="76" alt="Screenshot 2026-07-12 at 11 24 23 AM" src="https://github.com/user-attachments/assets/e2385eb4-7621-434f-bd11-ceda2bab424f" />

- Once the cluster is ready, we’ll configure kubectl to communicate with it:
```bash
aws eks update-kubeconfig --region "$AWS_REGION" --name "$CLUSTER_NAME"
```

### Step 3: Setting up IRSA and a least-privilege IAM policy
- To modify ASGs, the CA Pod needs AWS credentials. The most secure and recommended method for granting these permissions is IRSA. IRSA links a Kubernetes ServiceAccount to an AWS IAM role. This allows any Pod using that ServiceAccount to inherit the role’s permissions without needing static secret keys.

### Enable the IAM OIDC provider
- The first step is to create an OIDC identity provider for your cluster in IAM. This establishes a trust relationship, allowing IAM to accept identity tokens from your Kubernetes cluster’s OIDC provider. eksctl makes this a one-step process:
```bash
eksctl utils associate-iam-oidc-provider \
  --region "$AWS_REGION" \
  --cluster "$CLUSTER_NAME" \
  --approve
```

<img width="1306" height="73" alt="Screenshot 2026-07-12 at 11 29 08 AM" src="https://github.com/user-attachments/assets/851d4960-cfd0-4dcf-b96d-2c34f6e90da6" />

### Craft a least-privilege IAM policy
- Next, we’ll create an IAM policy that grants the CA only the permissions it needs. A common mistake is to grant overly broad permissions ("Resource": "*" everywhere). A stronger, more secure policy limits powerful actions to only the resources that are tagged for autoscaling.

```json
// cluster-autoscaler-policy.json
{
"Version": "2012-10-17",
"Statement": [
{
  "Effect": "Allow",
  "Action": [
    "autoscaling:DescribeAutoScalingGroups",
    "autoscaling:DescribeAutoScalingInstances",
    "autoscaling:DescribeLaunchConfigurations",
    "autoscaling:DescribeScalingActivities",
    "autoscaling:DescribeTags",
    "ec2:DescribeImages",
    "ec2:DescribeInstanceTypes",
    "ec2:DescribeLaunchTemplateVersions",
    "ec2:GetInstanceTypesFromInstanceRequirements",
    "eks:DescribeNodegroup"
  ],
  "Resource": "*"
},
{
  "Effect": "Allow",
  "Action": [
    "autoscaling:SetDesiredCapacity",
    "autoscaling:TerminateInstanceInAutoScalingGroup",
    "autoscaling:UpdateAutoScalingGroup"
  ],
  "Resource": "*",
  "Condition": {
    "StringEquals": {
      "aws:ResourceTag/k8s.io/cluster-autoscaler/enabled": "true"
    },
    "StringLike": {
      "aws:ResourceTag/k8s.io/cluster-autoscaler/*": "owned"
    }
   }
  }
 ]
}
```
- This IAM policy follows the principle of least privilege. It has two parts:
  - Read-only actions: It allows the autoscaler to read (Describe*) information about any node group or instance type. This is safe and necessary for it to see what’s available
  - Tag-restricted write permissions: It only grants powerful actions—such as changing capacity or terminating instances—to resources that have our specific autoscaling tags. This is a critical security measure that prevents the autoscaler from accidentally modifying the wrong set of machines.

- Now, let’s create this policy in IAM:
```bash
# Create the policy and capture its ARN
export CA_POLICY_ARN="$(
aws iam create-policy
--policy-name AmazonEKSClusterAutoscalerPolicy
--policy-document file://./cluster-autoscaler-policy.json
--query "Policy.Arn" --output text 2>/dev/null ||
aws iam list-policies --scope Local
--query "Policies[?PolicyName=='AmazonEKSClusterAutoscalerPolicy'].Arn |
[0]"
--output text
)"
echo "CA policy ARN: $CA_POLICY_ARN"
```

### Step 4: Create the IRSA and service account
- With the policy created, we can now create the final piece of the IAM puzzle. We need an IAM role that our service account can assume, and that role needs our new policy attached to it.
- Once again, eksctl simplifies this into a single, powerful command:
```bash
eksctl create iamserviceaccount
--cluster "$CLUSTER_NAME"
--region "$AWS_REGION"
--namespace kube-system
--name cluster-autoscaler
--attach-policy-arn "$CA_POLICY_ARN"
--approve
--override-existing-serviceaccounts
```

<img width="1341" height="266" alt="Screenshot 2026-07-12 at 11 36 15 AM" src="https://github.com/user-attachments/assets/5964a924-103e-4e7e-9fe4-d409985befca" />

- This command performs three actions:
  - Creates a new IAM role with a trust policy that allows the EKS OIDC provider to assume it
  - Attaches AmazonEKSClusterAutoscalerPolicy to this new role
  - Creates a Kubernetes ServiceAccount named cluster-autoscaler in the kube-system namespace and, most importantly, annotates it with the ARN of the IAM role it just created
- You can verify this by inspecting the ServiceAccount in Kubernetes:
```bash
kubectl -n kube-system get sa cluster-autoscaler -o yaml
```
- You should see an annotation like this, which is the magic that links Kubernetes to IAM:
```bash
annotations:
  eks.amazonaws.com/role-arn: arn:aws:iam::123456789012:role/eksctl-interview-ca-cluster-addon-iamserviceac-Role1-ABCDEFGHIJKL
```

### Step 5: Install the CA via Helm
- With the permissions correctly configured, we can now deploy the CA application itself. The recommended method is to use the official Kubernetes Helm chart.
- First, add the Helm repository:
```bash
helm repo add autoscaler https://kubernetes.github.io/autoscaler
helm repo update
```

<img width="1090" height="154" alt="Screenshot 2026-07-12 at 11 37 39 AM" src="https://github.com/user-attachments/assets/304dbdec-2788-4c8b-9a8b-3e83f516c420" />

- Now, we’ll install the chart. The parameters we pass via --set are crucial for a correct and secure installation.
```bash
helm upgrade --install cluster-autoscaler autoscaler/cluster-autoscaler \
  --namespace kube-system \
  --set nameOverride=aws-cluster-autoscaler \
  --set cloudProvider=aws \
  --set awsRegion="$AWS_REGION" \
  --set autoDiscovery.clusterName="$CLUSTER_NAME" \
  --set expander=least-waste \
  --set rbac.serviceAccount.create=false \
  --set rbac.serviceAccount.name=cluster-autoscaler \
  --set extraArgs.balance-similar-node-groups=true \
  --set extraArgs.scale-down-unneeded-time=5m \
  --set extraArgs.skip-nodes-with-local-storage=false \
  --set extraArgs.skip-nodes-with-system-pods=false
```

<img width="1308" height="459" alt="Screenshot 2026-07-12 at 11 39 05 AM" src="https://github.com/user-attachments/assets/89172f7e-8c71-4592-ad4e-2856e573a5aa" />

- Let’s highlight the most important settings for an IRSA-based setup:
  - autoDiscovery.clusterName="$CLUSTER_NAME": This tells the autoscaler which owner-ship tag to look for when discovering node groups.
  - rbac.serviceAccount.create=false: This is critical. We are telling the Helm chart, “Do not create a new ServiceAccount.”
  - rbac.serviceAccount.name=cluster-autoscaler: This tells the chart, “Instead, use the existing cluster-autoscaler ServiceAccount we already created and configured with IRSA.”

- Forgetting these two settings is a very common error. If the Helm chart creates its own ServiceAccount, it won’t have the IRSA annotation, and the CA Pod will fail with permission errors when trying to call AWS APIs.

### Step 6: Smoke test—Prove that it works!
- A deployment isn’t done until you’ve proven it works. We need to create a scenario that forces a scale-up, observe the Cluster Autoscaler’s response, and confirm the result.
- Remember, the trigger for the CA isn’t high CPU or memory; it’s unschedulable Pods. We will create a Deployment that requests more resources than our single node can provide.
```yaml
# scale-test.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: scale-test
  namespace: default
spec:
  replicas: 20
  selector:
    matchLabels: { app: scale-test }
  template:
    metadata:
      labels: { app: scale-test }
    spec:
      containers:
        - name: pause
          image: registry.k8s.io/pause:3.9
          resources:
            requests: { cpu: "200m", memory: "256Mi" }
            limits: { cpu: "200m", memory: "256Mi" }
```

- This manifest attempts to create 20 Pods, each requesting 200m CPU. Our t3.medium instance only has 2 vCPUs (2000m), so it can only fit a handful of these Pods. The rest will get stuck.

- Apply the test deployment:
```bash
kubectl apply -f ./scale-test.yaml
```
<img width="881" height="63" alt="Screenshot 2026-07-12 at 11 39 50 AM" src="https://github.com/user-attachments/assets/651d1abf-ffee-49b9-9d4b-cc89dfe29b71" />

- Confirm that some Pods are pending:
```bash
kubectl get pods -l app=scale-test
```

<img width="800" height="418" alt="Screenshot 2026-07-12 at 11 40 05 AM" src="https://github.com/user-attachments/assets/48df1cee-2467-47ec-a4ab-c84fa972ac0b" />

- You should see several Pods with the status pending. This is our trigger. Now, let’s watch the CA’s logs in real time to see its decision-making process.
```bash
# Tail the logs of the Cluster Autoscaler deployment
kubectl -n kube-system logs -f deploy/cluster-autoscaler-aws-cluster-autoscaler
```
- You are looking for key messages like the following:
  - Pod scale-test-xxxxxxxx-xxxxx is unschedulable
  - Scale-up: setting group managed-ng-1 size to 4
  - Expanding Node Group managed-ng-1 from 1 to 4

- While the logs are running, open a new terminal and watch the nodes join the cluster:
```bash
kubectl get nodes -w
```
<img width="731" height="527" alt="Screenshot 2026-07-12 at 11 50 08 AM" src="https://github.com/user-attachments/assets/5c7cbefc-38a0-4390-a563-d067a53c5bcf" />

- You will see new nodes appear and transition to the Ready state. Shortly after, the pending scale-test Pods will be scheduled on these new nodes. This completes the verification loop: we created demand, the autoscaler saw the need, it provisioned new supply (nodes), and Kubernetes scheduled the Pods.

- Finally, to test scale-down, simply remove the load:
```bash
kubectl delete -f ./scale-test.yaml
```

<img width="874" height="102" alt="Screenshot 2026-07-12 at 11 50 35 AM" src="https://github.com/user-attachments/assets/dface06f-caeb-4afb-bbe5-d30d98a02e2a" />

- After the scale-down-unneeded-time we set in the Helm chart (5 minutes), the CA will identify the now-empty nodes and issue commands to terminate them, scaling the group back down towards its minSize.

### Troubleshooting in the trenches: Quick hits for when things go wrong
- After deploying the CA, you might be asked, “Okay, you’ve deployed it, but it’s not scaling. What are the first three things you check?” This tests your diagnostic process. A great answer moves logically from the most likely cause to the more nuanced ones.

- Here are some common failure modes and how to address them.
  1. Problem: Scale-up isn’t happening despite pending Pods
     - This is often an issue with discovery. The CA simply can’t find the ASG it’s supposed to manage.
       - Verification: The link between the CA and an ASG is a set of specific tags. You need to confirm they are correctly applied to the ASG itself, not just the instances.
         1. Navigate to the EC2 Console in AWS.
         2. Under Auto Scaling, select Auto Scaling Groups.
         3. Find the ASG corresponding to your EKS node group.
         4. Select it and go to the Tags tab.
         5. Confirm these two tags exist and are spelled correctly:
            - k8s.io/cluster-autoscaler/enabled with a value of true.
            - k8s.io/cluster-autoscaler/interview-ca-cluster (or your cluster’s name) with a value of owned.
     - If these tags are missing or incorrect, the CA will ignore the node group completely.
       
   2. Problem: The CA Pod is crash-looping or its logs show AccessDenied errors
      - This is almost always an IRSA permissions problem. The Pod is trying to call AWS APIs (such as DescribeAutoScalingGroups) but is being told it doesn’t have the authority.
        - Verification: The magic link in the IRSA chain is the annotation on the Kubernetes ServiceAccount. This annotation tells the EKS control plane to swap the Pod’s default token for temporary IAM credentials.
        - Run this command to inspect the ServiceAccount:
          ```bash
          kubectl -n kube-system get sa cluster-autoscaler -o yaml
          ```
        - You must see an annotation that looks like this: eks.amazonaws.com/role-arn: arn:aws:iam::.... If that annotation is missing, it means you either forgot to create the IRSA mapping or, more likely, you let the Helm chart create a new, un-annotated ServiceAccount instead of using the one you prepared.

   3. Problem: Scale-up works, but scale-down never happens
      - This is a more subtle issue, as the autoscaler might be intentionally blocked from removing a node.
        - Verification: The CA won’t remove a node if it can’t safely evict all the Pods. Check for these common blockers:
          - minSize constraint: First, check the obvious: is minSize of your ASG set to a value that prevents further scale-down?
          - PodDisruptionBudgets (PDBs): A PDB can prevent the autoscaler from evicting a Pod if it would violate the budget (e.g., “always keep at least 3 replicas of this app running”). This is a legitimate safety feature, but it can pin a node.
          - “Unsafe to evict” Pods: The autoscaler respects certain Pods that it considers unsafe to evict. This includes Pods with local storage (emptyDir doesn’t count) or any Pod with the annotation cluster-autoscaler.kubernetes.io/safe-to-evict: "false". You can check for this by describing the Pods on the underutilized node

### Step 7: Cleaning up your environment
- A crucial part of any demonstration or project is responsible resource management. Leaving cloud resources running incurs costs, so always clean up when you’re done.
  - Uninstall the CA application: This command uses Helm to remove the Deployment, Services, and other Kubernetes resources associated with the CA.
  ```bash
  helm -n kube-system uninstall cluster-autoscaler || true
  ```

<img width="992" height="65" alt="Screenshot 2026-07-12 at 11 51 15 AM" src="https://github.com/user-attachments/assets/73474a22-fe1f-4f52-80c8-d8434f916a25" />

  - Delete the EKS cluster and nodes: This is the main cleanup command. It instructs eksctl to deprovision all AWS resources it created for the cluster, including the EKS control plane, the worker nodes in the ASG, IAM roles, and security groups.
  ```bash
  eksctl delete cluster --name "$CLUSTER_NAME" --region "$AWS_REGION"
  ```

<img width="1263" height="149" alt="Screenshot 2026-07-12 at 12 01 57 PM" src="https://github.com/user-attachments/assets/d6c48316-d2b8-4e4b-a372-ca5364c8786b" />

  - Remove the IAM policy (optional): After the cluster that used the policy is gone, it’s good practice to remove the now-orphaned IAM policy to keep your AWS account tidy.
  ```bash
  aws iam delete-policy --policy-arn "$CA_POLICY_ARN"
  ```

<img width="970" height="109" alt="Screenshot 2026-07-12 at 12 04 51 PM" src="https://github.com/user-attachments/assets/6127150f-b1e6-4113-a311-c0102aa0d30e" />

  ### From practice to theory: understanding the “why” behind the “how”
  - Navigating the detailed setup, deployment, and troubleshooting process reveals both the power and the inherent design philosophy of the CA. It is a robust, mature, and highly effective tool for managing capacity in a predictable way. You have proven you can configure it securely and make it work.
  - However, this hands-on success also highlights its specific mode of operation. It’s reactive, waiting for the Kubernetes scheduler to fail before it acts. Its actions are constrained by pre-defined groups (Auto Scaling Groups), which means you must decide on your instance types ahead of time. Finally, its speed is ultimately limited by how fast the cloud provider can provision and boot a new virtual machine.

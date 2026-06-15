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


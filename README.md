So, in our Parking Adapter application, users or client apps send HTTPS requests to our public API endpoint. These requests first go through AWS Route 53, which uses geo-based or latency-based routing to direct traffic to the nearest region—like Singapore or Mumbai for AMAP, Frankfurt or Ireland for EMEA, and Beijing for China.

When the request reaches the region, it hits an AWS Application Load Balancer, which then forwards it to our Kubernetes Ingress Controller—usually NGINX or AWS ALB Ingress. The Ingress Controller routes the request to the right microservice running inside our EKS cluster.

Our APIs are stateless and deployed as Docker containers. That means any pod can handle any request, which makes scaling really efficient. Kubernetes handles auto-scaling based on traffic, and we use Jenkins pipelines to manage rolling updates with zero downtime.

For authentication, we use OAuth2—especially when we need delegated access or integration with third-party identity providers. Depending on the flow, token validation happens either at the gateway or inside the service.

After authentication, the API interacts with backend services. For structured data, we use Amazon RDS, and for high-throughput NoSQL workloads, we use DynamoDB. Redis via ElastiCache handles caching—like session data or frequently accessed queries. We also integrate with third-party services like payment gateways and maps, and we’ve built in retry logic, circuit breakers, and fallback mechanisms to handle failures gracefully.

On the DevOps side, we use GitHub for source control, Jenkins for CI/CD, SonarQube for code quality, Docker for containerization, and Terraform for infrastructure provisioning. Jenkins pipelines trigger deployments and updates across regions.

For monitoring and observability, we rely on Datadog. It helps us track API latency, error rates, throughput, and resource usage. We also use distributed tracing to follow requests across services, which is super helpful for debugging and performance optimization.

And in case a region goes down, Route 53 automatically reroutes traffic to the next best region, so we maintain high availability globally. The whole setup is designed to be resilient, scalable, and secure."

undefined



the diagram representation of your Parking Adapter architecture:

User/Client → AWS Route 53 → AWS Application Load Balancer → Kubernetes Ingress Controller → Parking Adapter Pods
Authentication via OAuth2
Backend Services: Amazon RDS, DynamoDB, Redis/ElastiCache, and third-party integrations.





------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------

Cluster authentication



    

✅ Start with the context

"In our Parking Adapter application, we run multiple Kubernetes clusters across regions—AMAP, EMEA, and China. One of the biggest challenges was providing secure access to these clusters for developers and automation systems without relying on static credentials or insecure kubeconfig files."



✅ Explain the authentication flow

"Here’s how it works: When a user wants to interact with the cluster, they first authenticate with a trusted identity provider—like AWS Cognito, Okta, or Azure AD. This identity provider verifies the user and issues a signed token. Kubernetes API Server is configured to trust tokens from this provider using OIDC integration."

"The API Server validates the token and then checks RBAC (Role-Based Access Control) to determine what permissions the user has. For example, DevOps might have cluster-admin rights, while developers have namespace-level access. If authorized, the API Server allows the user to perform actions like deploying or scaling Parking Adapter pods."



✅ What is the Kubernetes API Server and RBAC?

"The API Server is the central control point for Kubernetes. All kubectl commands and API requests go through it. It handles authentication (who you are) and authorization (what you can do). RBAC is integrated with the API Server and uses RoleBindings or ClusterRoleBindings to map users or groups to permissions."



✅ How do Parking Adapter pods fit in?

"Pods don’t authenticate users directly. Authentication and authorization happen at the API Server level. Once the user is authorized, the API Server allows actions like creating or updating pods. The pods themselves just run workloads—they don’t validate tokens."



✅ Is kubeconfig required?

"Yes, kubeconfig is required because it tells kubectl how to connect to the cluster. But instead of storing static credentials, we use an exec plugin in kubeconfig that fetches a short-lived token dynamically. For example, the kubectl oidc-login plugin opens a browser, the user logs in via the identity provider, and the plugin retrieves a token. This token is then used for API calls."

"For automation like Jenkins pipelines, we use Kubernetes service accounts or identity-based tokens. The pipeline retrieves a short-lived token and applies manifests securely without storing static credentials."



✅ Benefits

"This approach gives us centralized identity management, MFA support, and short-lived tokens for security. It scales across multiple regions because the same identity provider can issue tokens for all clusters. Overall, it’s secure, developer-friendly, and compliant with enterprise standards."







------------------------------------------------------------------------------------------------------

how do you given an s3-access pod secure access



 

To give a D3-access pod secure access in Kubernetes, you need to follow a least privilege principle and use Kubernetes-native security features. Here’s how you can do it in a real-world scenario:



✅ Scenario

You have a pod that needs to access Amazon S3 (D3) or a secure external service. Instead of hardcoding credentials, you use IAM roles for service accounts and Kubernetes RBAC.



✅ Steps to Secure Access

1. Create a Kubernetes Service Account

This service account will be linked to the pod:







YAML

apiVersion: v1

kind: ServiceAccount

metadata:

name: d3-access-sa

Show more lines



2. Assign IAM Role to Service Account (IRSA)

On AWS EKS:

Create an IAM role with permissions for S3 (or D3).
Annotate the service account with the IAM role ARN:







YAML

apiVersion: v1

kind: ServiceAccount

metadata:

name: d3-access-sa

annotations:

eks.amazonaws.com/role-arn: arn:aws:iam::123456789012:role/D3AccessRole

``

Show more lines

This ensures the pod uses temporary credentials from AWS STS instead of static keys.



3. Bind Service Account to Pod

In your pod spec:







YAML

spec:

serviceAccountName: d3-access-sa

Show more lines



4. Apply RBAC for Kubernetes Resources

If the pod needs to access Kubernetes resources (like Secrets or ConfigMaps), create a Role and RoleBinding:







YAML

kind: Role

apiVersion: rbac.authorization.k8s.io/v1

metadata:

name: d3-access-role

rules:

- apiGroups: [""]

resources: ["secrets"]

verbs: ["get"]

Show more lines

Bind it:







YAML

kind: RoleBinding

apiVersion: rbac.authorization.k8s.io/v1

metadata:

name: d3-access-binding

subjects:

- kind: ServiceAccount

name: d3-access-sa

roleRef:

kind: Role

name: d3-access-role

apiGroup: rbac.authorization.k8s.io



Show more lines



✅ Security Best Practices

Use IRSA (IAM Roles for Service Accounts) for AWS access.
Avoid hardcoding credentials in pods.
Enable encryption at rest for secrets.
Restrict RBAC permissions to only what’s needed.
Rotate IAM credentials automatically (AWS STS does this).



🔥 Interview Answer (30 seconds):

“To give a pod secure access to D3 or S3, we use Kubernetes Service Accounts linked to IAM roles via IRSA. This avoids hardcoding credentials and uses temporary tokens. We also apply RBAC to restrict Kubernetes resource access and encrypt secrets at rest. This ensures least privilege and compliance.”





--------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------

suppose the Jenkins job is failed and how can we revert to the changes which has been moved to production





If a Jenkins job fails after deploying to production, we revert using Kubernetes rollout undo or redeploy the previous artifact from Nexus. We maintain versioned Docker images and Helm charts, so rollback is quick. For infrastructure changes, we use Terraform state to restore. In Blue-Green or Canary deployments, we simply switch traffic back to the stable environment





------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------

what all the metrics would be monitoring in datadog for containers, Kubernetes





In Datadog, we monitor container CPU, memory, network, and disk I/O. For Kubernetes, we track pod status, restart counts, resource requests vs limits, and node health metrics like CPU, memory, and disk. At the cluster level, we monitor API server latency, scheduler performance, and etcd health. These metrics help ensure stability and scalability.



-------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------

Reason for pod to pod communication failures

In our Kubernetes environment, the POS application pod is unable to establish connectivity with another application pod. This could indicate issues with inter-pod communication, such as missing or misconfigured Kubernetes Services, incorrect NetworkPolicies, DNS resolution failures, or pod-level resource constraints. We would start by checking if both pods are in the same namespace or if cross-namespace communication is allowed, verify that the target pod is exposed via a ClusterIP or headless service, and ensure there are no restrictive NetworkPolicies blocking traffic between them



--------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------let's talk about the scenario that disk space is high i cleaned up the disk an hour before and after one full i could see the disk uilization is more why all the possible there for this cause like real word issue



We had a case where disk usage spiked again after cleanup. It turned out Fluentd was failing to ship logs due to a misconfigured token, so it kept buffering logs locally. We also found orphaned Docker images and exited containers that weren’t garbage collected. We fixed it by rotating logs, pruning unused images, and setting up disk usage alerts via Prometheus

Log rotation is the process of archiving old log files and limiting their size so they don’t fill up the disk.

Rotate logs daily or when they exceed a size (e.g., 100MB)
Compress old logs
Delete logs older than a retention period

--------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------

DATA

Data "aws_vpc" "ccs" {

filter {

    name= "tag:name"

   value= "prod-vpc"

}



Resource "aws-subnet" "private" {

Vpc_id= data.aws_vpc.ccs.id





Output
Output "instance_public_ip"{
Value= aws_instance.web.public_ip
}
Local: Locals store values like strings, lists, numbers, or maps. These values can be used inside the module but cannot be changed from outside.

locals {
name   = "ccs"
 chandra = "sekhar"
 tags = {

   hello = "world"

   hi   = "devops"

 }

}



resource "aws_instance" "aws" {

 ami          = "ami-1234567890abcdef0" # Replace with a real AMI ID

 instance_type = "t2.micro"

 tags         = local.tags

}
-----------------------------------------------------------------------------------------------------------------------------------------------------------------------------
How can code be deployed to a Kubernetes cluster using Jenkins, and how does Jenkins determine where to deploy it



Prerequisites and cluster access

Cluster access: Ensure you can access each cluster locally with a kubeconfig.

RBAC (least privilege): Create a service account and role for CI deploys (per namespace).

Minimal kubeconfig: Generate a kubeconfig bound to the CI service account per cluster, so Jenkins doesn’t rely on your personal credentials.

       Get the service account token:

       Get the cluster CA data and server URL:

      Create a kubeconfig file (repeat per region, adjust name/server):

Configure Jenkins credentials for kubeconfig

Credential type: Use “Secret file” credentials to store kubeconfig files per region.
Jenkins steps (UI):

Dashboard → Manage Jenkins → Credentials → (select domain/global) → Add Credentials
Kind: Secret file
File: Upload the kubeconfig you created (e.g., emea-kubeconfig.yaml)
ID: Use a clear ID (e.g., kubeconfig-emea, kubeconfig-amap, kubeconfig-china)
Description: “Kubeconfig for EMEA CI deploy”
Job parameter: Create a “REGION” choice parameter with values AMAP, EMEA, CHINA.
Mapping credentials to regions: Use a simple map in the Jenkinsfile so the right kubeconfig is used automatically.



What to say in an interview

Credential security: “We store kubeconfigs as Secret file credentials in Jenkins, one per region. The pipeline selects the correct credential ID based on a REGION parameter.”
Least privilege: “Each kubeconfig uses a service account scoped by namespace and RBAC to only the resources we deploy.”
Multi‑region control: “Region‑specific values or manifests ensure we target AMAP/EMEA/CHINA correctly without changing code.”
Verification: “We use rollout status and pod/service checks post‑deploy; failures stop the pipeline and surface logs.”





-------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------

How can you deploy a Datadog Agent in a Kubernetes cluster using a manifest file, and how does it transmit metric data to Datadog



Get your Datadog API key

You need this to authenticate the agent with Datadog.

. Create a Kubernetes Secret

Store the API key securely in your cluster:

Deploy the Datadog Agent

Use a DaemonSet so one agent runs on each node.
You can use a manifest file or Helm chart.
The agent collects metrics, logs, and traces from containers and nodes.

Configure the Agent

Set environment variables like:

DD_API_KEY (from the secret)
DD_SITE (e.g., datadoghq.com)
DD_LOGS_ENABLED=true
DD_CONTAINER_COLLECT_ALL=true

--------------------------------------------------------------------------------------------------------------------------------------------------------------------------------

how can we go with automatic rollbacks if any issues have come in k8 deployment, take examples of prod or dev.





stage('Deploy to Dev') {

 steps {

   sh 'kubectl apply -f k8s/dev/deployment.yaml'

   sh 'kubectl rollout status deployment parking-adapter --namespace=dev'

 }

}



stage('Smoke Test') {

 steps {

   script {

     def result = sh(script: './scripts/smoke-test.sh', returnStatus: true)

     if (result != 0) {

       sh 'kubectl rollout undo deployment parking-adapter --namespace=dev'

       error("Smoke tests failed. Rolled back.")

     }

   }

 }

}





--------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------

suppose for the first time i am running the mvn pacakge build stage in Jenkins pipeline and parallels i am  running the sonar, how it is testing if any errors have been there in code before creating the artifacts, is it a better approach.





Running SonarQube in parallel with mvn package is not ideal. It’s better to run SonarQube after compilation and before artifact creation, so your pipeline enforces quality gates and prevents bad code from being packaged or deployed.

Running SonarQube in parallel with mvn package means you’re not gating artifact creation on code quality.
If SonarQube finds issues, your artifact may already be built and pushed to Nexus, which defeats the purpose of quality checks.
The safer approach is sequential execution: build → analyze → package → deploy.



stage('Build') {

   steps {

       sh 'mvn clean verify'

   }

}

stage('SonarQube Analysis') {

   steps {

       sh 'mvn sonar:sonar'

   }

}

stage('Quality Gate') {

   steps {

       timeout(time: 5, unit: 'MINUTES') {

           waitForQualityGate abortPipeline: true

       }

   }

}

stage('Package & Publish') {

   steps {

       sh 'mvn package'

       // upload to Nexus

   }

}



---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------

Horizontal Pod Autoscaler



The Horizontal Pod Autoscaler (HPA) in Kubernetes is a built-in feature that automatically adjusts the number of pods in a workload based on observed metrics such as CPU, memory, or even custom/external metrics. The main reason we use HPA is to ensure applications can handle varying traffic loads efficiently—scaling up when demand increases and scaling down when demand is low, which improves performance and optimizes cost.

Internally, HPA works by querying metrics from the metrics-server, which collects CPU and memory usage from nodes and pods. It then compares the current usage against the target values defined in the HPA spec. Based on this comparison, it updates the replica count in the associated Deployment, ReplicaSet, or StatefulSet. Importantly, HPA doesn’t modify pods directly—it only changes the replica count, and the controller creates or deletes pods accordingly.

By default, HPA supports CPU and memory metrics, but with custom setups (like Prometheus + Prometheus Adapter), you can scale on application-specific metrics (e.g., queue length) or even external metrics such as cloud service metrics. For example, resource metrics are CPU/memory, custom metrics come from inside applications, and external metrics originate outside the cluster.

The scaling decision is calculated using a formula that compares current usage with the target. For CPU, you can define scaling based on utilization percentage (relative to CPU requests) or an absolute average value (like 200m CPU). However, if CPU requests are not set on pods, HPA cannot calculate utilization correctly, leading to inaccurate or failed scaling.

HPA checks metrics roughly every 15 seconds, but to prevent unstable or rapid scaling, it uses stabilization windows and scaling policies. These are configurable via the behavior field, which lets you control how fast scaling up or down can happen. This is one of the key differences between HPA v1 and v2—v2 introduced support for multiple metrics, custom/external metrics, and advanced scaling behavior, whereas v1 only supported basic CPU scaling.

One limitation is that HPA cannot scale workloads down to zero replicas by default. For scale-to-zero scenarios, tools like KEDA are used. Also, if the metrics-server is down, HPA cannot retrieve metrics and scaling stops, with errors logged in events.

HPA also interacts with other autoscaling components. For example, if HPA increases pod replicas but the cluster lacks sufficient nodes, the Cluster Autoscaler will provision additional nodes. On the other hand, combining HPA with Vertical Pod Autoscaler (VPA) requires caution—both should not adjust CPU/memory simultaneously. A common practice is to use HPA for horizontal scaling and VPA only for recommendations or controlled updates.



--------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------



Your pod keeps getting stuck in CrashLoopBackOff, but logs show no errors. How would you approach debugging and resolution



Check Pod Events

kubectl describe pod <pod-name> -n <namespace>

Inspect Container Exit Codes

Even if logs are empty, exit codes tell you why the container stopped.

kubectl get pod <pod-name> -n <namespace> -o jsonpath='{.status.containerStatuses[*].lastState.terminated.exitCode}'

Check Resource Requests & Limits

kubectl describe pod <pod-name> | grep -A5 "Limits"

Validate Liveness/Readiness Probes

kubectl describe pod <pod-name> | grep -A5 "Liveness"

Run Pod in Debug Mode

kubectl run debug --rm -it --image=<image> -- bash

kubectl exec -it <pod-name> -n <namespace> -- /bin/bash

Check Node & Cluster Health

kubectl get nodes

kubectl describe node <node-name>

Inspect Deployment/ReplicaSet

kubectl get deployment <deployment-name> -n <namespace> -o yaml





----------------------------------------------------------------------------------------------------------------------------------------------------------------

You have a StatefulSet deployed with persistent volumes, and one of the pods is not recreating properly after deletion. What could be the reasons, and how do you fix it without data loss





I check the pod events with kubectl describe pod, inspect the PVC with kubectl describe pvc, confirm the PV and StorageClass with kubectl describe pv, and verify the StatefulSet spec. If the PVC is intact, I just delete the pod and let Kubernetes recreate it. If the PVC is stuck, I remind the PV manually. If the PV was deleted, I restore from backup. This way, I ensure the pod comes back online without losing data.”



Check the Pod Events

I would run:



kubectl describe pod parking-adapter-0 -n <namespace>

This shows me the events. If I see messages like FailedAttachVolume or PersistentVolumeClaim not found, it means the pod cannot attach its volume.

2. Inspect the PVC

I would run:

kubectl get pvc -n <namespace>
kubectl describe pvc data-parking-adapter-0 -n <namespace>

This tells me if the PVC is still Bound to a PV. If it’s in Released or Lost state, the pod won’t recreate.

3. Check the PV and StorageClass

I would run:

bash

kubectl get pv
kubectl describe pv <pv-name>
kubectl get storageclass

This helps me confirm whether the PV is available and what the reclaim policy is (Retain or Delete). If the PV was deleted due to a Delete policy, I know I’ll need to restore from backup.

4. Verify the StatefulSet Spec

I would run:

bash

kubectl get statefulset parking-adapter -n <namespace> -o yaml

This lets me check that the volumeClaimTemplates are correct and match the PVCs.

5. Check Node Scheduling

I would run:

bash

kubectl describe node <node-name>

This shows me if the node has taints, disk pressure, or if the volume cannot attach because of zone restrictions (common with AWS EBS volumes).

✅ Fix Without Data Loss

If the PVC is Bound correctly: I would simply delete the pod and let Kubernetes recreate it:
bash
kubectl delete pod parking-adapter-0 -n <namespace>
If the PVC is Released but the PV still exists: I would edit the PV to rebind it to the PVC:
bash
kubectl edit pv <pv-name>

Then update the claimRef to point to data-parking-adapter-0.
If the PV was deleted due to reclaim policy: I would restore the volume from an AWS EBS snapshot or backup, then recreate the PVC and reattach it.
If it’s a scheduling issue: I would check the pod’s node affinity with:
bash
kubectl describe pod parking-adapter-0

Then fix the node affinity or reschedule the pod to a node in the correct availability zone.



--------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------

Your cluster autoscaler is not scaling up even though pods are in Pending state. What would you investigate



Check Pending Pods

First, I would confirm why the pods are pending:

bash

kubectl get pods -n <namespace>
kubectl describe pod <pod-name> -n <namespace>

👉 Look for events like 0/10 nodes are available, Insufficient CPU, Insufficient memory, or node affinity/taints.

2. Check Node Group Capacity

If pods need more resources, the autoscaler should add nodes. I would check the node groups:

bash

kubectl get nodes
kubectl describe node <node-name>

👉 Verify if nodes are at capacity or tainted, preventing scheduling.

3. Check Cluster Autoscaler Logs

The autoscaler runs as a deployment in kube-system. I would check its logs:

bash

kubectl logs -n kube-system deployment/cluster-autoscaler

👉 Look for messages like max node group size reached, not enough resources, or skipping scale up.

4. Verify Resource Requests

Sometimes pods request more resources than any node type can provide. I would check:

bash

kubectl describe pod <pod-name> | grep -A5 "Requests"

👉 If a pod requests 64GB memory but nodes only have 32GB, autoscaler cannot help.

5. Check Node Group Limits in AWS

Cluster Autoscaler respects min/max node group sizes. I would check the AWS Auto Scaling Group or Managed Node Group:

bash

aws autoscaling describe-auto-scaling-groups --auto-scaling-group-names <asg-name>

👉 Ensure the max size is not already reached.

6. Check IAM Permissions

Cluster Autoscaler needs IAM permissions to modify node groups. I would confirm the IAM role attached to the autoscaler pod has:

autoscaling:DescribeAutoScalingGroups
autoscaling:SetDesiredCapacity
autoscaling:TerminateInstanceInAutoScalingGroup

7. Check Scheduling Constraints

If pods have node affinity, tolerations, or specific labels, autoscaler may not find a matching node type:

bash

kubectl describe pod <pod-name> | grep -A5 "Affinity"

👉 If affinity rules are too strict, autoscaler cannot scale.





-------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------

A network policy is blocking traffic between services in different namespaces. How would you design and debug the policy to allow only specific communication paths?



NetworkPolicies in Kubernetes are namespace‑scoped and by default deny traffic unless explicitly allowed. If services in different namespaces cannot communicate, I would debug the existing policies, verify pod labels, and then design a policy that allows only the specific communication path required. My approach is systematic: check what’s blocking, test connectivity, and then refine the policy.”

🔍 Step‑by‑Step Debugging

1. Check Existing Network Policies

I would start by listing and describing the policies in both namespaces:

bash

kubectl get networkpolicy -n amap
kubectl get networkpolicy -n emea
kubectl describe networkpolicy <policy-name> -n emea

👉 This shows me which pods are selected and what ingress/egress rules are defined.

2. Verify Pod Labels

NetworkPolicies rely on pod labels. I would check:

bash

kubectl get pods -n amap --show-labels
kubectl get pods -n emea --show-labels

👉 If labels don’t match the selectors in the policy, traffic may be unintentionally blocked.

3. Test Connectivity

I would run a temporary busybox pod in the source namespace to test communication:

bash

kubectl run test-pod -n amap --rm -it --image=busybox -- sh

Then try:

bash

wget http://<service-name>.<namespace>.svc.cluster.local:<port>

👉 This confirms whether traffic is blocked at the network layer.

4. Check Cross‑Namespace Rules

By default, policies only apply within a namespace. To allow traffic from another namespace, I must use a namespaceSelector:

yaml

ingress:
- from:
  - namespaceSelector:
      matchLabels:
        region: amap

👉 This allows pods in the amap namespace to talk to pods selected by this policy.

🛠 Designing the Correct Policy

Suppose I want to allow only parking-adapter pods in AMAP namespace to talk to database pods in EMEA namespace on port 5432. I would create this policy in the EMEA namespace:

yaml

apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-amap-to-db
  namespace: emea
spec:
  podSelector:
    matchLabels:
      app: database
  ingress:
  - from:
    - namespaceSelector:
        matchLabels:
          region: amap
    - podSelector:
        matchLabels:
          app: parking-adapter
    ports:
    - protocol: TCP
      port: 5432

👉 This ensures only pods labeled app=parking-adapter in the amap namespace can connect to the database pods in emea namespace.



-------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------

You’re running a multi-tenant platform on a single EKS cluster. How do you isolate workloads and ensure security, quotas, and observability for each tenant



I isolate tenants by namespaces, NetworkPolicies, RBAC, IRSA, and (when needed) dedicated node groups.
I enforce fairness with ResourceQuota, LimitRange, PriorityClasses, and PDBs for HA.
I harden workloads via PodSecurity standards, policy engines, and least‑privilege IAM.
I deliver per‑tenant observability with labeled metrics/logs/traces and scoped dashboards/alerts.
I validate routinely with reachability tests, quota probes, and RBAC checks, and automate guardrails with Gatekeeper/Kyverno.

This blueprint scales across AMAP, EMEA, and CHINA tenants in one EKS cluster while preserving security, isolation, and operational clarity.



--------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------

You notice the kubelet is constantly restarting on a particular node. What steps would you take to isolate the issue and ensure node stability



I start by checking node status with kubectl describe node, then inspect kubelet logs via journalctl -u kubelet. I validate system resources, kubelet configuration, and container runtime health. If the node is unstable, I cordon and drain it to protect workloads. Depending on the root cause — misconfiguration, resource exhaustion, or OS issues — I fix the kubelet config, adjust resources, renew certificates, or replace the node. This ensures node stability and keeps the cluster healthy.





------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------

A critical pod in production gets evicted due to node pressure. How would you prevent this from happening again, and how do QoS classes play a role



I prevent critical pods from eviction by configuring them with proper requests and limits so they fall into the Guaranteed QoS class. I also use PriorityClasses, PodDisruptionBudgets, and autoscaling to protect them. QoS classes matter because Kubernetes evicts pods in order: BestEffort first, then Burstable, and Guaranteed last. By making critical pods Guaranteed, I ensure they are the most protected under node pressure.

---------------------------------------------------------------------------------------------------------------------------------------------------------------------------
What is terraform taint?
	• terraform taint marks a resource in the Terraform state file as tainted.
	• A tainted resource will be destroyed and recreated during the next terraform apply.
	• It’s useful when:
		○ A resource is corrupted or misconfigured outside Terraform.
		○ You want to force a fresh deployment without changing the configuration.

          Example: terraform taint aws_instance.my_ec2

What is terraform untaint?
	• terraform untaint removes the taint mark from a resource.
	• It ensures the resource will not be destroyed/recreated in the next apply.
	• Useful when:
		○ You mistakenly tainted a resource.
		○ You decide the resource doesn’t need replacement.

Example: terraform untaint aws_instance.my_ec2<img width="899" height="460" alt="image" src="https://github.com/user-attachments/assets/ef1d5316-9aa3-45fa-8df9-6a78356a14bf" />


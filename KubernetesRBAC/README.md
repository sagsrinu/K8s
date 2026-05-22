Kubernetes RBAC
Role-Based Access Control on Amazon EKS


Overview
Kubernetes RBAC (Role-Based Access Control) is the mechanism that controls who can perform which actions on which resources inside a cluster. On Amazon EKS, RBAC works in tandem with AWS IAM: IAM handles authentication (who you are), while Kubernetes RBAC handles authorization (what you are allowed to do).

Authentication & Authorization Flow
Every kubectl command follows this pipeline before a response is returned:

Step	Component	Description
1	User	Runs kubectl command (e.g., kubectl get nodes)
2	kubeconfig	Reads cluster connection details and credentials
3	aws eks get-token	Fetches a short-lived token from AWS
4	IAM Authentication	AWS verifies the IAM identity via STS
5	aws-auth ConfigMap	Maps the IAM identity to a Kubernetes user/group
6	Kubernetes User / Group	The resolved identity used for RBAC evaluation
7	RBAC Authorization	Kubernetes checks Role/ClusterRole permissions
8	API Server Response	Returns result: allowed or Forbidden error

Note: The aws-auth ConfigMap is the bridge between AWS IAM identities and Kubernetes RBAC subjects. Without a matching entry, even a valid IAM user will receive a Forbidden error.

Step-by-Step RBAC Setup
Follow these steps in order to configure RBAC access for a new IAM user:

1.	Create an IAM user and attach the required EKS cluster permissions in the AWS console.
2.	Configure the AWS CLI profile for the new user: aws configure --profile <username>
3.	Create a Kubernetes Role (namespace-scoped) or ClusterRole (cluster-wide) that defines the permitted actions.
4.	Create a RoleBinding or ClusterRoleBinding to bind the Role to a Kubernetes user or group.
5.	Edit the aws-auth ConfigMap to map the IAM user ARN to the Kubernetes user/group.

 
Example 1 — Namespace-Scoped Access via Group
In this example, an IAM user is mapped to a Kubernetes group in the aws-auth ConfigMap, and that group is bound to a namespace-scoped Role via a RoleBinding.

Step 1: Create IAM User
6.	Create a user named dev in the AWS IAM console.
7.	Under Security credentials, create an access key for CLI use.
8.	Copy the access key, secret access key, and the user ARN.

Step 2: Configure AWS CLI Profile
aws configure --profile dev

AWS Access Key ID [None]: <your-access-key>
AWS Secret Access Key [None]: <your-secret-key>
Default region name [None]: us-east-1
Default output format [None]: json

Verify the profile was created:
aws configure list-profiles

Step 3: Create a Kubernetes Role
A Role defines what actions are permitted on which resources within a specific namespace. Use ClusterRole instead if cross-namespace or cluster-wide access is needed.

role.yml
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  namespace: default
  name: developer-role
rules:
  - apiGroups: [""]          # Core API group (pods, configmaps, etc.)
    resources: ["configmaps"]
    verbs: ["get", "list"]
  - apiGroups: [""]
    resources: ["pods"]
    verbs: ["get", "list", "create", "delete"]
  - apiGroups: ["apps"]      # Apps API group (deployments, etc.)
    resources: ["deployments"]
    verbs: ["get", "list"]

kubectl apply -f role.yml

Step 4: Create a RoleBinding (Group-based)
A RoleBinding assigns a Role to a Kubernetes subject (user, group, or service account). Here the developer-role is bound to the developer group.

rolebinding.yml
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: developer-rolebinding
  namespace: default
subjects:
  - kind: Group
    name: "developer"         # Must match the group name in aws-auth
    apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: Role
  name: developer-role
  apiGroup: rbac.authorization.k8s.io

kubectl apply -f rolebinding.yml

Step 5: Update the aws-auth ConfigMap
Map the IAM user ARN to the Kubernetes username and group by editing the aws-auth ConfigMap:

kubectl edit cm aws-auth -n kube-system

Add the following under mapUsers:
mapUsers: |
  - userarn: arn:aws:iam::<account-id>:user/dev
    username: dev
    groups:
      - developer

Note: To grant the same permissions to another user, simply add a new entry under mapUsers with the same group name. No additional Role or RoleBinding is required.

Step 6: Log In as the dev User
aws eks update-kubeconfig --region us-east-1 --name <cluster-name> --profile dev

Step 7: Create a Pod (pod.yml)
Use the following manifest to create a test pod:

pod.yml
apiVersion: v1
kind: Pod
metadata:
  name: nginx-pod
  labels:
    app: nginx
spec:
  containers:
    - name: nginx-container
      image: nginx:latest
      ports:
        - containerPort: 80

kubectl apply -f pod.yml

Step 8: Verify Permissions
Test each operation to confirm the Role is enforced correctly:

List pods — should succeed (get/list allowed):
kubectl get pods

# Expected output:
No resources found in default namespace.

Access the pod — should succeed (get allowed):
kubectl get pod nginx-pod

Delete the pod — should succeed (delete allowed):
kubectl delete pod nginx-pod

List nodes — should FAIL (nodes are cluster-scoped, not in Role):
kubectl get nodes

# Expected error:
Error from server (Forbidden): nodes is forbidden: User "dev" cannot list resource
"nodes" in API group "" at the cluster scope

Delete pod without delete permission (if delete verb is removed from Role):
kubectl delete pod nginx-pod

# Expected error:
Error from server (Forbidden): pods "nginx-pod" is forbidden: User "dev" cannot
delete resource "pods" in API group "" in the namespace "default"

 
Example 2 — Direct User Binding (Without Group)
Instead of mapping a user to a group, you can bind a Role directly to a Kubernetes username. This is simpler but less scalable — each new user requires a separate RoleBinding.

RoleBinding (User-based)
rolebinding-user.yml
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: developer-rolebinding
  namespace: default
subjects:
  - kind: User
    name: dev                  # Must match the username in aws-auth
    apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: Role
  name: developer-role
  apiGroup: rbac.authorization.k8s.io

aws-auth ConfigMap (Without Group)
mapUsers: |
  - userarn: arn:aws:iam::<account-id>:user/dev
    username: dev
    # No groups needed — RoleBinding references the username directly

Note: The group-based approach (Example 1) is preferred in production because adding a new user only requires an aws-auth entry — not a new RoleBinding.

Example 3 — Access Restricted to a Specific Resource
You can restrict a Role to a named resource instance using the resourceNames field. This is useful when a user should only access one specific pod or secret, not all resources of that type.

apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: pod-read-specific
  namespace: default
rules:
  - apiGroups: [""]
    resources: ["pods"]
    resourceNames:            # Restricts access to this pod only
      - my-pod
    verbs:
      - get
      - list

Note: resourceNames cannot be used with verbs like create, because the resource does not exist at request time and has no name to match against.

 
ClusterRole & ClusterRoleBinding
A ClusterRole and ClusterRoleBinding grant permissions that apply across all namespaces (or to cluster-scoped resources such as nodes and persistent volumes). Use these when namespace-scoped Roles are not sufficient.

Step 1: Create IAM User for Cluster-Wide Access
9.	Create a user named prod in the AWS IAM console.
10.	Create an access key for CLI use and note the ARN.
11.	Configure the AWS CLI profile:
aws configure --profile prod

AWS Access Key ID [None]: <your-access-key>
AWS Secret Access Key [None]: <your-secret-key>
Default region name [None]: us-east-1
Default output format [None]: json

Step 2: Create a ClusterRole
clusterrole.yml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: cluster-admin-custom
rules:
  - apiGroups: ["*"]         # All API groups
    resources: ["*"]         # All resource types
    verbs: ["*"]             # All actions (get, list, create, update, delete, etc.)

Step 3: Create a ClusterRoleBinding
clusterrolebinding.yml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: cluster-admin-custom-binding
subjects:
  - kind: Group
    name: admin-team          # Must match the group in aws-auth
    apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: ClusterRole
  name: cluster-admin-custom
  apiGroup: rbac.authorization.k8s.io

kubectl apply -f clusterrole.yml
kubectl apply -f clusterrolebinding.yml

Step 4: Update aws-auth ConfigMap
kubectl edit cm aws-auth -n kube-system

mapUsers: |
  - userarn: arn:aws:iam::<account-id>:user/prod
    username: prod
    groups:
      - admin-team

Alternative: Full Admin Access via system:masters
For trusted administrators, you can bypass custom ClusterRoles entirely by assigning the built-in system:masters group, which grants unrestricted cluster access:

mapUsers: |
  - userarn: arn:aws:iam::<account-id>:user/prod
    username: prod
    groups:
      - system:masters        # Full cluster admin — use with caution

Note: Only assign system:masters to fully trusted administrators. It bypasses all RBAC checks and cannot be restricted further.

Step 5: Log In as the prod User
aws eks update-kubeconfig --region us-east-1 --name <cluster-name> --profile prod

 
Quick Reference — Role vs ClusterRole

	Role	ClusterRole	Notes
Scope	Single namespace	All namespaces + cluster-scoped resources	
Bound with	RoleBinding	ClusterRoleBinding (or RoleBinding for ns-scoped use)	ClusterRole + RoleBinding = namespace-scoped
Use case	Dev/test namespace isolation	Admin, monitoring, cross-namespace tools	
Node access	No	Yes	Nodes are cluster-scoped
PV access	No	Yes	PersistentVolumes are cluster-scoped

Common Errors & Fixes

Error: Forbidden — cannot list nodes
Error from server (Forbidden): nodes is forbidden: User "dev" cannot list resource
"nodes" in API group "" at the cluster scope
Cause: The user's Role only covers namespace-scoped resources. Nodes are cluster-scoped.
Fix: Use a ClusterRole that includes nodes in its resources, or use system:masters for admins.

Error: Forbidden — cannot delete pods
Error from server (Forbidden): pods "nginx-pod" is forbidden: User "dev" cannot delete
resource "pods" in API group "" in the namespace "default"
Cause: The Role does not include the delete verb for pods.
Fix: Add "delete" to the verbs list in the Role definition and re-apply.

Error: User not found / Unauthorized
Cause: The IAM user ARN is missing or incorrect in the aws-auth ConfigMap.
Fix: Run kubectl edit cm aws-auth -n kube-system and verify the userarn, username, and groups fields.


Kubernetes RBAC Reference Guide

# RBAC EKS IAM Users

This repository demonstrates Role-Based Access Control (RBAC) integration with AWS IAM users in Amazon EKS (Elastic Kubernetes Service). It shows how to configure different levels of permissions for various IAM users to interact with Kubernetes resources.

## Table of Contents
- [Overview](#overview)
- [Architecture](#architecture)
- [Components](#components)
- [IAM User Interaction Flow](#iam-user-interaction-flow)
- [File Structure](#file-structure)
- [Commands and Setup](#commands-and-setup)
- [Attribute Explanations](#attribute-explanations)

---

## Overview

This project demonstrates how to:
1. Map AWS IAM users to Kubernetes identities
2. Create RBAC ClusterRoles with specific permissions
3. Bind IAM users to ClusterRoles via ClusterRoleBindings
4. Manage different access levels (Trainee, Developer, SuperAdmin)

---

## Architecture

```
┌─────────────────────────────────────────────────────────────┐
│                     AWS IAM Users                             │
│   (eks-user, eks-developer, eks-super-admin)                 │
└────────────────────┬────────────────────────────────────────┘
                     │ AWS STS Authentication
                     ▼
┌─────────────────────────────────────────────────────────────┐
│                    aws-auth ConfigMap                        │
│          (Maps IAM Users to Kubernetes Users)                │
│              mapUsers, mapRoles, mapAccounts                 │
└────────────────────┬────────────────────────────────────────┘
                     │ User Identity Resolution
                     ▼
┌─────────────────────────────────────────────────────────────┐
│              ClusterRoleBinding                              │
│    (Links Kubernetes User to ClusterRole)                    │
└────────────────────┬────────────────────────────────────────┘
                     │ Permission Assignment
                     ▼
┌─────────────────────────────────────────────────────────────┐
│                   ClusterRole                                │
│    (Defines permissions: verbs, resources, apiGroups)        │
└────────────────────┬────────────────────────────────────────┘
                     │ Authorization Decision
                     ▼
┌─────────────────────────────────────────────────────────────┐
│          Kubernetes Resources Access                         │
│  (Pods, Deployments, Secrets, Services, etc.)               │
└─────────────────────────────────────────────────────────────┘
```

---

## Components

### 1. **aws-auth.yml**
The bridge between AWS IAM and Kubernetes RBAC. This ConfigMap sits in the `kube-system` namespace and authenticates IAM users/roles.

### 2. **Trainee ClusterRole** (trainee-clusterrole.yml)
Read-only access for training/learning purposes.

### 3. **Developer ClusterRole** (developer-clusterrole.yml)
Create, read, update, and patch permissions for application deployment.

### 4. **SuperAdmin ClusterRole** (superadmin-clusterrole.yml)
Full administrative access to all resources.

---

## IAM User Interaction Flow

### Step 1: IAM User Authentication
```
IAM User (eks-developer) 
    ↓
Calls: aws sts get-caller-identity
    ↓
AWS verifies IAM credentials
    ↓
Returns: User ARN (arn:aws:iam::154292417400:user/eks-developer)
```

### Step 2: AWS-Auth ConfigMap Lookup
When a user runs a kubectl command:
```
kubectl apply -f secret.yml
    ↓
EKS API Server checks aws-auth ConfigMap
    ↓
Matches IAM user ARN: arn:aws:iam::154292417400:user/eks-developer
    ↓
Maps to Kubernetes username: eks-developer
    ↓
Identifies groups: [developer-clusterrole]
```

### Step 3: ClusterRoleBinding Resolution
```
kubernetes username: eks-developer
    ↓
ClusterRoleBinding (developer-clusterrolebinding) matches subject
    ↓
Retrieves roleRef: developer-clusterrole
    ↓
Loads permissions from ClusterRole
```

### Step 4: Permission Evaluation
```
ClusterRole: developer-clusterrole
    ↓
Checks if verb (get, create, patch) is allowed on resource (secrets)
    ↓
Checks if apiGroup is allowed ("" for core API)
    ↓
Decision: Allow/Deny
    ↓
Either executes kubectl command or returns Forbidden error
```

---

## File Structure

```
RBAC-EKS-IAMUSERS/
├── aws_auth.yml                      # ConfigMap mapping IAM users to K8s users
├── trainee-clusterrole.yml           # Read-only ClusterRole
├── developer-clusterrole.yml         # Developer ClusterRole with create/update
├── superadmin-clusterrole.yml        # Full admin access
└── README.md                         # This file
```

---

## Commands and Setup

### 1. **Get AWS Caller Identity** (Verify IAM User)
```bash
aws sts get-caller-identity
```
**Output:**
```json
{
    "UserId": "AIDASH3ELON4BMUPRL25N",
    "Account": "154292417400",
    "Arn": "arn:aws:iam::154292417400:user/eks-developer"
}
```
**Purpose:** Confirms which IAM user's credentials are currently active. Shows the AWS Account ID and User ARN needed to verify the correct credentials are loaded.

---

### 2. **Generate/Update Kubeconfig**
```bash
eksctl utils write-kubeconfig --cluster=eksdemo1 --set-kubeconfig-context=true
```
**Output:**
```
[✔]  saved kubeconfig as "/Users/Paras_Kamboj/.kube/config"
```
**Purpose:** Generates kubeconfig file with authentication tokens for the current IAM user. This file enables kubectl to authenticate against the EKS cluster using AWS SigV4 signing.

**What it does:**
- Queries EKS cluster details (API endpoint, CA certificate)
- Creates/updates kubeconfig with a user entry that uses `aws eks get-token` command
- Sets the current context to use this new user
- kubectl will automatically sign API requests with the current AWS credentials

---

### 3. **Check Current Kubernetes Context**
```bash
kubectl config get-contexts
```
**Output:**
```
CURRENT   NAME                                            CLUSTER                 AUTHINFO                        NAMESPACE
*         eks-developer@eksdemo1.us-east-1.eksctl.io     eksdemo1.us-east-1      eks-developer@eksdemo1...      default
```
**Purpose:** Verifies which Kubernetes context (and thus which IAM user) is currently being used for kubectl commands.

---

### 4. **Apply RBAC Configuration to Cluster**
```bash
# Apply trainee-clusterrole and its binding
kubectl apply -f trainee-clusterrole.yml

# Apply developer-clusterrole and its binding
kubectl apply -f developer-clusterrole.yml

# Apply superadmin-clusterrole and its binding
kubectl apply -f superadmin-clusterrole.yml

# Update aws-auth ConfigMap
kubectl apply -f aws_auth.yml
```
**Purpose:** Deploys the RBAC configuration to the EKS cluster. This must be done with admin credentials (or the account root user) before IAM users can access resources.

**Why important:**
- RBAC definitions must exist in the cluster before they can be referenced
- aws-auth ConfigMap must be updated to recognize the IAM users and map them to ClusterRoles
- Changes take effect immediately once applied

---

### 4.1. **⚠️ CRITICAL: Export Current aws-auth ConfigMap from Cluster**
```bash
kubectl get configmap aws-auth -n kube-system -o yaml > aws_auth.yml
```
**Output:**
```yaml
apiVersion: v1
data:
  mapRoles: |
    - rolearn: arn:aws:iam::154292417400:role/eksctl-eksdemo1-nodegroup-ng-d10da-NodeInstanceRole-zOsFnaDThoyG
      groups:
      - system:bootstrappers
      - system:nodes
      username: system:node:{{EC2PrivateDNSName}}
  mapUsers: |
    - userarn: arn:aws:iam::154292417400:user/eks-user
      username: eks-user
      groups:
      - trainee-clusterrole
kind: ConfigMap
metadata:
  name: aws-auth
  namespace: kube-system
```

**Purpose:** Exports the current aws-auth ConfigMap from the running EKS cluster to a YAML file for version control and backup.

**Why this is CRITICAL:**
- **Preserves existing configuration:** The aws-auth ConfigMap contains node IAM role mappings that are essential for the cluster to function. Losing this can break node communication.
- **Enables version control:** Keeps a Git-tracked copy of your IAM-to-Kubernetes user mappings for auditing and disaster recovery.
- **Prevents accidental deletion:** If you recreate aws_auth.yml from scratch, you might forget existing roles/users and break the cluster.
- **Merger tool for updates:** When adding new IAM users, export first, then manually merge your changes instead of overwriting.

**Important workflow:**
1. Always export the current aws-auth ConfigMap before modifying it
2. Compare your changes: `diff aws_auth_backup.yml aws_auth.yml`
3. Apply changes carefully: `kubectl apply -f aws_auth.yml`
4. Verify with: `kubectl get configmap aws-auth -n kube-system -o yaml`

---

### 5. **Deploy Kubernetes Resources**
```bash
# Deploy a secret
kubectl apply -f secret.yml

# Deploy an application
kubectl apply -f deployment.yml

# View pods
kubectl get pods
```
**Purpose:** Once RBAC is configured and the user has appropriate permissions, IAM users can deploy and manage Kubernetes resources according to their ClusterRole permissions.

---

### 6. **Verify Permissions**
```bash
# Check if current user can get secrets
kubectl auth can-i get secrets

# Check if current user can create deployments
kubectl auth can-i create deployments

# Check if another user can do something (requires admin)
kubectl auth can-i get secrets --as=eks-developer
```
**Purpose:** Tests RBAC permissions before attempting operations. Returns `yes` or `no`.

---

### 7. **View RBAC Resources**
```bash
# List ClusterRoles
kubectl get clusterroles

# List ClusterRoleBindings
kubectl get clusterrolebindings

# View specific ClusterRole details
kubectl describe clusterrole developer-clusterrole

# View aws-auth ConfigMap
kubectl get configmap aws-auth -n kube-system -o yaml
```
**Purpose:** Inspects RBAC configuration and troubleshoots permission issues.

---

## Attribute Explanations

### aws-auth.yml Attributes

#### **mapUsers Section**
```yaml
mapUsers: |
  - userarn: arn:aws:iam::154292417400:user/eks-developer
    username: eks-developer
    groups:
      - developer-clusterrole
```

| Attribute | Purpose | Example |
|-----------|---------|---------|
| `userarn` | **Full AWS IAM user ARN.** This uniquely identifies the IAM user in AWS. The EKS API server uses this to authenticate requests. Format: `arn:aws:iam::ACCOUNT-ID:user/USERNAME` | `arn:aws:iam::154292417400:user/eks-developer` |
| `username` | **Kubernetes username.** The identity this IAM user will have inside the Kubernetes cluster. This is used by Kubernetes RBAC to match ClusterRoleBindings. | `eks-developer` |
| `groups` | **Kubernetes groups/identities.** List of group identities this IAM user belongs to. These must match the ClusterRoleBinding subject names. Convention: typically the ClusterRole name. | `developer-clusterrole` |

**How it works:** 
1. IAM user `eks-developer` authenticates with AWS credentials
2. aws-auth ConfigMap matches the user ARN: `arn:aws:iam::154292417400:user/eks-developer`
3. Maps them to Kubernetes username: `eks-developer`
4. Assigns them to group: `developer-clusterrole`
5. ClusterRoleBinding looks for subjects matching username `eks-developer`
6. Grants permissions from the `developer-clusterrole` ClusterRole

---

#### **mapRoles Section**
```yaml
mapRoles: |
  - rolearn: arn:aws:iam::154292417400:role/eksctl-eksdemo1-nodegroup-ng-d10da-NodeInstanceRole-zOsFnaDThoyG
    groups:
      - system:bootstrappers
      - system:nodes
    username: system:node:{{EC2PrivateDNSName}}
```

| Attribute | Purpose | Example |
|-----------|---------|---------|
| `rolearn` | **AWS IAM role ARN.** Maps EC2 instances in the node group to Kubernetes. When EC2 nodes boot, they use this role to authenticate. | `arn:aws:iam::154292417400:role/eksctl-eksdemo1-nodegroup-...` |
| `groups` | **System groups for nodes.** `system:bootstrappers` and `system:nodes` are predefined Kubernetes groups allowing nodes to join the cluster and register. | `[system:bootstrappers, system:nodes]` |
| `username` | **Kubernetes username for nodes.** `{{EC2PrivateDNSName}}` is a placeholder replaced with the actual EC2 private DNS name of each node. | `system:node:ip-10-0-1-45.ec2.internal` |

---

### ClusterRole Attributes

#### **Example: developer-clusterrole.yml**
```yaml
kind: ClusterRole
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  name: developer-clusterrole
rules:
  - apiGroups: [""]
    resources: ["pods", "secrets", "configmaps", "services"]
    verbs: ["get", "list", "create", "update", "patch"]
  - apiGroups: ["apps"]
    resources: ["deployments", "statefulsets", "daemonsets", "replicasets"]
    verbs: ["get", "list", "create", "update", "patch"]
  - apiGroups: ["batch"]
    resources: ["jobs", "cronjobs"]
    verbs: ["get", "list", "create", "update", "patch"]
```

| Attribute | Purpose | Explanation |
|-----------|---------|-------------|
| `kind` | **Resource type** | `ClusterRole` (cluster-wide) vs `Role` (namespace-specific). ClusterRoles apply to all namespaces. |
| `metadata.name` | **ClusterRole identifier** | Referenced in ClusterRoleBindings. Must be unique across the cluster. |
| `rules` | **Permission rules** | Array of permissions defining what resources and actions are allowed. |
| `apiGroups` | **Kubernetes API groups** | `""` = core API (pods, secrets), `"apps"` = deployments, `"batch"` = jobs. Use `"*"` for all groups. |
| `resources` | **Kubernetes resource types** | Examples: `pods`, `secrets`, `deployments`. Use `"*"` for all resources. In one rule, you can list multiple related resources. |
| `verbs` | **HTTP methods/actions** | `get` (read single), `list` (read many), `create`, `update`, `patch` (modify), `delete`, `watch` (stream events). Use `"*"` for all. |

**Verb Details:**
- `get` - Read a single resource: `kubectl get secret my-secret`
- `list` - Read multiple resources: `kubectl get secrets`
- `create` - Create new resources: `kubectl apply -f` or `kubectl create`
- `update` - Replace entire resource
- `patch` - Modify resource fields: used by `kubectl apply` for incremental updates
- `delete` - Remove resources
- `watch` - Stream real-time events

**Why `update` AND `patch`?**
`kubectl apply` needs `patch` to make incremental changes. Traditional `kubectl replace` uses `update`. Including both ensures compatibility.

---

### ClusterRoleBinding Attributes

#### **Example: developer-clusterrolebinding.yml**
```yaml
kind: ClusterRoleBinding
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  name: developer-clusterrolebinding
subjects:
  - kind: User
    name: eks-developer
roleRef:
  kind: ClusterRole
  name: developer-clusterrole
  apiGroup: rbac.authorization.k8s.io
```

| Attribute | Purpose | Explanation |
|-----------|---------|-------------|
| `kind` | **Resource type** | `ClusterRoleBinding` (cluster-wide) or `RoleBinding` (namespace-specific). |
| `metadata.name` | **Binding identifier** | Unique name for this binding relationship. No functional purpose, but helps identify what the binding does. |
| `subjects` | **Who gets permissions** | Array of Users, Groups, or ServiceAccounts receiving the permissions. |
| `subjects[].kind` | **Subject type** | `User` (individual), `Group` (group of users), or `ServiceAccount` (service account). |
| `subjects[].name` | **Subject identifier** | For Users, this is the Kubernetes username from aws-auth ConfigMap (e.g., `eks-developer`). Must match exactly. |
| `roleRef.kind` | **Role type** | `ClusterRole` or `Role`. Must be uppercase. |
| `roleRef.name` | **Role identifier** | Name of the ClusterRole to bind to (e.g., `developer-clusterrole`). Must exist in the cluster. |
| `roleRef.apiGroup` | **API group** | Always `rbac.authorization.k8s.io` for RBAC resources. |

**How it connects:** 
This binding says: "Grant the ClusterRole `developer-clusterrole` to the Kubernetes User `eks-developer`."

The ClusterRoleBinding is the **glue** that connects:
- **Who** (subject: `eks-developer`) 
- **To What** (roleRef: `developer-clusterrole`)
- **At What Scope** (cluster-wide vs namespace-specific)

---

## Complete Interaction Example

### Scenario: Developer applies a secret to the cluster

**Step 1: Developer authenticates**
```bash
aws sts get-caller-identity
# Returns: arn:aws:iam::154292417400:user/eks-developer
```
What happens: AWS verifies the developer's credentials (AWS Access Key ID and Secret Access Key).

---

**Step 2: Developer updates kubeconfig**
```bash
eksctl utils write-kubeconfig --cluster=eksdemo1 --set-kubeconfig-context=true
# Kubeconfig now contains AWS SigV4 signing logic for this user
```
What happens: The kubeconfig is configured so kubectl will sign requests with the developer's AWS credentials.

---

**Step 3: Developer runs kubectl command**
```bash
kubectl apply -f secret.yml
```
What happens: kubectl reads the kubeconfig and prepares an HTTP request to the EKS API server.

---

**Step 4: Kubernetes API Server Authentication**
- kubectl sends the HTTP request to EKS API Server
- EKS API Server receives the request with AWS SigV4 signature
- Calls AWS STS to validate the signature
- AWS STS responds: "This signature is valid for IAM user `arn:aws:iam::154292417400:user/eks-developer`"
- API Server authenticates the request ✅

---

**Step 5: Kubernetes RBAC Authorization**
- API Server checks: "Does this user have permission to PATCH secrets?"
- Looks up aws-auth ConfigMap: finds mapping `eks-developer` → group `developer-clusterrole`
- Finds ClusterRoleBinding `developer-clusterrolebinding`: `eks-developer` → `developer-clusterrole`
- Loads ClusterRole `developer-clusterrole`
- Searches rules for: verb=`patch`, resource=`secrets`, apiGroup=`""`
- **Found!** The rule allows it
- **Decision: ALLOW** ✅

---

**Step 6: Secret is created successfully**
```bash
secret/my-app-secret created
```

---

## Troubleshooting

### Common Error: "Forbidden: User cannot get resource 'secrets'"
```
Error from server (Forbidden): error when retrieving current configuration of:
Resource: "/v1, Resource=secrets", GroupVersionKind: "/v1, Kind=Secret"
```
**Cause:** IAM user doesn't have `get` permission on secrets in their ClusterRole.

**Solution:** 
1. Add `get` verb to the secrets rule in the ClusterRole
2. Apply the updated ClusterRole: `kubectl apply -f developer-clusterrole.yml`

---

### Common Error: "User not found in aws-auth ConfigMap"
```
error: You must be logged in to the server (Unauthorized)
```
**Cause:** IAM user's ARN is not in the aws-auth ConfigMap, or ARN is incorrect.

**Solution:**
1. Verify IAM user ARN: `aws sts get-caller-identity`
2. Add the user's ARN to aws-auth.yml
3. Apply: `kubectl apply -f aws_auth.yml`
4. Regenerate kubeconfig: `eksctl utils write-kubeconfig --cluster=eksdemo1`

---

### Common Error: "ClusterRoleBinding not found"
```
error: unable to recognize "clusterrolebinding.yml": no matches for kind
```
**Cause:** ClusterRoleBinding hasn't been applied to the cluster yet.

**Solution:**
1. Run: `kubectl apply -f developer-clusterrolebinding.yml`
2. Verify: `kubectl get clusterrolebindings`

---

### Common Error: "ClusterRole not found"
```
The ClusterRole "developer-clusterrole" referenced in the ClusterRoleBinding does not exist
```
**Cause:** ClusterRole must exist before ClusterRoleBinding can reference it.

**Solution:**
1. Run: `kubectl apply -f developer-clusterrole.yml`
2. Verify: `kubectl get clusterroles | grep developer`

---

## References

- [AWS EKS RBAC Documentation](https://docs.aws.amazon.com/eks/latest/userguide/manage-users-or-iam-roles.html)
- [Kubernetes RBAC Authorization](https://kubernetes.io/docs/reference/access-authn-authz/rbac/)
- [eksctl Documentation](https://eksctl.io/)
- [AWS IAM Documentation](https://docs.aws.amazon.com/iam/)

---

## Author
Created for demonstrating RBAC integration with EKS and AWS IAM users. Includes complete examples, command references, and troubleshooting guides. 

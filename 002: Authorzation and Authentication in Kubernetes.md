002: Authorzation and Authentication in Kubernetes

## 🔍 What This Lab Is About

Technically, this lab focuses on manually constructing a kubeconfig file to authenticate a user with a Kubernetes cluster using client certificates. It demonstrates how the API server validates identity and grants access based on the active context, without relying on RBAC. I also extract and reference real certificate files, handle permissions, and test configurations using the kubectl config commands.

Instead of relying on defaults or prebuilt tooling, I manually constructed, inspected, and troubleshooted the kubeconfig file. By going through this lab I was able to know : 

1. What a kubeconfig file actually contains and how Kubernetes uses it

2. How authentication works using client certificates and keys

3. How authorization depends on the context and namespace setup

4. How to extract real certificate data from an existing config

5. How to fix common problems like missing contexts, bad paths, or permission issues


## Setting Up My Test Environment

First, I created a directory and started building my test kubeconfig:

```bash
cd delta
nano delta-demo
```

I started with this basic structure:
```yaml
apiVersion: v1
kind: Config
preferences: {}

clusters:
- cluster:
  name: development

users:
- name: developer

contexts:
- context:
  name: dev-frontend
```

## Building the Configuration Step by Step

### Adding Cluster Configuration
```bash
kubectl config --kubeconfig=delta-demo set-cluster development --server=https://1.2.3.4 --certificate-authority=fake-ca-file
```
Output: `Cluster "development" set.`

### Adding User Credentials
```bash
kubectl config --kubeconfig=delta-demo set-credentials developer --client-certificate=fake-cert-file --client-key=fake-key-seefile
```
Output: `User "developer" set.`

### Creating a Test Namespace
I also created a test namespace to work with:
```bash
kubectl create ns test
```
Output: `namespace/test created`

### Setting Up the Context
```bash
kubectl config --kubeconfig=delta-demo set-context dev-frontend --cluster=development --namespace=test --user=developer
```
Output: `Context "dev-frontend" modified.`

### Viewing My Configuration
After setting everything up, I checked what it looked like:
```bash
kubectl config --kubeconfig=delta-demo view
```

This showed me:
```yaml
apiVersion: v1
clusters:
- cluster:
    certificate-authority: fake-ca-file
    server: https://1.2.3.4
  name: development
contexts:
- context:
    cluster: development
    namespace: test
    user: developer
  name: dev-frontend
current-context: ""
kind: Config
preferences: {}
users:
- name: developer
  user:
    client-certificate: fake-cert-file
    client-key: fake-key-seefile
```

## First Problem: No Current Context Set

When I tried to check the current context:
```bash
kubectl config --kubeconfig=delta-demo current-context
```
I got: `error: current-context is not set`

**How I fixed it:**
```bash
kubectl config --kubeconfig=delta-demo use-context dev-frontend
```
Output: `Switched to context "dev-frontend".`

Now checking the current context worked:
```bash
kubectl config --kubeconfig=delta-demo current-context
```
Output: `dev-frontend`

## Exploring My Working Configuration

I wanted to understand my main kubeconfig better, so I looked at it:
```bash
kubectl config view --raw
```

This showed me a working configuration with real certificate data (base64 encoded). I could also view it in a cleaner format:
```bash
kubectl config view
```

Which displayed:
```yaml
apiVersion: v1
clusters:
- cluster:
    certificate-authority-data: DATA+OMITTED
    server: https://172.30.1.2:6443
  name: kubernetes
contexts:
- context:
    cluster: kubernetes
    user: kubernetes-admin
  name: kubernetes-admin@kubernetes
current-context: kubernetes-admin@kubernetes
kind: Config
preferences: {}
users:
- name: kubernetes-admin
  user:
    client-certificate-data: DATA+OMITTED
    client-key-data: DATA+OMITTED
```

## Converting Certificate Data to Files

I decided to extract the real certificates and create file-based versions. First, I saved my current environment:
```bash
export KUBECONFIG_SAVED="$KUBECONFIG"
```

### Extracting Certificates from the Working Config
```bash
mkdir -p ~/.kube/certs

# Extract CA certificate
kubectl config view --raw -o jsonpath='{.clusters[0].cluster.certificate-authority-data}' | base64 -d > ~/.kube/certs/ca.crt

# Extract client certificate  
kubectl config view --raw -o jsonpath='{.users[0].user.client-certificate-data}' | base64 -d > ~/.kube/certs/client.crt

# Extract client key
kubectl config view --raw -o jsonpath='{.users[0].user.client-key-data}' | base64 -d > ~/.kube/certs/client.key
```

### Setting Proper Permissions
```bash
chmod 600 ~/.kube/certs/client.key
chmod 644 ~/.kube/certs/ca.crt ~/.kube/certs/client.crt
```

I also displayed what I had done:
```bash
echo "Certificates extracted to:"
echo "CA Certificate: ~/.kube/certs/ca.crt"
echo "Client Certificate: ~/.kube/certs/client.crt"
echo "Client Key: ~/.kube/certs/client.key"
```

### Creating a File-Based Config
I created a new kubeconfig that referenced the certificate files instead of embedding the data:
```bash
cat << EOF > ~/.kube/config-with-files
apiVersion: v1
clusters:
- cluster:
    certificate-authority: $HOME/.kube/certs/ca.crt
    server: https://172.30.1.2:6443
  name: kubernetes
contexts:
- context:
    cluster: kubernetes
    user: kubernetes-admin
  name: kubernetes-admin@kubernetes
current-context: kubernetes-admin@kubernetes
kind: Config
preferences: {}
users:
- name: kubernetes-admin
  user:
    client-certificate: $HOME/.kube/certs/client.crt
    client-key: $HOME/.kube/certs/client.key
EOF
```

## Second Problem: Wrong Directory

I ran into a silly issue when I changed directories:
```bash
cd ..
cat delta-demo
```
Got: `cat: delta-demo: No such file or directory`

**The issue:** I was in the wrong directory! My file was in `~/delta/` but I was in `~`.

**Simple fix:** Go back to the right directory:
```bash
cd delta
cat delta-demo
```

## Updating My Test Config with Real Certificates

Now I wanted to update my test configuration to use real certificates instead of the fake ones. I extracted certificates specifically for this demo:

```bash
mkdir -p ~/delta/certs
echo "Extracting certificates from current kubeconfig..."

kubectl config view --raw -o jsonpath='{.clusters[0].cluster.certificate-authority-data}' | base64 -d > ~/delta/certs/ca.crt
kubectl config view --raw -o jsonpath='{.users[0].user.client-certificate-data}' | base64 -d > ~/delta/certs/client.crt
kubectl config view --raw -o jsonpath='{.users[0].user.client-key-data}' | base64 -d > ~/delta/certs/client.key

chmod 600 ~/delta/certs/client.key
chmod 644 ~/delta/certs/ca.crt ~/delta/certs/client.crt

echo "Certificates extracted to ~/delta/certs/"
```

Then I updated my delta-demo file with real certificate paths:
```bash
cat << 'EOF' > ~/delta/delta-demo
apiVersion: v1
clusters:
- cluster:
    certificate-authority: /root/delta/certs/ca.crt
    server: https://1.2.3.4
  name: development
contexts:
- context:
    cluster: development
    namespace: test
    user: developer
  name: dev-frontend
current-context: dev-frontend
kind: Config
preferences: {}
users:
- name: developer
  user:
    client-certificate: /root/delta/certs/client.crt
    client-key: /root/delta/certs/client.key
EOF
```

I documented what I had created:
```bash
echo "Updated delta-demo file with real certificate paths:"
echo "- CA Certificate: /root/delta/certs/ca.crt"
echo "- Client Certificate: /root/delta/certs/client.crt" 
echo "- Client Key: /root/delta/certs/client.key"
echo ""
echo "To test the config:"
echo "export KUBECONFIG=~/delta/delta-demo"
echo "kubectl config current-context"
```

## Third Problem: Minify Without Current Context

Earlier, before I had set the current context, I tried to minify and got an error. But now I could test it properly:

From the wrong directory (outside delta):
```bash
kubectl config --kubeconfig=delta-demo view --minify
```
Still got: `error: current-context must exist in order to minify`

But from the correct directory:
```bash
cd delta
kubectl config --kubeconfig=delta-demo view --minify
```

This worked perfectly and showed me:
```yaml
apiVersion: v1
clusters:
- cluster:
    certificate-authority: /root/delta/certs/ca.crt
    server: https://1.2.3.4
  name: development
contexts:
- context:
    cluster: development
    namespace: test
    user: developer
  name: dev-frontend
current-context: dev-frontend
kind: Config
preferences: {}
users:
- name: developer
  user:
    client-certificate: /root/delta/certs/client.crt
    client-key: /root/delta/certs/client.key
```

## Final Configuration Verification

My final delta-demo configuration looked like this:
```bash
cat delta-demo
```

```yaml
apiVersion: v1
clusters:
- cluster:
    certificate-authority: /root/delta/certs/ca.crt
    server: https://1.2.3.4
  name: development
contexts:
- context:
    cluster: development
    namespace: test
    user: developer
  name: dev-frontend
current-context: dev-frontend
kind: Config
preferences: {}
users:
- name: developer
  user:
    client-certificate: /root/delta/certs/client.crt
    client-key: /root/delta/certs/client.key
```

## What I Learned From This Experience

### Key Issues I Encountered:
1. **Forgetting to set current-context** - Created contexts but didn't activate one
2. **Working directory confusion** - Trying to access files from wrong location  
3. **Fake vs real certificates** - Started with placeholders, had to extract real ones
4. **File permissions** - Needed proper chmod settings for certificate files

### Commands That Saved Me:
```bash
# Always check where you are
pwd

# See your contexts
kubectl config get-contexts

# Set current context  
kubectl config use-context <context-name>

# View clean config
kubectl config view

# View with certificate data
kubectl config view --raw

# Minify to see only current context
kubectl config view --minify

# Work with specific kubeconfig
kubectl config --kubeconfig=filename <command>
```

### The Process That Worked:
1. Create basic structure
2. Set cluster, user, context
3. **Don't forget to set current-context!**
4. Extract real certificates if needed
5. Set proper file permissions
6. Update paths in config
7. Test with minify to verify everything works

This whole exercise taught me that most kubeconfig issues are simple mistakes - wrong directory, missing current-context, or incorrect file paths. Taking it step by step and checking your work at each stage prevents most problems.

```
kubectl create namespace unprivileged-user
kubectl create serviceaccount -n unprivileged-user fake-user
kubectl create rolebinding -n unprivileged-user fake-editor --clusterrole=edit  --serviceaccount=unprivileged-user:fake-user

# Create the fake-user's kubeconfig file
secretName=$(kubectl -n unprivileged-user get serviceAccount fake-user -o jsonpath='{.secrets[0].name}')
ca=$(kubectl -n unprivileged-user get secret/$secretName -o jsonpath='{.data.ca\.crt}')
token=$(kubectl -n unprivileged-user get secret/$secretName -o jsonpath='{.data.token}' | base64 --decode)

# Echo to screen to be saved as kubeconfig file, and later used with kubectl
echo "
---
apiVersion: v1
kind: Config
clusters:
- name: default-cluster
  cluster:
    certificate-authority-data: ${ca}
    server: https://192.168.232.100:6443
contexts:
- name: default-context
  context:
    cluster: default-cluster
    namespace: default
    user: default-user
current-context: default-context
users:
- name: default-user
  user:
    token: ${token}
"
```
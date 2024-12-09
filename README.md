
### Securing-K8s-7: Encrypting Confidential Data with Encryption Configuration on a Non-Managed Kubernetes Cluster on Linode

In this guide, we will configure **encryption at rest** for Kubernetes Secrets stored in **etcd** on a **self-hosted Kubernetes cluster** deployed on Linode. Encryption ensures sensitive information such as passwords, API keys, and tokens remain secure in the event of unauthorized access to etcd. Additionally, we’ll explore why this approach is not applicable to managed Kubernetes solutions like **Linode Kubernetes Engine (LKE)** and discuss real-world applications.

---

### Why This Doesn't Apply to LKE

Managed Kubernetes services like LKE provide limited access to the **API server configuration** and etcd because:
1. **Restricted Access**: The control plane is managed entirely by Linode, preventing direct modifications to components like etcd or the API server.
2. **Built-in Security Features**: Managed clusters often include default security measures, like encryption at rest, configured by the provider, rendering custom encryption redundant.
3. **Maintenance Overhead**: Providers handle upgrades, backups, and scaling of the control plane, eliminating the need for user-managed configurations.

If encryption customization is a priority, hosting your own Kubernetes cluster is the appropriate choice.

---

### Step 1: Prerequisites

Ensure the following are in place:
1. **Non-Managed Kubernetes Cluster**: A Kubernetes cluster deployed on Linode, with full access to etcd and the API server.
2. **kubectl Configured**: Your local `kubectl` is configured to communicate with the cluster.
3. **etcdctl Installed**: The etcd command-line tool (`etcdctl`) is installed and configured to access your etcd instance.
4. **Backup Setup**: A reliable etcd backup process is in place to prevent data loss during configuration changes.

---

### Step 2: Create an Encryption Configuration File

On your master node, create the encryption configuration file at `/etc/kubernetes/encryption-config.yaml`:

```yaml
apiVersion: apiserver.config.k8s.io/v1
kind: EncryptionConfiguration
resources:
  - resources:
      - secrets
    providers:
      - aesgcm:
          keys:
            - name: key1
              secret: c2VjdXJlLXNlY3JldC1rZXktaGV4LXN0cmluZw==
      - identity: {}
```

#### Explanation:
- **aesgcm**: Configures encryption using AES-GCM, a secure and performant encryption algorithm.
- **identity**: Acts as a fallback to allow backward compatibility for plaintext data during the transition.

Secure the file to prevent unauthorized access:

```bash
chmod 600 /etc/kubernetes/encryption-config.yaml
```

---

### Step 3: Update the API Server Configuration

1. Locate the **API server configuration** file, typically `/etc/kubernetes/manifests/kube-apiserver.yaml`.
2. Add the following flag to the `kube-apiserver` command:

```yaml
- --encryption-provider-config=/etc/kubernetes/encryption-config.yaml
```

3. Save the changes. The kube-apiserver process will automatically restart and apply the new configuration.

---

### Step 4: Create and Test a Secret

To confirm encryption is working:

1. Create a Kubernetes Secret:

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: my-secret
type: Opaque
data:
  username: YWRtaW4=  # Base64 for "admin"
  password: MWYyZDFlMmU2N2Rm  # Base64 for "1f2d1e2e67df"
```

Apply the Secret:

```bash
kubectl apply -f my-secret.yaml
```

2. Verify the Secret in etcd:

```bash
etcdctl --endpoints=<etcd-endpoint> get /registry/secrets/default/my-secret --cert=<path-to-cert> --key=<path-to-key> --cacert=<path-to-cacert>
```

The output should display encrypted data, confirming the configuration is working.

---

### Step 5: Real-World Applications

#### **Use Case 1: Securing Sensitive Information**
Encrypting Secrets ensures that passwords, API keys, and tokens remain secure, even if etcd is compromised.

#### **Use Case 2: Regulatory Compliance**
Encryption at rest helps meet security requirements for data protection regulations like GDPR and HIPAA.

#### **Use Case 3: Multi-Tenant Clusters**
In self-hosted multi-tenant environments, encryption ensures isolation and confidentiality between tenants.

---

### Ongoing Maintenance

#### Rotate Keys Periodically:
1. Add a new key to the `encryption-config.yaml`:

```yaml
- aesgcm:
    keys:
      - name: key2
        secret: bmV3LXNlY3JldC1rZXktaGV4LXN0cmluZw==
      - name: key1
        secret: c2VjdXJlLXNlY3JldC1rZXktaGV4LXN0cmluZw==
```

2. Apply the configuration and re-encrypt Secrets:

```bash
kubectl get secrets --all-namespaces -o yaml | kubectl replace -f -
```

#### Regular Audits:
Periodically verify that Secrets remain encrypted and ensure configuration files are secure.

---

### Conclusion

For self-hosted Kubernetes clusters on Linode, enabling encryption at rest for etcd protects sensitive data from unauthorized access, mitigates insider threats, and ensures compliance with regulatory requirements. While managed services like LKE handle encryption automatically, this guide empowers you to implement and manage encryption for clusters requiring custom configurations and advanced security measures.

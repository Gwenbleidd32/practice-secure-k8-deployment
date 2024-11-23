* Secure Kubernetes Stateful Application Configuration

This repository contains working iterations of a secure Kubernetes stateful application configuration. The development began after encouragement from Theo following our study of [NIST SP 800-190](https://csrc.nist.gov/publications/detail/sp/800-190/final). My objective was to adapt a basic Kubernetes application to adhere to best practices by observing alerts received in the pipeline and applying new methods of protection learned from NIST guidelines and other Kubernetes documentation.

The files in this repository were developed in stages, incorporating newly discovered tools and methods. Below, I outline the most recent changes implemented in detail, as well as provide quick bullet points for the existing resources.

---

## Overview of the YAML Files

Aside from the deployment file, which we will review in detail near the end, the configurations include:

### Namespace

The namespace is created using the imperative Kubernetes command:
`kubectl create namespace lizzoslunch`

Beyond naming it `lizzoslunch`, I have included additional configurations to enhance security.

**Pod Security Policies (PSPs)** are implemented through labels within the namespace. These policies provide three tiers to enforce and monitor security for the replicas we intend to deploy:

- **Warn**: Provides a terminal warning upon execution of a manifest or command that deploys pods without a security context, non-root user designation, limited privileges, or resource constraints.
- **Audit**: Logs any policy violations within your cluster's audit logs.
- **Enforce**: Prevents deployments that violate security policies from deploying at all.

**Key Takeaways**:
 Enhances development by providing immediate feedback on security policy compliance.
- Adds an additional layer of defense that aligns with organizational security guidelines.
- Encourages adherence to principles such as non-root usage, least privilege, and resource limitations.

---

### Storage Class

A basic storage class is implemented to support the stateful set. In this configuration, the `reclaimPolicy` is set to `Delete`:
```python
reclaimPolicy: Delete  # Deletes volume after cluster teardown.
```

This is suitable for a lab environment where resource cleanup is desired after teardown. In a production environment, the `reclaimPolicy` would typically be set to `Retain` to preserve persistent volumes in the event the cluster is compromised or destroyed.

**Key Takeaways**:
- The `Delete` policy aids in resource management during development.
- In production, `Retain` ensures data persistence and aids in disaster recovery.

---

### Service

The service configuration is straightforward, targeting the container's listening port of `5000`.

**Key Takeaways**:
- Exposes the application on the specified port.
- Facilitates network communication with the deployed pods.

---

### Autoscaler

The Autoscaler provides basic functionality similar to Auto Scaling Groups (ASGs) or Managed Instance Groups, allowing pods to scale in response to CPU utilization.

**Key Takeaways**:
- Enhances application availability and performance.
- Adjusts pod replicas based on real-time resource demands.

---

### Resource Quota

The `ResourceQuota` defines the maximum resource utilization for compute and memory within the entire namespace. This configuration connects to the resource specifications defined within the pods being deployed.
```python
      #>>> RESOURCE-REQUESTS
        resources:
          requests: # Minimum resources
            memory: "256Mi"  
            cpu: "250m"            
          limits: # Maximum resources
            memory: "512Mi"      
            cpu: "500m"
```

**Key Takeaways**:
- Protects cluster integrity by preventing resource overutilization.
- Ensures that resource consumption remains within predefined limits.
- Requires all containers within the namespace to specify their own resource requests and limits.

---

### Network Policies

Network policies define where applications can send requests and from whom they can receive requests. By enabling a policy within a namespace, you can either define rules for all applications or match specific applications by their labels.
```python
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: isolation-policy
  namespace: lizzoslunch
spec:
  podSelector: #{}              
    matchLabels:
      app: type-a     # Applies to only a specific group of pods within our namespace
  policyTypes:      # Establishing rules after Deny all is implied upon Policy creation
  - Ingress
  - Egress
  ingress:
  - from:
      - podSelector:  
          matchLabels:
            app: type-a 
  - from:
      - ipBlock:
          cidr: 0.0.0.0/0     # Allow traffic from external sources 
    ports:          # Can be finegrained and sync'd up with your load balancer's allowed Ranges.
    - protocol: TCP
      port: 80            # Allow traffic on port 80
    - protocol: TCP
      port: 5000         # Allow traffic on port 5000
  egress:
  - to:
      - namespaceSelector:  
          matchLabels:
            app: database      # Dummy label for a database namespace
        podSelector: #{}
            matchLabels:
              app: database-app  # Dummy label for a database pods in another namespace
  - to:
      - ipBlock:                 
          cidr: 10.176.76.4/32     #Allowing Export to my PSC Endpoint
    ports:
    - protocol: TCP
      port: 80
    - protocol: TCP
      port: 443                      
---
```

By default, the policy denies all ingress and egress traffic, forcing explicit specification of allowed traffic. This process encourages fine-grained control over network communications.

**Key Takeaways**:
- Enhances security by adhering to the principle of least privilege.
- Requires deliberate configuration of allowed network traffic.
- Reduces the attack surface by restricting unnecessary communication paths.

---

### Statefulset

In the `StatefulSet` configuration, several key points go beyond previous security settings:

#### Init Container

The `initContainer` is designed to administer settings for the environments within the pod before the main containers are started. Specifically, it addresses the challenge:

**"How do I give my application permissions to use its attached storage without compromising security?"**

By using an `initContainer`, which only exists during the initialization phase, we can grant necessary permissions without exposing the main container to elevated privileges.

Key configurations include:
- **Security Context**: The `initContainer` runs as the root user to change ownership and permissions of the mounted volume.
- **Linux Permissions**: Grants read and write permissions to the main container.
- **Resource Specifications**: Defines resource utilization to comply with namespace requirements.
- **Volume Mounts**: Mounts the storage so the container can set permissions.

**Key Takeaways**:
- Minimizes security risks by limiting the duration and scope of elevated privileges.
- Ensures the main container operates with the least privilege necessary.
- Aligns with security best practices by isolating permission changes to the initialization phase.

#### Main Container

The main container's security context remains restricted, adhering to non-root execution and limited privileges.

Additional configurations include:
- **Labels and specified port names for Istio Prometheus Monitoring**: Sets up monitoring capabilities (to be expanded upon later).
- **Volume Mount**: Provides a path for the container and application to access attached storage.
- **Environment Variable**: Ensures the application consistently accesses the correct mount.
- **Liveness and Readiness Probes**:
    - **Readiness Probe**: Checks if the application is ready to receive traffic before adding it to the service.
    - **Liveness Probe**: Monitors the application's health and restarts the pod if necessary.

**Python functions for probes**:
```python
#Liveliness and readiness functions:
# We Specify a target route on top of the applications IP for requests to be served
#Tests can be configured to be much more dynamic and application specific.
# >>> Probe Functions
@app.route('/i-am-ready')
def vigor():
    # Check application dependencies or health conditions if needed
    return 'Et ano quettue sena vut?', 200

@app.route('/i-am-alive')
def fitness():
    return 'Fa quettue tu sei fraisse?', 200
#>>> End of Probe Functions
```

**Container Script**:

To confirm functionality of the route to the attached storage under strict security protocols, a container script is used:
```python
#>>> INIT/CONTAINER SCRIPT TO TEST MOUNT PATH AND WRITE ACCCESS.
#Writes date and time to a new text file within the mounted storage every 30's
command: ["sh", "-c", "python app.py & while true; do date >> /data/date.txt; sleep 30; done"] 
```

This script writes the date and time to a new text file within the mounted storage every 30 seconds, verifying read and write access.

**Key Takeaways**:
- Maintains security by operating under restricted privileges.
- Ensures application functionality with necessary storage access.
- Incorporates monitoring and health checks to improve reliability.

## Challenges and Lessons Learned

During the configuration process, several challenges were encountered:

- **Balancing Security and Functionality**: Granting necessary permissions for storage access without compromising security required careful planning using `initContainers`.
- **Resource Quota Compliance**: Ensuring all containers specified their resource requests and limits to comply with namespace `ResourceQuotas`.
- **Network Policy Complexity**: Configuring fine-grained network policies demanded attention to detail to avoid unintended communication blocks.

**Lessons Learned**:

- **Iterative Development**: Incremental changes and testing allowed for manageable troubleshooting and validation.
- **Documentation and References**: Consulting Kubernetes documentation and NIST guidelines was crucial for implementing best practices.


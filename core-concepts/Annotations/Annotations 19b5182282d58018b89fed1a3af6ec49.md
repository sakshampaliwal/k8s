# Annotations

Kubernetes annotations are key-value pairs of metadata that can be attached to Kubernetes objects. Unlike labels, annotations are primarily used to attach non-identifying metadata that's meant for tools and libraries rather than for selecting or grouping objects.

The rise of annotations in Kubernetes stems from several key factors:

1. Need for Extended Metadata
Before annotations, there was no clean way to attach arbitrary metadata to Kubernetes objects. Teams needed a way to store information like:
- Build/release IDs
- Timestamps
- Git branch information
- Contact information for teams managing the resource
- Tool-specific configuration

1. Tool Ecosystem Development
As the Kubernetes ecosystem grew, third-party tools and operators needed a way to:
- Store their configuration
- Track state
- Enable features
- Manage deployments
- Configure ingress behavior

For example, ingress controllers commonly use annotations to configure SSL, rate limiting, and routing rules:

```yaml
metadata:
  annotations:
    nginx.ingress.kubernetes.io/ssl-redirect: "true"
    nginx.ingress.kubernetes.io/rate-limit: "100"

```

1. Cloud Provider Integration
Cloud providers needed a way to specify cloud-specific configurations. Annotations became the standard way to configure:
- Load balancer settings
- Storage configurations
- Auto-scaling behaviors

Example for AWS load balancer configuration:

```yaml
metadata:
  annotations:
    service.beta.kubernetes.io/aws-load-balancer-type: "nlb"
    service.beta.kubernetes.io/aws-load-balancer-internal: "true"

```

Key differences that led to annotations' adoption alongside labels:

- Labels are for identification and selection
- Annotations are for non-identifying metadata
- Labels have strict character limitations
- Annotations can store larger amounts of data
- Labels are queryable for object selection
- Annotations are meant for tool consumption

Common use cases today:

- Ingress configuration
- Service mesh settings
- Monitoring configuration
- Deployment strategies
- Documentation
- Build/release tracking
- Security policies
- Cost allocation

The rise of annotations reflects Kubernetes' evolution from a container orchestrator to a full platform for cloud-native applications. They've become essential for extending Kubernetes' functionality without modifying its core API.

### Example:

### **Understanding Annotations in Kubernetes**

Annotations in Kubernetes **add extra information** to objects, but they **do not affect how Kubernetes schedules or manages them**. Instead, they provide **metadata** that tools like monitoring, logging, or security systems can use.

---

### **Example: Prometheus Annotations**

Consider the following example in a Pod or Service manifest:

```yaml
metadata:
  annotations:
    prometheus.io/scrape: "true"
    prometheus.io/path: "/metrics"
    prometheus.io/port: "9102"

```

Each of these annotations tells **Prometheus** (a monitoring tool) **how to scrape metrics** from the application.

### **What Each Annotation Does?**

| Annotation | Meaning |
| --- | --- |
| `prometheus.io/scrape: "true"` | This tells Prometheus to collect (scrape) metrics from this pod or service. |
| `prometheus.io/path: "/metrics"` | This tells Prometheus **where** to find the metrics endpoint in the application (usually `/metrics`). |
| `prometheus.io/port: "9102"` | This tells Prometheus **which port** the application exposes metrics on. |

---

### **How This Works in Practice?**

1. **Your application runs inside a pod or container.**
    - It has an endpoint (`/metrics`) that exposes monitoring data in Prometheus format.
2. **Prometheus discovers the pod/service.**
    - Prometheus can be configured to check for these annotations.
3. **If `prometheus.io/scrape: "true"` is present, Prometheus collects the metrics.**
    - It will request the data from `http://<pod-ip>:9102/metrics`.
4. **Prometheus stores and visualizes the collected data.**
    - This helps track CPU, memory, request counts, etc.

---

### **Without These Annotations**

If you **don’t** add these annotations, **Prometheus will not collect metrics** from this pod or service unless it is manually configured.

---

### **Example: Full Pod Manifest with Annotations**

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: my-app
  labels:
    app: my-app
  annotations:
    prometheus.io/scrape: "true"
    prometheus.io/path: "/metrics"
    prometheus.io/port: "9102"
spec:
  containers:
  - name: my-app
    image: my-app:latest
    ports:
    - containerPort: 9102

```

**Explanation:**

- The app inside the pod exposes metrics on port `9102` at the `/metrics` path.
- Prometheus sees the annotations and starts scraping metrics from `http://<pod-ip>:9102/metrics`.

---

### **AWS Load Balancer Controller & Annotations in Kubernetes**

The **AWS Load Balancer Controller** is a Kubernetes controller that manages AWS **Application Load Balancers (ALB)** and **Network Load Balancers (NLB)** for services and ingress resources. It dynamically provisions and configures these load balancers based on the annotations provided in Kubernetes manifests.

---

## **How AWS Load Balancer Controller Works**

- When you deploy a **Kubernetes Service** or **Ingress** with AWS-specific annotations, the AWS Load Balancer Controller reads these annotations and **creates an AWS Load Balancer accordingly**.
- The controller automatically **attaches EC2 instances or Kubernetes pods** behind the load balancer based on the configurations in the annotations.

---

## **Example 1: Creating an Internal Load Balancer (NLB)**

If you want to create an **internal Network Load Balancer** instead of a public one, you can use the annotation:

```yaml
service.beta.kubernetes.io/aws-load-balancer-internal: "true"

```

### **Service Manifest Example**

```yaml
apiVersion: v1
kind: Service
metadata:
  name: my-internal-service
  annotations:
    service.beta.kubernetes.io/aws-load-balancer-type: "nlb"
    service.beta.kubernetes.io/aws-load-balancer-internal: "true"
spec:
  selector:
    app: my-app
  ports:
    - protocol: TCP
      port: 80
      targetPort: 8080
  type: LoadBalancer

```

### **What Happens?**

- The AWS Load Balancer Controller **provisions an NLB** instead of an ALB.
- The **NLB is internal**, meaning it is only accessible within the AWS **VPC**.
- The NLB forwards traffic from port **80** to the application running on port **8080** in Kubernetes pods.

---

## **Example 2: Creating an ALB with Ingress Annotations**

If you want to use an **Application Load Balancer (ALB)** with an Ingress resource, you need to add ALB-specific annotations.

### **Ingress Manifest Example**

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: my-ingress
  annotations:
    alb.ingress.kubernetes.io/scheme: "internet-facing"
    alb.ingress.kubernetes.io/target-type: "ip"
    alb.ingress.kubernetes.io/listen-ports: '[{"HTTP": 80}, {"HTTPS": 443}]'
spec:
  ingressClassName: alb
  rules:
    - host: myapp.example.com
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: my-service
                port:
                  number: 80

```

### **What Happens?**

- The **AWS Load Balancer Controller provisions an ALB** (instead of an NLB).
- The ALB is **internet-facing** (`alb.ingress.kubernetes.io/scheme: "internet-facing"`), so it's accessible from the public internet.
- The **target type is `ip`**, meaning ALB will route traffic directly to Kubernetes pods instead of node instances.
- The ALB listens on **port 80 (HTTP) and 443 (HTTPS)** and forwards traffic to the Kubernetes **Service** named `my-service`.

---

### **Example:
Nginx Ingress Controller & Annotations in Kubernetes**

The **Nginx Ingress Controller** is a Kubernetes controller that manages external access to services inside the cluster using **Ingress resources**. It allows you to control routing, security, and other behaviors using **annotations** in the Ingress manifest.

## **How Nginx Ingress Controller Works**

- When you create an **Ingress resource**, the Nginx controller reads the annotations and configures the Nginx proxy accordingly.
- Annotations **modify Nginx behavior**, such as:
    - Setting timeouts
    - Enabling rate limiting
    - Configuring SSL termination
    - Redirecting traffic

---

## **Example 1: Basic Nginx Ingress with Annotations**

This example sets up an **Ingress resource** to route traffic to an internal Kubernetes **Service**.

### **Ingress Manifest Example**

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: my-ingress
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: "/"
    nginx.ingress.kubernetes.io/proxy-body-size: "50m"
spec:
  ingressClassName: nginx
  rules:
    - host: myapp.example.com
      http:
        paths:
          - path: /app
            pathType: Prefix
            backend:
              service:
                name: my-service
                port:
                  number: 80

```

### **What Happens?**

- Requests to `myapp.example.com/app` are **rewritten** to `/` in the backend using `nginx.ingress.kubernetes.io/rewrite-target: "/"`.
- The Nginx controller allows **larger request bodies** (e.g., file uploads) with `nginx.ingress.kubernetes.io/proxy-body-size: "50m"`.
- Traffic is forwarded to the **Service** named `my-service` on port **80**.

---

## **Example 2: Enabling HTTPS with SSL**

To **secure Ingress traffic with HTTPS**, you can configure SSL termination using an annotation.

### **Ingress Manifest for HTTPS**

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: secure-ingress
  annotations:
    nginx.ingress.kubernetes.io/ssl-redirect: "true"
    nginx.ingress.kubernetes.io/backend-protocol: "HTTPS"
spec:
  ingressClassName: nginx
  tls:
    - hosts:
        - mysecureapp.example.com
      secretName: my-tls-secret
  rules:
    - host: mysecureapp.example.com
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: secure-service
                port:
                  number: 443

```

### **What Happens?**

- The annotation `nginx.ingress.kubernetes.io/ssl-redirect: "true"` **forces HTTPS** for incoming requests.
- The Ingress resource uses a **TLS certificate stored in the secret `my-tls-secret`**.
- The backend service (`secure-service`) expects traffic on **port 443 (HTTPS)**.

---

## **Example 3: Rate Limiting Requests**

To prevent **DDoS attacks** or **limit API usage**, you can set rate limits using annotations.

### **Ingress Manifest for Rate Limiting**

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: rate-limit-ingress
  annotations:
    nginx.ingress.kubernetes.io/limit-rps: "10"
spec:
  ingressClassName: nginx
  rules:
    - host: myapi.example.com
      http:
        paths:
          - path: /api
            pathType: Prefix
            backend:
              service:
                name: api-service
                port:
                  number: 80

```

### **What Happens?**

- The annotation `nginx.ingress.kubernetes.io/limit-rps: "10"` restricts each client to **10 requests per second (RPS)**.
- Any requests exceeding this limit **will be throttled**.

---

### **When to Use What?(Labels and Annotations)**

- Use **Labels** when:
    - You need to **group**, **filter**, or **select** objects (e.g., for Deployments, Services, or queries).
    - Example: Labeling all Pods in a microservice as `app=cart`.
- Use **Annotations** when:
    - You need to **attach extra information** that is not used for selection.
    - Example: Adding a description, contact info, or configuration for an external tool.
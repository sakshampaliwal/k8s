# Secrets

A Secret is an object that allows you to store and manage sensitive information, such as passwords, access tokens, and TLS certificates, in a secure way. Secrets are stored in etcd, the Kubernetes key-value store, and can be mounted as a volume or exposed as environment variables in a pod.

To create a Secret, you can create a YAML file with the following format:

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: my-secret
type: Opaque
data:
  username: YWRtaW4=   # base64-encoded value of 'admin'
  password: MWYyZDFlMmU2N2Rm    # base64-encoded value of '1f2d1e2e67df'
```

In this example, we're creating a Secret called **`my-secret`** with two key-value pairs: **`username`** and **`password`**. The values are base64-encoded to protect their confidentiality.

To use the Secret in a pod, you can add a **`volumeMount`** and a **`volume`** section to the pod's YAML file. Here's an example:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: my-pod
spec:
  containers:
  - name: my-container
    image: my-image
    volumeMounts:
    - name: secret-volume
      mountPath: /etc/my-app
  volumes:
  - name: secret-volume
    secret:
      secretName: my-secret
```

In this example, we're creating a pod called **`my-pod`** with a container called **`my-container`**. We're mounting the **`my-secret`** Secret to the container's file system at **`/etc/my-app`**. The Secret is made available to the pod as a volume named **`secret-volume`**.

When the pod is created, Kubernetes will automatically inject the contents of the Secret into the **`/etc/my-app`** directory, making it available to the container. The container can then access the sensitive information as files on its file system.

Note that Secrets are not encrypted by default, so you should take care to protect them from unauthorized access. You can use Kubernetes RBAC (Role-Based Access Control) to limit access to Secrets to only authorized users and services.

To base64 encode values:

To base64 encode values, you can use a command-line tool such as **`base64`** in Linux or macOS, or an online base64 encoder tool.

For example, to base64 encode the string **`admin`** in Linux or macOS, you can run the following command:

```bash

$ echo -n 'admin' | base64
YWRtaW4=
```

The **`-n`** option is used to remove the newline character from the end of the string, which would otherwise be included in the encoding.

To encode the string **`1f2d1e2e67df`**, you can run the following command:

```bash
shellCopy code
$ echo -n '1f2d1e2e67df' | base64
MWYyZDFlMmU2N2Rm
```

Similarly, you can use an online base64 encoder tool to encode these values. Simply copy and paste the values into the encoder, and it will return the base64-encoded values.

Note that base64 encoding is not a form of encryption, and the encoded values can be easily decoded by anyone who has access to the encoded data. Therefore, you should not use base64 encoding to protect sensitive information unless it is also encrypted or otherwise secured.

Why Encoding??

The purpose of encoding sensitive data such as passwords or access tokens in Kubernetes Secrets is to protect them from being easily read or interpreted by unauthorized parties.

When you encode data using base64, the original data is converted into a sequence of characters that are safe to transmit and store in a variety of contexts, including text files, emails, and YAML files. The base64-encoded data can be decoded back into its original binary form when needed, but it appears as random, meaningless characters when viewed in its encoded form.
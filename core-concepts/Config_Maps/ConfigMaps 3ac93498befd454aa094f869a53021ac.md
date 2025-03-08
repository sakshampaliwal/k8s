# ConfigMaps

A ConfigMap is a way to store data that is used by your application running in Kubernetes. This data could be things like settings or configuration information that the application needs to run.

For example, if you have a web application that connects to a database, you might need to store the database hostname, port, username, and password somewhere so that your application can access it. Instead of hard-coding this information into your application, you can store it in a ConfigMap, which your application can then reference when it needs to connect to the database.

The benefit of using a ConfigMap is that it allows you to separate your application code from the configuration data. This makes it easier to update or change the configuration data without having to make changes to your application code.

Another Example: 

Suppose you have a microservices-based application that consists of multiple containers, each running a different service. One of the services is a notification service that sends email notifications to users. The email content and templates are stored in separate configuration files.

Instead of embedding the email templates in the notification service container, you could create a ConfigMap that stores the email templates as key-value pairs. You can then mount the ConfigMap as a volume in the notification service container so that the container has access to the email templates.

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: email-templates
data:
  welcome.txt: |
    Dear {{username}},

    Thank you for joining our platform!

    Best regards,
    The Platform Team

  reminder.txt: |
    Hi {{username}},

    Just a friendly reminder that you have an upcoming appointment with us.

    Best regards,
    The Platform Team
```

In this example, we're defining a ConfigMap called **`email-templates`**
 that has two entries, **`welcome.txt`**
 and **`reminder.txt`**, each with a corresponding email template. We're using the **`|`**
 character to indicate that the templates are multi-line strings.

Secrets are designed to be more secure than ConfigMaps. Secrets are base64-encoded and encrypted at rest, whereas ConfigMaps are not encrypted by default. This means that Secrets are a better choice for storing sensitive data that you don't want to be visible in plaintext.

Here's an example YAML file for creating a ConfigMap:

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: my-config
data:
  DB_HOST: db.example.com
  DB_PORT: "5432"
```

In this example, we're creating a ConfigMap called **`my-config`** that contains two key-value pairs: **`DB_HOST`** and **`DB_PORT`**.

Once you have created a ConfigMap, you can consume its data in several ways:

1. As environment variables in a Pod:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: my-pod
spec:
  containers:
    - name: my-container
      image: my-image:latest
      env:
        - name: DB_HOST
          valueFrom:
            configMapKeyRef:
              name: my-config
              key: DB_HOST
        - name: DB_PORT
          valueFrom:
            configMapKeyRef:
              name: my-config
              key: DB_PORT
```

In this example, we're creating a Pod called **`my-pod`** that uses the **`my-image:latest`** container image. We're also specifying two environment variables called **`DB_HOST`** and **`DB_PORT`** that are sourced from the **`my-config`** ConfigMap.

1. As command-line arguments in a Pod:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: my-pod
spec:
  containers:
    - name: my-container
      image: my-image:latest
      command: ["/bin/my-app", "--db-host=$(DB_HOST)", "--db-port=$(DB_PORT)"]
      env:
        - name: DB_HOST
          valueFrom:
            configMapKeyRef:
              name: my-config
              key: DB_HOST
        - name: DB_PORT
          valueFrom:
            configMapKeyRef:
              name: my-config
              key: DB_PORT
```

In this example, we're creating a Pod called **`my-pod`** that uses the **`my-image:latest`** container image. We're also specifying a command that references the **`DB_HOST`** and **`DB_PORT`** environment variables sourced from the **`my-config`** ConfigMap.

1. As a volume in a Pod:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: my-pod
spec:
  containers:
    - name: my-container
      image: my-image:latest
      volumeMounts:
        - name: config-volume
          mountPath: /etc/config
  volumes:
    - name: config-volume
      configMap:
        name: my-config
```

In this example, we're creating a Pod called **`my-pod`** that uses the **`my-image:latest`**container image. We're also specifying a volume called **`config-volume`**that is sourced from the **`my-config`**ConfigMap. The **`config-volume`**volume is mounted to the container at the path **`/etc/config`**
.
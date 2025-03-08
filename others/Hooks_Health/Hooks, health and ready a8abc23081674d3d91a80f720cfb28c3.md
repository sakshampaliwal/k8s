# Hooks, /health and /ready

What are Hooks?

Hooks are a set of mechanisms that allow you to execute scripts or commands at various stages of a Pod's lifecycle. Hooks provide a way to customize the behavior of a Pod by running custom code before or after certain events occur. Hooks are specified in the Pod definition file. There are several types of hooks in Kubernetes:

1. Init Containers: Init Containers are a type of hook that run before the main containers in a Pod. They can be used to perform pre-initialization tasks, such as setting up configuration files or downloading data. Init Containers are executed sequentially, and each must complete successfully before the next one starts.
2. PostStart and PreStop Hooks: PostStart and PreStop Hooks are executed after a container starts and before it stops, respectively. They can be used to perform tasks such as database initialization or clean-up before a container shuts down. These hooks are useful for handling dependencies and ensuring that data is properly saved before a container is terminated.
3. Lifecycle Hooks: Lifecycle Hooks are a set of hooks that run at specific points in a container's lifecycle. There are two types of Lifecycle Hooks: the preStop hook and the postStart hook. The preStop hook runs before a container is terminated, and the postStart hook runs immediately after the container is started. These hooks can be used to perform tasks such as database backups or data migration.

The **`/health`** endpoint is typically used to indicate the overall health of the application or service. It can be used to report on the status of various components or subsystems that are critical to the application's operation, such as databases, message queues, or other external services. The **`/health`** endpoint should return a 200 OK response if the application is healthy, and a non-200 response if there are any issues or failures.

The **`/ready`** endpoint is used to indicate when an application or service is ready to receive traffic. It is typically used during startup or initialization, and can be used to delay the startup of a Pod until the application or service is ready to handle requests. The **`/ready`** endpoint should return a 200 OK response when the application or service is ready to receive traffic, and a non-200 response if it is not yet ready.

Exposing **`/health`**and **`/ready`** endpoints is a common best practice in Kubernetes, as it allows for better monitoring and management of applications or services running in the cluster. By using probes and these endpoints, you can ensure that your applications or services are always available and responsive to user requests.
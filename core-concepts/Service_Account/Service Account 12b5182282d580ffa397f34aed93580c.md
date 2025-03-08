# Service Account

Whenever you access your Kubernetes cluster with [**kubectl**](https://kubernetes.io/docs/reference/kubectl/), you are authenticated by Kubernetes with your user account. User accounts are meant to be used by humans. But when a pod running in the cluster wants to access the Kubernetes API server, it needs to use a service account instead. Service accounts are just like user accounts but for non-humans.

Why Service Account Exists?

Why do Kubernetes Service Accounts exist? The simple answer is because pods are not humans, so it's good to have a distinction from user accounts. It's especially important for security reasons. Also, once you start using an external user management system with Kubernetes, it becomes even more important since all your users will probably follow typical [firstname.lastname@your-company.com](mailto:firstname.lastname@your-company.com) usernames.

But, you may wonder, why would pods inside the Kubernetes cluster need to connect to the Kubernetes API at all? Well, there are multiple use cases for it. The most common one is when you have a CI/CD pipeline agent deploying your applications to the same cluster. Many cloud-native tools also need access to your Kubernetes API to do their jobs, such as logging or monitoring applications.

The first thing you need to know is that you have probably already used service accounts even if you never configured any. That's because Kubernetes comes with a predefined service account called "default." And by default, every created pod has that service account assigned to it.

To See which service account is used by the pod:

`kubectl get pod <pod-name> -n <namespace> -o=jsonpath='{.spec.serviceAccountName}'`

So, it turns out that your pods have the default service account assigned even when you don't ask for it. This is because every pod in the cluster needs to have one (and only one) service account assigned to it. What can your pod do with that service account? Well, pretty much nothing. That default service account doesn't have any permissions assigned to it.

https://www.loft.sh/blog/kubernetes-service-account-what-it-is-and-how-to-use-it
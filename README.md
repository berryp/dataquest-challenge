# DataQuest DevOps take-home

My submission for the DevOps take-home task.
See [instructions](#instructions) at the end of this file for more
information.

## Rough instructions for getting the environment setup.

Here is a rough illustration of the steps I finally took to get Kubernetes
and Gitlab working, including deployments.

I am using macOS so all commands will be for that system. Commands
for the other systems are very similar.

### Notes

- Whist this exercise only asks for a small setup, I experienced issues during
  discovery period where Gitlab would not install/run properly if there weren't
  enough resources.
- Kubernetes is creating some pretty huge volumes, and I'm not sure why -
  especially as I am using defaults. This is going to take you way out of
  your free tier.
- The tools I used to help getting you setup quick, but the whole setup is
  very fragile. It would be much better to maintain a suite of config files
  that represent the complete stack. Alterntively, it would be nice to have
  Terraform templates to represent and manage the whole stack.
- If you follow Gitlab's official line and a number of blog posts on installing
  Gitlab on Kubernetes then you may find yourself in a deep rabit hole - as I
  did. If you are prototyping then use the simple tools with the simple configs.

### Tools

- [kops](https://github.com/kubernetes/kops) - like `kubectl` for clusters.
- [kubecli](https://kubernetes.io/docs/user-guide/kubectl-overview/) - Kubernetes CLI.
- [gitlab-runner](https://docs.gitlab.com/runner/) - used for registering runners.

### Install dependencies

```
$ brew install kops kubernetes-cli
$ curl --output /usr/local/bin/gitlab-runner https://gitlab-ci-multi-runner-downloads.s3.amazonaws.com/latest/binaries/gitlab-ci-multi-runner-darwin-amd64
$ chmod +x /usr/local/bin/gitlab-runner
```

### Setup environment

A lot of the commands here will use environment variables. You will need to
set the following:

- `AWS_ACCESS_KEY_ID` - AWS access key ID
- `AWS_SECRET_ACCESS_KEY` - AWS secret access key
- `CLUSTER_NAME` - The name of your cluster _(e.g. k8s.mydomain.com)_
- `DNS_ZONE` - The TLD of your cluster _(e.g. mydomain.com)_
- `NODE_ZONES` - The availability zones of the nodes _(e.g. eu-west-1a,eu-west-1b)_
- `MASTER_ZONES` - The availability zones of your master _(e.g. eu-west-1a)_

### Create the Kubernetes cluster

```
$ kops create cluster \
    --node-count 3 \
    --zones $NODE_ZONES \
    --master-zones $MASTER_ZONES \
    --dns-zone $DNS_ZONE \
    --node-size t2.medium \
    --master-size t2.small \
    ${CLUSTER_NAME}
```

There is no web UI by default so you'll need to install it manually:

```
$ kubectl create -f https://git.io/kube-dashboard
```

Due to the nature of the permissions system in Kubernetes you cannot access
the API/UI directory. You can instead reach it by running `kubectl proxy &`
and visiting `http://localhost:8001`.

### Setup Gitlab

Install the image:

```
$ kubectl run gitlab --image=gitlab/gitlab-ce --port=80
```

and create a service with a load balancer for external access:

```
$ kubectl expose deployment gitlab --type="LoadBalancer"
```

_Note: The hostname I was seeing in the Gitlab web interface was one that
had been auto-generated. Rather than start using my own customised config
files, I decided it was easier - in the context of this task - to SSH into
the Github host (via the Kubernetes UI) and change the value in the configs
directly._

Now install the runner image:

```
$ kubectl run gitlab-runner --image=gitlab/gitlab-runner
```

Before you can register the runner you will need to get the runner token
from the Gitlab web interface.

Now register the runner with Gitlab:

```
$ gitlab-runner register -n --url GITLAB_URL --registration-token TOKEN \
    --executor kubernetes --description "Kubernetes runner" \
    --docker-image docker:latest
```

### Configure Route 53

The services configured in this document will have public hostnames pointint to the
load balancers. To make this URL more friendly we can create aliases in Route 53.

To get a load balancer's hostname (replacing the values in capitals):

```
$ kubectl get svc --namespace NAMESPACE SERVICE_NAME -o jsonpath='{.status.loadBalancer.ingress[0].hostname}'
```

Now create an alias A record set in your hosted zone that points to the hostname
from the output of the previous command.

---

## <a name="instructions"></a>Instructions from DataQuest

The goal here is to demonstrate an understanding of some of the tools we use at Dataquest. If you don't know or understand all of them, that's ok! We spend a good portion of our time reading documentation, too.

In general, you'll want to stay pretty close to the spec, but if you feel like you can showcase your knowledge better by choosing different technology or making different architectural decisions, go for it. We'll dive into the why's and how's in a follow-up call.

If you want to dive deeper into any part of the task that interests you, or to exclude any parts of it that you are unable to complete (for any reason, including lack of knowledge or time), that's great. We know that a devops role involves knowing when to cut corners, when to go further, and how to take initiative. Of course, you'll need to provide good reasoning for these decisions, but that's part of the job, too. If you have doubts, just stick to the task as described.

### Specification

Here's the basic structure of the task:

 *  Set up a small kubernetes cluster on an AWS tier:
     *  The AWS free tier should be sufficient if you have access to it.
     *  Otherwise we can reimburse you for AWS fees.
 *  Set up a gitlab image and runners on that tier.
 *  Create a new repository in the running image and write a small Python service on it:
     *  The service doesn't need to be big, just a couple endpoints that do something.
     *  Use whatever libraries, frameworks, transports, protocols, or storage that you like.
     *  Make sure it's easy for us to test it without looking at code.
     *  Some ideas to get your imagination started:
         *  microservice with endpoint for tracking student hours on site.
         *  simple chat or todo webapp.
         *  an api for providing hints to students.
 *  Set up a gitlab pipeline that deploys the service in kubernetes.

### Advice

 *  Keep it simple. This is a big task already.
 *  The Python service is the least important part of the task.
 *  Make it super easy for us to understand, test, and demo all aspects.
 *  Document your decisions and be prepared to discuss them.
 *  Play to your strengths.

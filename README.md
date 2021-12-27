# Digital Ocean's Kubernetes Challenge

## Who am I?
I guess before explaining how the challenge went I should talk about who I am and what
I already knew before diving into the challenge.

Nice to meet you, I'm Carlos Ruiz and I'm currently an IT student. My study program has
been quite wide so I have a fair understanding of networking, system administration,
development, electronics, etc. And last year I learned about containers and their benefits
compared to regular software execution, I really felt in love with that concept.

Naturally, the next step is orchestration, I read about it a bit and had the chance to
give OpenShift (IBM's orchestrator based on Kubernetes) a try, but it was still very vague.

Luckily for me, this challenge gave me the ressources and motivation to give Kubernetes a
real try. And we even got tasks to do, the options are so broad with orchestration that 
having a clear objective is priceless.

## Some basic explanations

### Containers, what's that?
If you don't know what containers are [Docker's explanation](https://www.docker.com/resources/what-container)
is quite compact, if you want to learn how it works way more in detail, 
[ArchLinux's wiki's explanation](https://wiki.archlinux.org/title/Linux_Containers) is 
very interesting.

If you are lazy, a container of something is a neat bundle that contains everything that 
something needs to run, allowing you to download it and have the same experience on any
platform where you can have Docker.

### Orchestration, what's that?
Containers are neat and all, but they are limited to the host they are running on, it sure would be nice
if we could link hosts together, define a list of containers to run and how they interact together and
have them being distributed transparently between all the hosts.

Well that's basically what orchestration is, you can learn more about it on 
[Red Hat's explanation](https://www.redhat.com/en/topics/containers/what-is-kubernetes).

## The actual challenge

Since I'm new to kubernetes, I decided to go with one of the "easier" ones and so I'm going to explain
how the "Deploy an internal container registry" challenge went.

### Step 0: Choosing the right challenge

This may seem common sense, but before choosing any of this challenges, you need to know
what the thing you are going to deploy does and how it vagely works. 

You are going to shoot yourself in the foot if you have to both learn what a container
registry and how to make it work on kubernetes.

### Step 1: Getting to learn the options we have

DO's challenges are quite nice in that they give you some options that would fit in that
specific challenge, in this particular case Harbor and Trow were the two options given.

You can still choose something else if you prefer, but it's easier to already get those
when you have no idea what to look for.

I ended up choosing Harbor simply because it looked like it had more features and also
Trow's installation looked more boring (there's a installation script to run)

### Step 2: Understanding what parts compose the option you chose

Kubernetes "services" are most of the time more complex that just a container running,
some of it comes from kubernetes itself (for example, persistent storage needs to
be defined explicitly, there needs to be a load balancer if your service is exposed...),
but most of it actually comes from how customizable and modular those services are.

So you should dive a bit into the service's documentation to get to see what parts,
because maybe you'll want to change one of the default one with a different one that fits
better your needs (SPOILER, I did it and I'll explain it later on)

For Harbor, the main elements are (they aren't all mandatory):
- A relational database (postgres)
- A redis database
- Notary, to take care of images' signatures (to be sure an image update is trustworthy)
- Chart museum, to also be able to host Helm charts
- Trivy, to analyse the images for vulnerabilities
- Metrics, to send data to something like Prometheus

### Explanation: How to install stuff in kubernetes
You add stuff to Kubernetes by adding ressources, a ressource can be a container running
(called Pods in kubernetes) but they are mostly more abstract:

- Replica Set: Manages a group of pods to be sure there's always X pods running
- Secret: Contains some data and only shares it with the allowed ressources
- Persistent Volume Claim: Persistent storage that's reserved and can be provided to pods
- Namespace: Groups ressources together, to be able to isolate them, or delete them all for example
- Service: Defines what ports are exposed on a ressource, internally or even externally
- Ingress: Basically defines a rule for a reverse proxy (ex. redirect traffic to this service
if the hostname matches this)

To create a new ressource you have to define it as a YAML file, and for this just find
copy one from [Kubernetes documentation](https://kubernetes.io/docs) and edit the things
that need to be changed.

Once that file is created you simply have to run `kubectl apply -f <PATH_TO_FILE>` and the
ressource will be added.

It's fine if it sounds vague and weird, it'll be more clear with the actual installation.

### Extra explanation: Helm

As I said, a "service" usually will need a set of kubernetes' ressources to be functional.
It'd be nice if we could bundle all those together, have a main config file and create all
the resources from that main config file.

That's exactly what Helm is, you download a "chart" from a repository that contains the
definition of all the ressources needed and a list of "values" that are simply the settings 
to customize the ressources.

So with Helm you only need to create the config file, and with one command the whole service
will be up.

### Extra extra explanation: Helmfile

You don't really need to know this, but this is the way I'm sharing how I installed helm
charts in this repo. Helmfile allows you to define the repository charts come from and the 
values, so you just have to run `helmfile apply` to install it.

### Step 3: Creating a redis service

Harbor is installed with Helm and it already comes with a redis service, but I thought
it'd be a nice excercise to install redis separately and then connect them together.

I used bitnami's helm chart ([helmfile](manifests/redis/helmfile.yaml)) and I simply had
to choose how many replicas there would be and to use the DO's persistent storage.

### Step 4: Creating a postgres service

Same as for redis, Harbor comes with a postgres service out of the box, but I decided to
use a different one and link them together. (This also happens to complete a different
challenge: "Deploy a scalable SQL database cluster")

In this case I decided to use Kubegres, its installation is a bit different as it has no
Helm chart, but it's a nice exercise to get used to the regular Kubernetes ressources.

Kubegres requires 4 steps to be set up properly, first of all we need to import Kubegres
definitions, it sets up the kubegres controller and defines a custom resource that represents
a kubegres instance.

Then a namespace should be created to keep everything tidy. Namespaces are more easily created 
directly with `kubectl create namespace <ns_name>`, but as any other ressource they can
be defined in a file.

Then we need to create a secret that'll store the postgres credentials, this allows us to
configure the databases without having to hardcode those credentials into the files themselves.

Finally, we need to create the actual kubegres instance, this includes the amount of replicas,
the postgres image to use, the persistent volume size and the credentials.

Once all of [those files](manifests/kubegres) are created, they just have to be applied into `kubectl`.

### Explanation: Ingress

It's very common to want to make some of your services to be available from outside the cluster.
Services already allow us to expose on different IPs, but having a single IP per service isn't
practical.

It'd very nice if we could have internal services for everything and just one services publicly
available that would recieve requests and forward them to different internal services depending
on some criteria, oh that's what an ingress service is.

This is so common that there's a specific Kubernetes ressource to define those rules.

Something that should be very clear (and that caused me unnecessary headaches) is that ingresses
themselves are just rules that do nothing on their own, there needs to be a service that reads
them and actually applies them.

Since the ingress format is simply a definition, an ingress can be any webserver, as long as it's
configured to follow those rules.

### Step 5: Creating an ingress service

The most used ingress service is the nginx one, which happens to be available as a Helm chart,
so after tweaking the values a bit we can easily install it. ([helmfile](manifests/ingress/helmfile.yaml))

### Step 6: Ingress' TLS

Currently there's no excuse to be using HTTP and not HTTPS, but the nginx ingress doesn't take care
of that.

To get those certificates needed for HTTPS communication we need a certificate issuer. In this case
we are going to be using `certmanager`, it has a Helm chart as well so after setting up the values
it's quickly installed. ([helmfile](manifests/cert-manager/helmfile.yaml))

There are different ways those certificates can be obtained, so instead of setting those up with
Helm's values, it creates a new definition of ressource that represents an actual certificate
issuer, so we just have to create a [definition for that ressource](manifests/cert-manager/production-issuer.yaml).

### Step 7: DNS

Let's Encrypt (what we use to get certificates) requires a domain name to be linked to the ingress
service address, to verify that you actually own that domain name. (There are other ways to do it,
but it always requires the DNS at some level).

To do that you can use your registrar's own service, your own DNS server or DO's DNS service. I chose
DO's so that I could keep everything under the same hood.

### Final step: Creating the Harbor service

As said before, Harbor has a Helm chart, so the installation only requires to get the values and change
them to our fit,  [Harbor's helmfile](manifests/harbor/helmfile.yaml) is way bigger that the other
helmfiles so far but this is simply because of the amount of customization this chart has, I'll go
over the most important parts:
- `tls`, this defines the tls section in the generated ingresses, we set it to secret to let `certmanager` handle it.
- `ingress.annotations` this adds some settings to the generated ingresses, mostly what ingress and what issuer to use.
- `persistence.persistentVolumeClaim` defines what persistent volumes will be used, this needs to be
changed to use DO's storage class, and be at least 1GiB big.
- `imageChartStorage.s3` DO's spaces are similar to Persistent Volumes, but they are mostly meant for bigger
storage.
- `database.external` Since we are using a different we need to give helm the credentials to log in. Usually 
we'd reuse the secret that holds those credentials, but this chart doesn't allow you do that.
- `redis.external` Same thing.
- `metrics.enabled` If you want to give prometheus a try, this a very good thing to have.

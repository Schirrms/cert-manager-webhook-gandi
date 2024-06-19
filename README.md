# cert-manager-webhook-gandi

This is a completely temporary README.md. At this time, I just did a fork from sintef version of the Webhook, who was mostly a port of bwolf version.

The original README is still present, as README-orig.md

The goals for this port are (at least):

* Allow the use of a Gandi Token instead of the API KEY (requires the use of the lib gandi-go in version 0.7.0 at least)
* Correct a small cosmetic bug in the log in level 6 mode (a missing dot between the hostname and the domain)
* update to most recent dependencies
* If possible, find why this plugin is not working when more than one domain name is given for the SAN field.

For now, I just was able to add a buildjob.yaml file, allowing me to build the image in kaniko on my k3s testbed. This is heavily a Work In Progress, as, for instance, the source path is hardcoded in the yaml file. But, so far, starting the job build in 4 minutes a container image, stored in my local repository.

While really straightforward, here is the current build process (Not really ready to use make!)

~~~bash
# to start the build (I'm currently using the default namespace, as this is mostly a temporary job)
kubectl apply -f buildjob.yaml

# to see the job status
# While running...
kubectl get job,pod
NAME                                         COMPLETIONS   DURATION   AGE
job.batch/cert-manager-webhook-gandi-build   0/1           84s        84s

NAME                                         READY   STATUS      RESTARTS   AGE
pod/cert-manager-webhook-gandi-build-fpn9b   1/1     Running     0          84s

# after completion
kubectl get job,pod
NAME                                         COMPLETIONS   DURATION   AGE
job.batch/cert-manager-webhook-gandi-build   1/1           4m15s      22m

NAME                                         READY   STATUS      RESTARTS   AGE
pod/cert-manager-webhook-gandi-build-fpn9b   0/1     Completed   0          22m

# to see the actual situation (extract):
kubectl logs pod/cert-manager-webhook-gandi-build-fpn9b
INFO[0000] Resolved base name golang:1.22-alpine to base 
INFO[0000] Resolved base name base to build             
……………                
INFO[0006] COPY go.* .                                  
INFO[0006] Resolving srcs [go.*]...                     
INFO[0006] Taking snapshot of files...                  
INFO[0006] Resolving srcs [*.go]...                     
……………
INFO[0008] Running: [/bin/sh -c apk add --no-cache git ca-certificates &&     go mod download] 
fetch https://dl-cdn.alpinelinux.org/alpine/v3.20/main/x86_64/APKINDEX.tar.gz
fetch https://dl-cdn.alpinelinux.org/alpine/v3.20/community/x86_64/APKINDEX.tar.gz
(1/12) Installing brotli-libs (1.1.0-r2)
……………
(12/12) Installing git-init-template (2.45.2-r0)
Executing busybox-1.36.1-r28.trigger
OK: 20 MiB in 27 packages
INFO[0018] Taking snapshot of full filesystem...
~~~

Next step is to see if this image actually works, before I try to make change on it.

# Build a Container image and a Helm repository

To ease the use of this webhook, I created a public helm repository and i built a container's image.

You can find here some informations, should you want to build your own image, or maybe for any other project.

I choose tu use github for both image repository and helm repository. Obviously, you need a github account. I did everything with a free account.

*There are references to the version tag in both the image build and in the Helm repo. So if you plan to build another version, edit accordingly the `/buildjob.yaml` and the `/charts/cert-manager-webhook-gandi/values.yaml` files*

Disclaimer: While those explanations works for me, I'm pretty sure that those are not the smartest ways to do. Don't hesitate to upgrade and let me know.

## Building a new cert-manager-webhook-gandi image

You should find in this repository all pieces to build your own image for the gandi webhook for cert-manager.

I use this repository myself to build the image.

I'm running in a full Kubernetes / crio environment (more precisely, a k3s environment), without Docker. So the builds are done with [kaniko](https://github.com/GoogleContainerTools/kaniko)

Problem : You need a image repository to store the builded image. You can of course install a repository on your kubernetes environment, but you probably don't have at this time a valid SSL/TLS certificate!

And if you plan to share your build, you need a public storage.

I follow sintef, and choose to use [ghcr.io](https://ghcr.io) to store my build.

It's a little hard to find informations on the way to use kaniko with ghcr.io.

So there is the path that I followed (probably not the best, but it's a working path).

*I did all this build work in the default namespace,* *adjust to suit your needs*

1. create a github Personal Access Token. This token needs at least the rights:  `write:packages` and `read:packages`, and maybe also `delete:packages`

    more explanations here: [GitOps GitHub Container Registry (GHCR) integration](https://codefresh.io/docs/docs/gitops-integrations/container-registries/github-cr/#:~:text=The%20GitHub%20Container%20registry%20allows,image%20independent%20from%20any%20repository.) and here: [Managing your personal access tokens](https://docs.github.com/en/authentication/keeping-your-account-and-data-secure/managing-your-personal-access-tokens)

2. Using this token, create a secret in Kubernetes:

    ~~~bash
    kubectl create secret docker-registry ghcrcred     
        --docker-server=https://ghcr.io/    
        --docker-username={Your GitHub Username, case sensitive]     
        --docker-password={Your personal access token}
        --docker-email={your E-Mail}
    ~~~

    Remember to protect this token as any other sensitive password.

3. create the job yaml file to actually build the image. Here is a model, adapt as needed.

    ~~~yaml
    apiVersion: batch/v1
    kind: Job
    metadata:
      name: cert-manager-webhook-gandi-build-{Build version}
    spec:
      template:
        spec:
          containers:
          - image: gcr.io/kaniko-project/executor:latest
            args: 
            - "--dockerfile=/tmp/build/Dockerfile"
            - "--context=dir:///tmp/build"
            - "--destination=ghcr.io/{Github username in lowercase}/cert-manager-webhook-gandi:{Build version}"
            name: cert-manager-webhook-gandi
            resources: {}
            volumeMounts:
            - name: build-tools-volume
              mountPath: /tmp/build
            - name: kaniko-secret
              mountPath: /kaniko/.docker
          volumes:
            - name: build-tools-volume
              hostPath: 
                path: /work/git/cert-manager-webhook-gandi
            - name: kaniko-secret
              secret:
                secretName: ghcrcred
                items:
                  - key: .dockerconfigjson
                    path: config.json
          restartPolicy: Never
    status: {}
    ~~~~

4. Start the job, and check the result

    ~~~bash
    $ kubectl apply -f buildjob.yaml
    job.batch/cert-manager-webhook-gandi-build-0-4-8 created

    $ kubectl get jobs,pods
    NAME                                               STATUS     COMPLETIONS   DURATION   AGE
    job.batch/cert-manager-webhook-gandi-build-0-4-8   Running    0/1           4s         4s

    NAME                                               READY   STATUS      RESTARTS     AGE
    pod/cert-manager-webhook-gandi-build-0-4-8-5mc4m   1/1     Running     0            4s

    $ kubectl logs pod/cert-manager-webhook-gandi-build-0-4-8-5mc4m -f
    INFO[0000] Resolved base name golang:1.22-alpine to base
    INFO[0000] Resolved base name base to build
    INFO[0000] Resolved base name scratch to image
    ………………
    INFO[0269] Pushing image to ghcr.io/schirrms/cert-manager-webhook-gandi:0.4.8
    INFO[0272] Pushed ghcr.io/schirrms/cert-manager-webhook-gandi@sha256:8cad02fede73e8c781b7752ec2bcd1c5727e7b1d9970756b9414c1f23a16b436
    ~~~

Done!

## Creating a helm repo, and uploading the Helm chart

This is basically me using sintef work in the charts folder, and following this great article: [How to make and share your own Helm package](https://medium.com/containerum/how-to-make-and-share-your-own-helm-package-50ae40f6c221)

I'll try to put here all informations in a direct useable form, but the link above is much more interesting.

You have to create on github a repository named `{your username in lowercase}.github.io` . This repository is like any other repository, but the url will be `https://{your github username in lowercase}.github.io`. You can see this repository as a web site of sort.

Clone the repository, and copy the cert-manager-webhook-gandi charts subfolder in this new repository.

Do all need adjustments. You should only have to put changes in the `values.yaml` file.

touch cert-manager-webhook-gandi/index.yaml

git commit and push

go in the root directory of the git repository

build the package:

~~~bash
helm package cert-manager-webhook-gandi
~~~

Move the package in the `cert-manager-webhook-gandi` directory:

~~~bash
mv cert-manager-webhook-gandi-v0.4.8.tgz cert-manager-webhook-gandi/
~~~

Create the index file for the repository:

~~~bash
helm repo index cert-manager-webhook-gandi --url https://{your github username in lowercase}.github.io/cert-manager-webhook-gandi/
~~~

The file index.yaml is now ready to use

then, git commit / push

The repository should be up & running:

~~~bash
$ helm repo add cert-manager-webhook-gandi https://{your github username in lowercase}s.github.io/cert-manager-webhook-gandi
"cert-manager-webhook-gandi" has been added to your repositories

$ helm repo update
Hang tight while we grab the latest from your chart repositories...
...Successfully got an update from the "cert-manager-webhook-gandi" chart repository
Update Complete. ⎈Happy Helming!⎈

$ helm search repo  cert-manager-webhook-gandi
NAME                                                    CHART VERSION   APP VERSION     DESCRIPTION
cert-manager-webhook-gandi/cert-manager-webhook...      v0.4.8          0.4.8           A Helm chart for cert-manager-webhook-gandi
~~~

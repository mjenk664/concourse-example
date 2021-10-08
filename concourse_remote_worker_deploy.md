# Deploy External Concourse Workers - Helm

This guide demonstrates how to deploy an external concourse worker into a Kubernetes cluster using Helm.

## Pre-requisites

- Helm
- 1 K8s cluster running the Concourse web, postgres, and worker Pods
- 1 K8s cluster to deploy the Concourse external worker Pods
- Port 2222 must be accessible between the External Worker and Concourse Web

## External Worker Communication

In Concourse, external workers runs a process called *Beacon*. Beacon handshakes with TSA (running on the Web Pod) and tells it to open a reverse SSH tunnel for ongoing communication between the ATC and the worker.

You can read more here about the configuration: https://docs.pivotal.io/p-concourse/v6/architecture/#external-worker-communications

---

## Step 1: Generate an SSH key for the External Worker

First, we will need to generate a SSH key pair for the external worker.

You can generate the key pair by following either:

**Concourse Binary**

```
docker run -v $PWD:/keys --rm -it concourse/concourse generate-key -t ssh -f /keys/external-worker-key
```

**ssh-keygen**
```
ssh-keygen -t rsa -f external-worker-key  -N '' -m PEM
```

These key pairs will be used to configure the external worker. Later, we will add the public key as an `authorized_keys` to the TSA running on the web pod. This will allow the TSA trust the connection from the external worker.

---

## Step 2: Update the values YAML for the Concourse Deploment

The Concourse Web Pod must be contain the public key for the external worker that was generated above in **Step 1**.

Additionally, the `web.WorkerGateway.type` is set to `ClusterIP` by default. This means that the service will be deployed and listen to communications only internal to the cluster. We must set this value to `LoadBalancer` to expose the service externally to the cluster.

You will need to update your `values.yaml` to include the following configuration:
```yaml
web:
  workerGateway:
    type: LoadBalancer
    ## Optionally, specify a static IP for the Load Balancer
    loadBalancerIP:
secrets:
  ## Array of team names and public keys for team external workers. A single
  ## team can have many keys defined in the key field.
  ##
  ## Example:
  ## - team: main
  ##   key: |-
  ##     ssh-rsa AAAAB3NzaC1yc2EAAAADAQABAAABAQDYBQ9fG6IML+qs....
  ##     ssh-rsa AAAAB3NzaC1yc2EAAAADAQABAAACAQDzpK/sIOtL9SCj....
  ##
  ## Make sure to chack the security caveats here: https://concourse-ci.org/teams-caveats.html
  ## Extra Reads: https://github.com/concourse/concourse/issues/1865#issuecomment-464166994
  ## https://concourse-ci.org/global-resources.html#complications-with-reusing-containers
  ##
  teamAuthorizedKeys:
    - team: tanzu-remote
      key: |-
        ssh-rsa AAAAB384o3dfpgza....
```

For each team you have created in Concourse, you can add multiple SSH public keys for each external worker. Once you have updated the configuration, we will need to deploy Concourse in the 1st cluster.

## Step 3: Deploy the Concourse chart using Helm

---

1. Login to your first K8s cluster where you want to deploy Concourse (this may vary depending upon your platform)

2. Deploy the chart

   ```
   $ helm repo add concourse https://concourse-charts.storage.googleapis.com/
   $ helm install concourse/concourse -f values.yaml
   ```

3. Validate that the pods are running by performing:
   ```
   ➜ kubectl get pods   
   NAME                             READY   STATUS    RESTARTS   AGE
   concourse-postgresql-0           1/1     Running   0          55s
   concourse-web-84fb54544f-jq8v7   1/1     Running   0          55s
   concourse-worker-0               1/1     Running   0          55s
   ```
---

## Step 4: Obtain the IP Address for the Worker Gateway Service

Before deploying the external worker into the 2nd Kubernetes cluster, we will need to obtain the IP Address for the gateway service that the extenral worker will communicate through.

As mentioned above, a Reverse SSH tunnel will get created between the TSA (web) and Beacon (worker) over port 2222

Record the EXTERNAL-IP address for `concourse-web-worker-gateway` by running:

```
➜ kubectl get services                 
NAME                            TYPE           CLUSTER-IP     EXTERNAL-IP     PORT(S)                        AGE
concourse-postgresql            ClusterIP      10.0.192.112   <none>          5432/TCP                       10m
concourse-postgresql-headless   ClusterIP      None           <none>          5432/TCP                       10m
concourse-web                   LoadBalancer   10.0.41.221    20.98.188.132   8080:30061/TCP,443:31511/TCP   10m
concourse-web-worker-gateway    LoadBalancer   10.0.220.232   20.98.188.155   2222:31778/TCP                 10m
concourse-worker                ClusterIP      None           <none>          <none>                         10m
kubernetes                      ClusterIP      10.0.0.1       <none>          443/TCP                        10d
```

---

Record the IP address of the concourse-worker-gateway Service. It will be used in the next step

## Step 5: Create a values YAML for the External Worker:

Using the editor of your choice, create a file called `external_worker_values.yaml` and copy the following configuration:

```yaml
web:
  enabled: false
worker:
  enabled: true
persistence:
  enabled: true
  worker:
    storageClass:
postgresql:
  enabled: false
concourse:
  worker:
    ## The name to set for the worker during registration. 
    ## If not specified, the hostname will be used.
    ##
    name: <name-of-external-worker>

    ## A tag to set during registration. Can be specified multiple times.
    ##
    tag: <tag-to-assign-external-worker>

    ## The name of the team that this worker will be assigned to.
    ##
    team: <team-name-to-assign-worker>

    tsa:

      ## TSA host(s) to forward the worker through.
      ## Only used for worker-only deployments.
      ## Example:
      ## hosts:
      ##   - 1.1.1.1:2222
      ##   - 2.2.2.2:2222
      ##
      hosts:
        - <IP_of_Worker_Gateway_Service>:2222
secrets:
  workerKey: |-
    -----BEGIN RSA PRIVATE KEY-----
    mg03mxS.....
    -----END RSA PRIVATE KEY-----
  workerKeyPub: |-
    ssh-rsa AAAAB384o3dfpgza....
```

Ensure you update the `concourse.worker.tsa.hosts` parameter to include the `concourse-web-worker-gateway` service's EXTERNAL-IP address that was recorded from Step 4. Without this, the deployment of the external worker will fail.

Additionally, when deploying the external workers the Concourse Web and Postgres Pods are disabled. You must set the `enabled` flags to `false` in your external values YAML configuration file.

---

## Step 6: Deploy the External Workers

1. Login to the 2nd kubernetes cluster where you want to deploy your external workers (this may vary depending upon your platform)

2. Deploy the chart using the `external_worker_values.yaml` that you created in Step 5.

    ```
    $ helm install concourse/concourse -f external_worker_values.yaml
    ```

3. Validate that the external worker pods are running by performing:
   ```
   ➜ kubectl get pods
   NAME                 READY   STATUS    RESTARTS   AGE
   concourse-worker-0   1/1     Running   0          11s
   concourse-worker-1   1/1     Running   0          81s
   ```

---

## Step 7: Create the Team in Concourse

Before running a pipeline in Concourse, we will create the Team that we have assigned to our external workers.

To create a team, perform the following:

1. Login to Concourse using the `fly` CLI:

   ```
   fly -t <target> login -c <concourse_url> -u <username>
   ```

2. Create the team(s) specified in your `external_workers_values.yaml` and `values.yaml`

   ```
   fly -t <target> set-team -n <team-name> --local-user=<username>
   ```

3. Enter (Y) to accept the changes. You should now be able to login to this team using fly:

   ```
   fly -t <target> login -c <concourse_url> -u <username> -n <team_name>
   ```

---

## Step 8: Create a sample pipeline using the team/tag for the external worker

Let's validate that we can run pipelines using the external workers.

You can use the below sample to test running a pipeline on the external worker:

```yaml
jobs:
- name: hello-world
  public: true
  plan:
  - task: hello-world
    tags:
    - tanzu-remote
    config:
      platform: linux
      image_resource:
        type: docker-image
        source:
          repository: ubuntu
      run:
        path: /bin/sh
        args:
        - -c
        - |
          echo "Hello World!"
```

Save the above contents into a `pipeline.yaml` and set the pipeline using fly:

```
fly -t <target> set-pipeline -p hello-world -c pipeline.yaml
                           
jobs:
  job show-conjur-secret has been added:
+ name: hello-world
+ plan:
+ - config:
+     image_resource:
+       name: ""
+       source:
+         repository: ubuntu
+       type: docker-image
+     platform: linux
+     run:
+       args:
+       - -c
+       - |
+         echo "Hello World!"
+       path: /bin/sh
+   tags:
+   - tanzu-remote
+   task: hello-world
+ public: true
  
pipeline name: hello-world

apply configuration? [yN]: y
pipeline created!
```

---

## Step 9: Run the pipeline

Execute the pipeline by logging into the Concourse UI or using fly:

```
fly -t <target> trigger-job -j <pipeline-name>/<job-name>
```

You should see the selected worker name pointing to the external worker that was deployed.
# Workspace for running on-site workshops

This project is a cloud based workspace for running workshop for Pivotal Platform. It sets up IDE for each participant.

## Requirement

- **Instrucator**
  - PKS Environment
    - Privileged Plan
  - kubectl
  - helm
- **Participants**
  - Browser (preferrably Chrome or Chromium based)
  - Passion to learn

# Configure PKS environment

1. Setup a PKS cluster.

   > I used Toolsmith to setup a cluster quickly on GCP.

   1. Create a plan `small-privileged` with:
      - 1 x Master (medium.disk)
      - 3-50 x Workers (medium.disk)
      - Enabled Privileges Access

# Setup Workshop Cluster

1. Login on PKS

   ```
   pks login -a api.<pks-domain> -u admin -p <password from opsman>
   ```

1. Provision cluster with `manage-pks`

   ```
   manage-cluster provision
   ```

   - Name: `workspace`
   - Plan: `small-privileged`
   - Managed Zone: _`Choose one of the lister zones`_

     (**Example:** run-on-cf-zone)

1. After cluster is created successfully, enable access to cluster

   ```
   manage-cluser acccess
   ```

   - Name: `workspace`

# Configure workspace cluster

1. Get credentials and verify that you are connected to the workspace cluster. **Note** that if you have used `manage-cluster`command, this is already done

   - Get credentials

     ```
     pks get-credentials workspace
     ```

   - Check connection

     ```
     kubect cluster-info
     ```

1. Setup Helm

   1. You should have `helm` cli install. You can follow instructions [here][helm-install] to install for you platform

   1. Setup Helm rbac permission

      ```
      kubectl create sa  tiller -n kube-system
      kubectl create clusterrolebinding  tiller  --clusterrole=cluster-admin --serviceaccount=kube-system:tiller
      ```

   1. Install helm

      ```
      helm init --service-account tiller --upgrade
      ```

1. Setup NGIX Ingress (based on [community doc][pks-nginx] )

   1. Install NGIX Ingress cotroller

      ```
      helm install stable/nginx-ingress \
          --name nginx \
          --set rbac.create=true \
          --namespace ingress \
          --set controller.config.proxy-buffer-size=16k

      ```

   1. Get IP for ingress controller, might be few minutes before the setup is complete.

      ```
      NGINX_IP=$(kubectl get service/nginx-nginx-ingress-controller -n ingress -o jsonpath="{.status.loadBalancer.ingress[0].ip}")
      ```

   1. Setup a DNS name for workspaces ingress resources and point to this IP

      ```
      ZONE_NAME=run-on-cf-zone
      WORKSPACE_DOMAIN=workspace.run-on.cf
      gcloud dns record-sets transaction start --zone=$ZONE_NAME

      gcloud dns record-sets transaction add $NGINX_IP --name=\*.$WORKSPACE_DOMAIN. --ttl=300 --type=A --zone=$ZONE_NAME

      gcloud dns record-sets transaction add $NGINX_IP --name=$WORKSPACE_DOMAIN. --ttl=300 --type=A --zone=$ZONE_NAME

      gcloud dns record-sets transaction execute --zone=$ZONE_NAME

      ```

      > Update Change `ZONE_NAME` and `WORKSPACE_DOMAIN` as per your config

1. Prepare SSL certs for the workspace

   1. I use certbot and a small wrapper script on it to quickly generate certificate to do this effortlessly.

      ```
      certs.sh g gcp $WORKSPACE_DOMAIN,\*.$WORKSPACE_DOMAIN
      ```

   1. Copy key to `ssl/privkey.pem`

      ```
      cp path-to/privkey.pem $PROJECT_ROOT/ssl/privkey.pem
      ```

   1. Copy certificatew to `ssl/cert.pem`

      ```
      cp path-to/cert.pem $PROJECT_ROOT/ssl/cert.pem
      ```

# Create Workspace for each user

There are multiple object that need to be setup for each workspace for each user. So I am using a helm template to easily create all at one go.

Every user gets its own namespace, service account, secrets, deployment, etc.

Command to create each user's workspace is:

```
helm template \
   theia-on-kubernetes/theia-k8s-helm \
   --set user=username \
   --set password=password \
   --set namespace=username \
   --set-file ingressKey=ssl/privkey.pem \
   --set-file ingressCrt=ssl/cert.pem \
   | kubectl apply -f -
```

Replace `username`, `password`, `namespace`, `key` and `cert` with correct values

## Appendix

1. Masters Firewall Rule

   ```
   gcloud compute \
       firewall-rules create \
       chino-k8s-masters \
       --description="Allow k8s api traffic to master" \
       --direction=INGRESS \
       --priority=1000 \
       --network=chino-pcf-network \
       --action=ALLOW \
       --rules=tcp:8443 \
       --source-ranges=0.0.0.0/0 \
       --target-tags=master
   ```

1. Worker Firewall Rule

   ```
   gcloud compute --project=pa-yrampuria \
       firewall-rules create \
       chino-k8s-workers \
       --description="Allow node port traffic to workers" \
       --direction=INGRESS \
       --priority=1000 \
       --network=chino-pcf-network \
       --action=ALLOW \
       --rules=tcp:30000-32767,udp:30000-32767 \
       --source-ranges=0.0.0.0/0 \
       --target-tags=worker
   ```

[che-doc]: https://che.eclipse.org/eclipse-che-7-is-now-available-40ae07120b38
[pks-ingress]: https://community.pivotal.io/s/article/how-to-set-up-an-ingress-controller-for-a-pks-cluster
[helm-install]: https://helm.sh/docs/intro/

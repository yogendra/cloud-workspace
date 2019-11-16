# cloud-workspace

Cloud based workspace for demo, handson labs, etc

## Requirement

- Kubernetes Cluster
- kubectl
- helm

# Setup steps

1. Create Global IP
1. Create a wildcard dns name for your workshop/event. Example: `*.workshop.runs-on.cf` and point to IP
1. For each user run following command

   ```
   export USERS=3
   for USER_ID in {1..$USERS}
   do
    USER=user_$USER_ID
    PASSWORD=keepitsimple
    helm template . --set user=$USER --set password=$PASSWORD --set namespace=$USER --set domain=workshop-jakarta.runs-on.cf | kubectl apply -f -
   done
   ```

# Create PKS environment

I used Toolsmith to setup a cluster quickly on GCP.

# Setup Workshop Cluster

1. Login on PKS

   ```
   pks login -a api.<pks-domain> -u admin -p <password from opsman>
   ```

1. Provision cluster with `manage-pks`

   ```
   manage-cluster provision
   ```

   - Name: workshop
   - Plan: small
   - Managed Zone: _choose forom one that is present_

1. After cluster is created successfully, enable access to cluster

   ```
   manage-cluser acccess
   ```

   - Name: workshop

1. Setup Helm

   1. Setup Helm rbac permission

      ```
      kubectl create sa  tiller -n kube-system
      kubectl create clusterrolebinding  tiller  --clusterrole=cluster-admin --serviceaccount=kube-system:tiller
      ```

   1. Install helm

      ```
      helm init --service-account tiller
      ```

1. Setup NGIX Ingress (based on [community doc](pks-nginx) )

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
      export NGINX_IP=$(kubectl get service/nginx-nginx-ingress-controller -n ingress -o jsonpath="{.status.loadBalancer.ingress[0].ip}")
      ```

   1. Setup a DNS name for workspaces ingress resources and point to this IP

      ```
      gcloud dns record-sets transaction start --zone=run-on-cf-zone

      gcloud dns record-sets transaction add $NGINX_IP --name=\*.workshop.run-on.cf. --ttl=300 --type=A --zone=run-on-cf-zone

      gcloud dns record-sets transaction execute --zone=run-on-cf-zone

      ```

1. Prepare SSL certs for the workshop

   1. I user `certs.sh` + certbot to do this automatically

      ```
      ./certs.sh g gcp workshop.run-on.cf,\*.workshop.run-on.cf
      ```

   1. Copy certificate to `ssl/` directory

   1. Create secret with ssl Certificates

      ```

      kubectl create secret tls ingress-cert --key=ssl/privkey.pem --cert=ssl/cert.pem 
      ```

# Configure

[che-doc]: https://che.eclipse.org/eclipse-che-7-is-now-available-40ae07120b38
[pks-ingress]: https://community.pivotal.io/s/article/how-to-set-up-an-ingress-controller-for-a-pks-cluster

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

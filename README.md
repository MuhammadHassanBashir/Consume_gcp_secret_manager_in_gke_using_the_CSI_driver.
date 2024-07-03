# Consume_gcp_secret_manager_in_gke
We can consume gcp secret stored in gcp secret manager with gke deployment or pods..

This repo explains how you can use the Secret Manager add-on to access the secrets stored in Secret Manager as volumes mounted in Kubernetes Pods.

This process involves the following steps:

    1-Install the Secret Manager add-on on a new or existing GKE cluster.
    2-Configure applications to authenticate to the Secret Manager API.
    3-Define which secrets to mount onto Kubernetes Pods using a SecretProviderClass YAML file.
    4-Create a volume where the secrets will be mounted. After the volume is attached, applications in the container can access the data in the container file system.

  The Secret Manager add-on is derived from the open source Kubernetes Secrets Store CSI Driver and the Google Secret Manager provider. If you're using the open source Secrets Store CSI Driver to access secrets, you can migrate to the Secret Manager add-on. For information, see Migrate from the existing Secrets Store CSI Driver.

**Steps:**
  **1-Enable the Secret Manager and Google Kubernetes Engine APIs.**
  
      gcloud services enable secretmanager.googleapis.com
  
  **2- Install the Secret Manager add-on.**
            
    You can install the Secret Manager add-on on both Standard clusters as well as Autopilot clusters. **Ensure that Workload Identity Federation for GKE is enabled on the Standard cluster(for this go through gke cluster console and enable ***workload identity**).** Workload Identity Federation for GKE is enabled by default on an Autopilot cluster. Kubernetes Pods use workload identity to authenticate to the Secret Manager API.
              
  **now install secret manager add-on on exiting cluster using below commands**
     
     gcloud beta container clusters update disearch-cluster \
        --enable-secret-manager \
        --location=us-central1-c 
  3- Configure applications to authenticate to the Secret Manager API
  **To allow your applications to authenticate to the Secret Manager API using Workload Identity Federation for GKE, follow these steps:**

    1- Create a new Kubernetes ServiceAccount or use an existing Kubernetes ServiceAccount in any namespace, including the default Kubernetes ServiceAccount.
     
    service account:
    apiVersion: v1
    kind: ServiceAccount
    metadata:
      name: disearch-serviceaccount
      namespace: airflow   
   
    Apply:
   
    kubectl apply -f service-account.yaml

    Verfication:
    
    kubectl get sa -n airflow

    2- Create an Identity and Access Management (IAM) allow policy for the secret in Secret Manager. 
    
    Create an IAM allow policy that references the new Kubernetes ServiceAccount and grant it permission to access the secret:
    
    gcloud secrets add-iam-policy-binding DB_HOST \
    --role=roles/secretmanager.secretAccessor \
    --member=principal://iam.googleapis.com/projects/22927148231/locations/global/workloadIdentityPools/world-learning-400909.svc.id.goog/subject/ns/airflow/sa/disearch-serviceaccount 

    repeat this step for accessing each secret stored in gcp secret manager to use in gke deployment.

    gcloud secrets add-iam-policy-binding DB_User \
    --role=roles/secretmanager.secretAccessor \
    --member=principal://iam.googleapis.com/projects/22927148231/locations/global/workloadIdentityPools/world-learning-400909.svc.id.goog/subject/ns/airflow/sa/disearch-serviceaccount 

    gcloud secrets add-iam-policy-binding DB_PASSWORD \
    --role=roles/secretmanager.secretAccessor \
    --member=principal://iam.googleapis.com/projects/22927148231/locations/global/workloadIdentityPools/world-learning-400909.svc.id.goog/subject/ns/airflow/sa/disearch-serviceaccount 

  4- Define which secrets to mount

    apiVersion: secrets-store.csi.x-k8s.io/v1
    kind: SecretProviderClass
    metadata:
      name: disearch-secret-provider-class
      namespace: airflow
    spec:
      provider: gcp
      parameters:
        secrets: |
          - resourceName: "projects/world-learning-400909/secrets/DB_HOST/versions/latest"
            path: "db_host"
          - resourceName: "projects/world-learning-400909/secrets/DB_PASSWORD/versions/latest"
            path: "db_password"
          - resourceName: "projects/world-learning-400909/secrets/DB_USER/versions/latest"
            path: "db_user"  

    verification:

    kubectl get secretproviderclass -n airflow

    verify IAM policy bind members

    gcloud secrets get-iam-policy DB_HOST --project=714482271007
    gcloud secrets get-iam-policy DB_PASSWORD --project=714482271007
    gcloud secrets get-iam-policy DB_USER --project=714482271007
   
  5-  Configure a volume where the secrets will be mounted
  
    apiVersion: apps/v1
    kind: Deployment
    metadata:
      name: nginx-deployment
      namespace: airflow
    spec:
      replicas: 1
      selector:
        matchLabels:
          app: nginx
      template:
        metadata:
          labels:
            app: nginx
        spec:
          serviceAccountName: disearch-serviceaccount
          initContainers:
          - name: init-secrets
            image: busybox
            command: ["/bin/sh", "-c"]
            args:
              - |
                export DB_HOST=$(cat /var/secrets/db_host)
                export DB_PASSWORD=$(cat /var/secrets/db_password)
                export DB_USER=$(cat /var/secrets/db_user)
                echo "DB_HOST=${DB_HOST}" > /etc/nginx/env.conf
                echo "DB_PASSWORD=${DB_PASSWORD}" >> /etc/nginx/env.conf
                echo "DB_USER=${DB_USER}" >> /etc/nginx/env.conf
            volumeMounts:
              - mountPath: "/var/secrets"
                name: mysecret
              - mountPath: "/etc/nginx"
                name: env-config
          containers:
          - name: nginx
            image: nginx:latest
            ports:
            - containerPort: 80
            envFrom:
              - configMapRef:
                  name: nginx-env
            volumeMounts:
              - mountPath: "/var/secrets"
                name: mysecret
          volumes:
          - name: mysecret
            csi:
              driver: secrets-store.csi.k8s.io
              readOnly: true
              volumeAttributes:
                secretProviderClass: "disearch-secret-provider-class"
          - name: env-config
            emptyDir: {}
    
    ---
    
    apiVersion: v1
    kind: Service
    metadata:
      name: nginx-service
      namespace: airflow
    spec:
      selector:
        app: nginx
      ports:
        - protocol: TCP
          port: 80
          targetPort: 80
      type: LoadBalancer
         
    ---

    apiVersion: v1
    kind: ConfigMap
    metadata:
      name: nginx-env
      namespace: airflow
    data:
      env.conf: |
        # This file is generated by the init container and will be sourced by Nginx.
        include /etc/nginx/env.conf;

  6- Fetch secret Verfication(from gcp secret manager to gke)

    Exec into the Pod to check mounted secrets:
    
    kubectl exec -it <pod-name> -n airflow -- /bin/sh
    ls /var/secrets
    cat /var/secrets/db_host
    cat /var/secrets/db_password
    cat /var/secrets/db_user

    here you can see your fetched gcp secret in gke pod.
   


   

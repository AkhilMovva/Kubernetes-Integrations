# GKE Secret Manager. Consuming secrets using external Secrets

## Prepare the environment

If you already have a GKE cluster with [Workload Identity](https://cloud.google.com/kubernetes-engine/docs/how-to/workload-identity) enabled and configured to pull a secret from GSM, you can skip this step.

Otherwise, start by preparing the environment using these [instructions](./README.md). ONLY follow the [Prepare Env](https://github.com/boredabdel/gke-secret-manager#prepare-environment), [Create Cluster](https://github.com/boredabdel/gke-secret-manager#create-cluster) and [Fetch Credentials](https://github.com/boredabdel/gke-secret-manager#fetch-credentials-for-the-cluster) steps. After you are done come back here.

Create a Google Service Account (GSA), this SA will be used by External-Secrets to access GSM

```bash
gcloud iam service-accounts create secret-gsa --project ${PROJECT_ID}

gcloud iam service-accounts create secret-gsa --project gke-practice-341914
```

## Installing external secrets

You will need [helm](https://helm.sh/) to install [External Secrets](https://github.com/external-secrets/kubernetes-external-secrets#how-to-use-it).

The command below will deploy the CRD and controller. It creates a Kubernetes Service Account (KSA) called ```secret-ksa``` and annotate it with the GSA for Workload Identity to work.

```bash
$ helm repo add k8s-external-secrets https://external-secrets.github.io/kubernetes-external-secrets/

$ helm install k8s-external-secrets k8s-external-secrets/kubernetes-external-secrets \
    --set serviceAccount.annotations."iam\.gke\.io/gcp-service-account"='secret-gsa@'"${PROJECT_ID}"'.iam.gserviceaccount.com' \
    --set serviceAccount.create=true \
    --set serviceAccount.name="secret-ksa"

# helm repo add external-secrets https://charts.external-secrets.io
# helm install external-secrets/external-secrets

helm install k8s-external-secrets k8s-external-secrets/kubernetes-external-secrets \
    --set serviceAccount.annotations."iam\.gke\.io/gcp-service-account"='secret-gsa@gke-practice-341914.iam.gserviceaccount.com' \
    --set serviceAccount.create=true \
    --set serviceAccount.name="secret-ksa"

helm uninstall k8s-external-secrets
```

Verify that controller and CRDs have been installed

```bash
kubectl get po -l app.kubernetes.io/name=k8s-external-secrets
```

You should see one pod

```bash
NAME                                                            READY   STATUS    RESTARTS   AGE
k8s-external-secrets-kubernetes-external-secrets-694f98f7bhbxgp   1/1     Running       0          20m
```

```bash
kubectl get crd | grep external
```

You should see 1 crd

```bash
externalsecrets.kubernetes-client.io                        2021-11-18T10:53:05Z
```

## Configuring External Secrets

Create a Secret in GSM. NB: External Secrets requires secrets to be in JSON format, we will explain why later.

```bash
echo -n '{"value": "my-secret-value"}' | gcloud secrets create my-secret --replication-policy="automatic" --data-file=-
```

Grant the GSA access to Secrets. NB: We grant access to ALL Secrets in the project as External Secrets doesn't support fine grain access, the controller in the cluster would access secrets in GSM on behalf of all workloads that needs one.

```bash
$ gcloud config set project $PROJECT_ID
gcloud config set project gke-practice-341914

gcloud projects add-iam-policy-binding ${PROJECT_ID} \
    --member=serviceAccount:secret-gsa@${PROJECT_ID}.iam.gserviceaccount.com \
    --role=roles/secretmanager.secretAccessor

gcloud projects add-iam-policy-binding gke-practice-341914 `
    --member=serviceAccount:secret-gsa@gke-practice-341914.iam.gserviceaccount.com `
    --role=roles/secretmanager.secretAccessor
```

Grant the External Secrets KSA the permission to impresonate the GSA

```bash
gcloud iam service-accounts add-iam-policy-binding secret-gsa@${PROJECT_ID}.iam.gserviceaccount.com \
    --role roles/iam.workloadIdentityUser \
    --member "serviceAccount:${PROJECT_ID}.svc.id.goog[default/secret-ksa]"

gcloud iam service-accounts add-iam-policy-binding secret-gsa@gke-practice-341914.iam.gserviceaccount.com `
    --role roles/iam.workloadIdentityUser `
    --member "serviceAccount:gke-practice-341914.svc.id.goog[default/secret-ksa]"
```

## Deploy the ExternalSecret Object

```yaml
apiVersion: kubernetes-client.io/v1
kind: ExternalSecret
metadata:
  name: my-gcp-secret #name of the External Secret and the generate k8s secret
spec:
  backendType: gcpSecretsManager
  projectId: gke-practice-341914
  data:
    - key: my-secret # name of the GCP secret
      version: latest # version of the GCP secret
      property: value # name of the field in the GCP secret
      name: secret-key # key name in the k8s secret
```

## Deploy the Deployment using as normal secret

```yaml
      containers:
      - name: nginx
        image: nginx:1.14.2
        ports:
        - containerPort: 80
        env:
        - name: API_TOKEN
          valueFrom:
            secretKeyRef:
              name: my-gcp-secret
              key: secret-key
```

## Updating secret in GSM

ExternalSecret automatically syncs the secret in k8s.
Deployment needs to be updated.

## Links

- Git - [kubernetes-external-secrets](https://github.com/external-secrets/kubernetes-external-secrets#how-to-use-it/)
- Git - [boredabdel](https://github.com/boredabdel/gke-secret-manager/tree/main/hello-secret-external-secrets)
- Git - [external-secrets](https://github.com/external-secrets/external-secrets)
- external-secrets - [provider-google-secrets-manager](https://external-secrets.io/provider-google-secrets-manager/)

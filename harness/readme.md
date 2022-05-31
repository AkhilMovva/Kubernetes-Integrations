# Harness

## Start Harness CD

Start by cloning this repo.

```shell
git clone https://github.com/harness/harness-cd-community.git
cd harness-cd-community/helm
```

Start Harness CD using the helm-chart

```shell
helm install harness ./harness --create-namespace --namespace harness
```

## Use Harness CD

Follow the notes printed by Helm to access the application.
You should wait for the harness-cd application to start before moving to Step 2.

```shell
 kubectl wait --namespace harness --timeout 900s --selector app=proxy --for condition=Ready pods
```

You can also use the following commands to check status.

```shell
kubectl get pods -n harness
kubectl get services -n harness
```

1) Open the link which is displayed and complete the registration form at `<URL>/signup`. Now your Harness CD account along with the first (admin) user is created. If you have already completed this step, then login to Harness CD at `<URL>/signin`.
2) Follow the Harness CD Community Edition [quickstart](https://ngdocs.harness.io/article/ltvkgcwpum-harness-community-edition-quickstart)

## Troubleshooting

If you run into issues when installing Harness this section will help identify where the issue is.

### View running processes

```shell
kubectl get pods -n harness
```

### View logs of processes

```shell
kubectl logs -n harness -f <POD-NAME>
```

## Support

[Join the Harness Community Slack](https://join.slack.com/t/harnesscommunity/shared_invite/zt-y4hdqh7p-RVuEQyIl5Hcx4Ck8VCvzBw)
[Join the Harness Community Forum](https://community.harness.io/)

## Profiles

Harness CD Community Edition supports multiple hardware profiles. The default profile is `laptop` for low resource environments. There is also a `production` profile available for use in more demanding environments.

To run the `production` profile use this startup command

```shell
helm install -f harness/values-production.yaml harness ./harness --create-namespace --namespace harness
```

## Remove Harness CD

```shell
helm uninstall harness -n harness
```

You can even delete the harness namespace altogether.

```shell
kubectl delete namespace harness
```

harness-delegate.yaml

```shell
kubectl apply -f harness-delegate.yaml
kubectl get pods -n harness-delegate-ng
```

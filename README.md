These steps show how to deploy Redpanda with the helm chart, configured to use a single traefik LoadBalancer service to handle TCP and HTTP traffic to multiple brokers. All traffic is encrypted by TLS using certificates pulled from a secret.

## Prerequisites

The following must be available:

- [kubectl](https://kubernetes.io/docs/tasks/tools/)
- [helm](https://github.com/helm/helm/releases)
- [rpk](https://docs.redpanda.com/current/get-started/rpk-install/)

Add the Redpanda helm repo:

```
helm repo add redpanda https://charts.redpanda.com
helm repo update
```

A Kubernetes cluster must be accessible in your current context, and it must have the ability to provide an external IP to a LoadBalancer resource. Clusters running on cloud providers (AKS, EKS, etc.) are already enabled, but with local clusters you will need to follow instructions similar to the following:

```
kubectl apply -f https://raw.githubusercontent.com/metallb/metallb/v0.13.10/config/manifests/metallb-native.yaml
kubectl wait --namespace metallb-system --for=condition=ready pod --selector=app=metallb --timeout=90s
kubectl apply -f metallb-config.yaml
```

You must have certificates available. The instructions below assume you have the following files:
1. `certs/ca.crt`
2. `certs/node.crt`
3. `certs/node.key`

If you don't have these certs, then you can use the provided script to generate self-signed certificates in the location listed above:

```
./generate-certs.sh
```

These certs will work for the internal broker hostnames, as well as the external hostname `*.local`. If you chose a different external domain, then modify the above script to make use of that domain.

## Create the secret

Create a secret resource definition using your certificates:

```
kubectl create secret generic tls-external --from-file=ca.crt=certs/ca.crt --from-file=tls.crt=certs/node.crt --from-file=tls.key=certs/node.key --dry-run=client -o yaml > tls-external.yaml
```

Now create the `redpanda` namespace and then create the secret using the definition:

```
kubectl create ns redpanda
kubectl apply -f tls-external.yaml -n redpanda
```

## Deploy Redpanda

A [values.yaml](./values.yaml) is provided that does the following:

- references the secret created above (`tls-external`) to create a TLS certificate named `external`
- disables global TLS, and then enables TLS on the Kafka and admin listeners and ensures they use the `external` certificate
- disables the `default` certificate so that `cert-manager` isn't required
- sets the external domain to `local` (change this whatever domain that is resolvable within your environment)

Use helm to deploy Redpanda using the provided `values.yaml`:

```
helm upgrade --install redpanda redpanda/redpanda -n redpanda --wait --debug --timeout 1h -f values.yaml
```

## Deploy traefik

Traefik references both the internal and external addresses for each broker (ie. `redpanda-0.redpanda.redpanda.svc.cluster.local`, `redpanda-0.local`, etc.). If you modified the external domain above, then make sure you edit [traefik.yaml](./traefik.yaml) to change any reference to `.local` to your chosen domain.

Deploy traefik with the following command:

```
kubectl apply -f traefik.yaml
```

Eventually the LoadBalancer service `traefik-lb` will obtain an external IP. You can verify this with the following command:

```
kubectl get service/traefik-lb -n traefik
```

This external IP is what will need to resolve the the external addresses of each broker to. If you are running locally, then you can edit your `/etc/hosts` with the following:

```
127.0.2.1 redpanda-0.local
127.0.2.1 redpanda-1.local
127.0.2.1 redpanda-2.local
```

Replace `127.0.2.1` with the external IP address of the `traefik-lb` service, and replace `local` with your chosen external domain.

## Verification

Your cluster should now be available, and can be accessed with `rpk`. Create a profile (we'll call it `aks-redpanda`):

```
rpk profile create aks-redpanda -s brokers=redpanda-0.local:9094 -s tls.ca="$(realpath certs/ca.crt)" -s admin.hosts=redpanda-0.local -s admin.tls.ca="$(realpath certs/ca.crt)"
```

Replace `local` in the above command with your external domain.

Now run the following command and add in the other brokers to both the `brokers` and `addresses` sections:

```
rpk profile edit
```

Now run the following commands to connect to each of your brokers through the single traefik LoadBalancer:

```
rpk cluster info
rpk cluster health
rpk topic create log -r 3
```


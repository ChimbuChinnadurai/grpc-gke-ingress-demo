# Streaming gRPC on GKE with GKE Ingress

Steps to deploy gRPC server streaming service running on GKE behind Google's Global External Application Load Balancer using GKE Ingress, Google-managed certificate and Envoy proxy for TLS termination at the backends.

This project is based on the gateway api example available at https://github.com/stepanstipl/gke-grpc-gateway-api/tree/master

## Prerequisites

These steps expect a GKE cluster and internet access (to download the prebuilt container image). Also, the usual `gcloud` CLI configured for your project and `kubectl` with credentials to your cluster. You'll also need a working DNS subdomain to point to the load balancer IP.

## Setup Steps

- Create a static IP for the external LB
```shell
$ gcloud compute addresses create grpc-gke-ingress-demo-vip \
--ip-version=IPV4 \
--global
```

- Point the public DNS to the previously created global IP (I'll be
using `gke-grpc-ingress.chimbuc.dns.doit-playground.com`)

- Generate a self-signed certificate for the backend and I am using mkcert [^1] to generate the certificate. 
- TLS is required both between client and GFE, as well as GFE and backend [^2]. The Client -> GFE tls certificate is managed by Google and GFE -> backend will use the self-signed certificate.

```shell
mkcert internal
```

**Important:** The certificate has to use one of the supported signatures compatible with BoringSSL, see [^3][^4] for more details. 

- Create K8S Secret with the self-signed cert
```shell
$ kubectl create secret tls grpc-gke-ingress-demo-tls \
--cert=internal.pem \
--key=internal-key.pem
```

- Create K8S Configmap with Envoy config file
```shell
$ kubectl create configmap envoy-config --from-file=envoy.yaml
```

- Deploy the demo app
```shell
$ kubectl apply -f manifests/grpc-demo.yaml
```

We're using the *Cloud Run gRPC Server Streaming sample application*[^5] which listens on port 8080.Build a new image [^6] and push it to your container registry if required.
The Envoy then listens on port 8443 with the self-signed certificate.

- Deploy K8s svc
```shell
$ kubectl apply -f manifests/service.yaml
```

- Deploy managedcertificate for LB (Don't forget to change the FQDN in the manifest). Also, ensure the DNS record is updated with the IP reserved for the external LB.
```shell
$ kubectl apply -f manifests/managedcertificate.yaml
```

- Deploy backendconfig for LB healthchecks
```shell
$ kubectl apply -f manifests/backendconfig.yaml
```

- Deploy ingress
```shell
$ kubectl apply -f manifests/ingress.yaml
```

- Test the grpc streaming service after the LB is successfully configured.

Clone the repository [^5] and build the client:
```shell
$ git clone https://github.com/GoogleCloudPlatform/golang-samples
$ cd golang-samples/run/grpc-server-streaming
$ go build -o cli ./client
```

And run the client:
```shell
$ ./cli -server grpc-gke-ingress-demo.chimbuc.dns.doit-playground.com:443
rpc established to timeserver, starting to stream
received message: current_timestamp: 2023-10-24T19:15:32Z
received message: current_timestamp: 2023-10-24T19:15:33Z
received message: current_timestamp: 2023-10-24T19:15:34Z
...
```


[^1]: https://github.com/FiloSottile/mkcert
[^2]: https://cloud.google.com/load-balancing/docs/https#using_grpc_with_your_applications
[^3]: https://github.com/grpc/grpc/issues/6722
[^4]: https://groups.google.com/a/chromium.org/forum/#!msg/blink-dev/kWwLfeIQIBM/9chGZ40TCQAJ
[^5]: https://github.com/GoogleCloudPlatform/golang-samples/tree/main/run/grpc-server-streaming
[^6]: https://github.com/GoogleCloudPlatform/golang-samples/blob/main/run/grpc-server-streaming/Dockerfile
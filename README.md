# Teleport on Kubernetes

ref: https://goteleport.com/docs/kubernetes-access/getting-started/cluster/

Teleport has a great how to get started installing Teleport in Kubernetes cluster with acme, but I didn't see an example of how to get started if you already have an ingress already setup in your cluster, which will do the certificates for you...

So, in this example, I'll be installing Teleport in a Microk8s cluster that already has ingress setup.

# Step 1/4. Install Teleport

## Install Teleport with kubectl

I like to see/know what's getting deployed, so I use `helm template` with `kubectl` instead

```bash
$ helm repo add teleport https://charts.releases.teleport.dev
$ helm repo update teleport

# Generate the teleport.yaml
$ helm template teleport-cluster teleport/teleport-cluster --create-namespace --namespace=teleport-cluster --set clusterName=teleport.example.com --set service.type=ClusterIP --set chartMode=standalone > teleport.yaml

# Review and/or tweak teleport.yaml

# Create teleport-ingress.yaml
$ cat <<EOF >./teleport-ingress.yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  annotations:
    cert-manager.io/cluster-issuer: letsencrypt-prod
    nginx.ingress.kubernetes.io/backend-protocol: "HTTPS"
  name: teleport-cluster
  namespace: teleport-cluster
spec:
  ingressClassName: public
  tls:
    - hosts:
      - teleport.example.com
      secretName: teleport-tls
  rules:
  - host: teleport.example.com
    http:
      paths:
      - backend:
          service:
            name: teleport-cluster
            port:
              number: 443
        path: /
        pathType: Prefix
EOF

# Install teleport with ingress
$ kubectl apply -f teleport.yaml -f teleport-ingress.yaml --namespace=teleport-cluster

```

## Install Teleport with kustomize

Edit `configmap.yaml` and `kustomization.yaml` files

```bash
# Create a kustomization.yaml file
cat <<EOF >./kustomization.yaml
namespace: teleport-cluster

bases:
  - https://github.com/jefferyb/k8s-teleport.git?ref=v8

commonLabels:
  app: teleport-cluster
  env: production

patchesJson6902:
  - path: ingress-patch.yaml
    target:
      group: networking.k8s.io
      version: v1
      kind: Ingress
      name: teleport-cluster
EOF

# Create a ingress-patch.yaml file
cat <<EOF >./ingress-patch.yaml
- op: replace
  path: /spec/tls/0/hosts/0
  value: gitea.192.168.1.37.nip.io
- op: replace
  path: /spec/rules/0/host
  value: gitea.192.168.1.37.nip.io
EOF
```

To view the Deployment, run:

```bash
kubectl kustomize ./
```

To apply/deploy, run:

```bash
kubectl kustomize ./ | kubectl apply -f -
#  OR
kubectl apply --kustomize ./
```

# Step 2/4. Configure ingress to expose some extra ports

We need to expose some ports. On microk8s ( for ref: https://microk8s.io/docs/addon-ingress ):

  * Edit the `nginx-ingress-tcp-microk8s-conf` configmap in the `ingress` namespace
  * Add ports:
    ```yaml
    data:
      "3023": teleport-cluster/teleport-cluster:3023
      "3026": teleport-cluster/teleport-cluster:3026
      "3024": teleport-cluster/teleport-cluster:3024
      "3036": teleport-cluster/teleport-cluster:3036
    ```
  * and then expose the port in the Ingress controller
    ```yaml
    apiVersion: apps/v1
    kind: DaemonSet
    metadata:
      name: nginx-ingress-microk8s-controller
      namespace: ingress
    spec:
      template:
        spec:
          containers:
          - name: nginx-ingress-microk8s
            ports:
            - containerPort: 80
              hostPort: 80
              name: http
              protocol: TCP
            - containerPort: 443
              hostPort: 443
              name: https
              protocol: TCP
            - containerPort: 10254
              hostPort: 10254
              name: health
              protocol: TCP
            - containerPort: 3023
              hostPort: 3023
              name: proxy-tcp-3023
              protocol: TCP
            - containerPort: 3026
              hostPort: 3026
              name: proxy-tcp-3026
              protocol: TCP
            - containerPort: 3024
              hostPort: 3024
              name: proxy-tcp-3024
              protocol: TCP
            - containerPort: 3036
              hostPort: 3036
              name: proxy-tcp-3036
              protocol: TCP
    ```

and if you're running microk8s in lxc, make sure to add your proxies, otherwise, skip this part

```bash
### For running Teleport
# Port 3023/teleport-sshproxy
lxc config device add microk8s teleport-sshproxy-3023 proxy listen=tcp:0.0.0.0:3023 connect=tcp:127.0.0.1:3023
# Port 3026/teleport-k8s-proxy
lxc config device add microk8s teleport-k8s-proxy-3026 proxy listen=tcp:0.0.0.0:3026 connect=tcp:127.0.0.1:3026
# Port 3024/teleport-sshtun
lxc config device add microk8s teleport-sshtun-3024 proxy listen=tcp:0.0.0.0:3024 connect=tcp:127.0.0.1:3024
# Port 3036/teleport-mysql
lxc config device add microk8s teleport-mysql-3036 proxy listen=tcp:0.0.0.0:3036 connect=tcp:127.0.0.1:3036
```

If you want to remove the proxies:

```bash
# To remove a proxy device, run
lxc config device remove microk8s teleport-sshproxy-3023
lxc config device remove microk8s teleport-k8s-proxy-3026
lxc config device remove microk8s teleport-sshtun-3024
lxc config device remove microk8s teleport-mysql-3036
```

# Step 3/4. Create a local user
Local users are a reliable fallback for cases when the SSO provider is down. Let's create a local user alice who has access to Kubernetes group system:masters ( admin access ).

```bash
# To create a local user, we are going to run Teleport's admin tool tctl from the pod.

POD=$(kubectl get pod -l app=teleport-cluster -o jsonpath='{.items[0].metadata.name}')

# Generate an invite link for the user.
kubectl exec -ti ${POD?} -- tctl  users add alice --roles=access
# User "alice" has been created but requires a password. Share this URL with the user to
# complete user setup, link is valid for 1h:
# https://tele.example.com:443/web/invite/random-token-id-goes-here
# NOTE: Make sure tele.example.com:443 points at a Teleport proxy which users can access.

# Edit user
kubectl exec -ti ${POD?} -- tctl get user > user.yaml

# Edit the user.yaml file and add system:masters ( admin permissions ) to kubernetes_groups
#    traits:
#     kubernetes_groups:
#     - "system:masters"

# Update user
kubectl exec -i ${POD?} -- tctl create -f < user.yaml
```

# Step 4/4. SSO for Kubernetes

https://goteleport.com/docs/kubernetes-access/getting-started/cluster/#step-33-sso-for-kubernetes


# Connecting Web Applications

ref: https://goteleport.com/docs/application-access/guides/connecting-apps/#start-application-service-with-a-config-file

What I have found working best, is to first configure the ingress, and then the app side

Example:

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  annotations:
    cert-manager.io/cluster-issuer: letsencrypt-prod
    nginx.ingress.kubernetes.io/backend-protocol: HTTPS
  name: teleport-cluster
  namespace: teleport-cluster
spec:
  ingressClassName: public
  rules:
  - host: teleport.example.com
    http:
      paths:
      - backend:
          service:
            name: teleport-cluster
            port:
              number: 443
        path: /
        pathType: Prefix
  - host: grafana.teleport.example.com
    http:
      paths:
      - backend:
          service:
            name: teleport-cluster
            port:
              number: 443
        path: /
        pathType: Prefix
  - host: git.example.com
    http:
      paths:
      - backend:
          service:
            name: teleport-cluster
            port:
              number: 443
        path: /
        pathType: Prefix
  tls:
  - hosts:
    - teleport.example.com
    - git.example.com
    - grafana.teleport.example.com
    secretName: teleport-tls
```

next you configure the application service

```yaml
$ cat /etc/teleport/teleport.yaml 
teleport:
  # Data directory for the Application Proxy service. If running on the same
  # node as Auth/Proxy service, make sure to use different data directories.
  data_dir: /var/lib/teleport-app
  # Instructs the service to load the join token from the specified file
  # during initial registration with the cluster.
  auth_token: /tmp/token
  # Proxy address to connect to. Note that it has to be the proxy address
  # because the app service always connects to the cluster over a reverse
  # tunnel.
  auth_servers:
    - teleport.example.com:3080
app_service:
    enabled: yes
    # Teleport provides a small debug app that can be used to make sure application
    # access is working correctly. It'll output JWTs so it can be useful
    # when extending your application.
    # debug_app: true
    # This section contains definitions of all applications proxied by this
    # service. It can contain multiple items.
    apps:
      # Name of the application. Used for identification purposes.
    - name: "git"
      # URI and port the application is available at.
      uri: "http://localhost:3000"
      # Optional application public address to override.
      public_addr: "git.example.com"
      # Optional static labels to assign to the app. Used in RBAC.
      labels:
        env: "prod"
      # Optional dynamic labels to assign to the app. Used in RBAC.
      commands:
      - name: "os"
        command: ["/usr/bin/uname"]
        period: "21s"
    
    ### kibana
    - name: "grafana"
      # URI and port the application is available at.
      uri: "http://localhost:3000"
      # Optional application public address to override.
      public_addr: "grafana.teleport.example.com"
      # Optional static labels to assign to the app. Used in RBAC.
      labels:
        env: "prod"
      # Optional dynamic labels to assign to the app. Used in RBAC.
      commands:
      - name: "os"
        command: ["/usr/bin/uname"]
        period: "21s"
auth_service:
  enabled: "no"
ssh_service:
  enabled: "no"
proxy_service:
  enabled: "no"
```

make sure that the service is pointing to your config, with `--config=/etc/teleport/teleport.yaml`

```bash
$ cat /usr/lib/systemd/system/teleport.service
[Unit]
Description=Teleport App Service
After=network.target

[Service]
Type=simple
Restart=on-failure
EnvironmentFile=-/etc/default/teleport
ExecStart=/usr/local/bin/teleport start --pid-file=/run/teleport.pid --config=/etc/teleport/teleport.yaml
ExecReload=/bin/kill -HUP $MAINPID
PIDFile=/run/teleport.pid
LimitNOFILE=8192

[Install]
WantedBy=multi-user.target
```

and then start/restart teleport

```bash
systemctl restart teleport.service
systemctl status teleport.service
# And you can enable it to start on reboot
systemctl enable teleport.service
```

And with that, you should be able to access your apps at:

  - https://teleport.example.com
  - https://git.example.com
  - https://grafana.teleport.example.com


# Use Cloudflare Tunnels as an Ingress Controller

## Introduction

### Why?

Cloudlflare tunnels are a secure way to outsource your edge routing without exposing ports in your cluster. This is achieved by running a lightweight daemon called cloudflared as a deployment within the cluster. This component establishes an encrypted tunnel out of the cluster into the Cloudflare network.

When suitably authenticated clients attempt to connect to the tunnel they are routed to the relevant configured services running in the cluster.

### Advantages

- No need to use the Kommander Traefik ingress controller or deploy another
- No need to provide a load balancer; a simple clusterIP will suffice
- Does not expose an ingress to the outside world
- Manage all configuration through the Cloudflare "Zero Trust" dashboard which is much simpler and faster than Traefik
- Enrol users, devices and groups to fine tune security and access
- Always have a valid SSL certificate for you service without any config
- Always presents end users with a "locked padlock" in the web-browser
- It's free for up to 50 users
- Full logging for all access and network traffic

### Deployment

Deployment is simple

  1. Ensure you have a Cloudflare account (free) and host the domain you want to use.

  2. [Download and Run](https://developers.cloudflare.com/cloudflare-one/connections/connect-apps/install-and-setup/installation/) the cloudflared package locally.

  3. Log with the cli

        ```bash
        cloudflared tunnel login
        ```

  4. Create a new tunnel. This will generate your tunnel credentials in ~/.cloudflared/<tunnel_id>.json

        ```bash
        cloudflared tunnel create example-tunnel
        ```

  5. Create a secret in the target cluster to store your creds. Make a note of the tunnel ID. It should look similar to this - "ef824aef-7557-4b41-a398-4684585177ad"

        ```bash
        kubectl create secret generic tunnel-credentials \
        --from-file=credentials.json=~/.cloudflared/<tunnel_id>.json
        ```

  6. Go to the Cloudflare DNS administration page for your domain and create a CNAME record as follows. This will map your chosen domain / sub-domain to the tunnel so your end users can access your kubernetes service.

        ```bash
        Type:       CNAME
        Domain:     my-domain.com
        Target:     <tunnel_id>.cfargotunnel.com
        ```

  7. Modify your configuration. Modify ./install/config.yaml for your particular service. You will need to:

     - Ensure that the "tunnel" field is the same name as tunnel you just created.
     - The hostname is the one you set up to resolve to the service, ie my-service.my-domain.com
     - The service field is the name of the service (use "kubectl get svc -A") to get its name and port number.

  8. Apply the cloudflared deployment and config manifests in the "Install" directory.

  9. Wait for the cloudflared pods to initialise, the default for this deployment is 2 replicas.

  10. Noting that it can take a while to propagate your new DNS entry (stage 6), visit the hostname. It should resolve to the kubernetes service you defined in stage 7.

  11. Configure all your ingress routes by simply adding them to the config map we created in step 7.

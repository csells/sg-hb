# Managed Instance Private Code Hosts support

WARNING: This is still work in progress!

Every [v2.0 Cloud Instance](../v2.0/index.md) is deployed in Google Cloud Platform.

**Private Code Host** is a code host deployed in a private network (for example AWS EC2 instance within VPC). To connect to this code host a user has to have access to the private network usually via VPC Peering, VPN, or tunneling.

## Managed Instance NAT IP allowlist solution (recommended)

This solution is recommended by Cloud Team, but only possible if customers' code hosts can be accessible publicly and customer is able to allow incoming traffic from Sourcegraph-owned IP addresses.
Outgoing traffic of Cloud instances goes through Cloud NAT with stable IPs. All IPs are reserved exclusively on a per customer basis.
[More informations about IP allowlist](../../index.md#faq-what-is-the-cloud-instance-ip)

## AWS GCP site-to-site VPN solution (AWS code hosts only)

This option is for customers who want to connect to a private code host that is hosted on AWS infrastructure.

### Architecture

[AWS VPN architecture](https://app.excalidraw.com/s/4Dr1S6qmmY7/4L5TAaxiYAy)

### Creation process

AWS GCP VPN extension connects an existing Cloud Instance with customer dedicated AWS account (maintained by Cloud Team).

#### 1. Modify `config.yaml` with additional section:

```yaml
spec:
  infrastructure:
    privateCodeHost:
      aws:
        accountID: <AWS_ACCOUNT_ID>
        highAvailable: true
        region: <AWS_REGION>
      enabled: true
      type: awsvpn
```

#### 2. Generate additional terraform stacks

```
mi2 generate cdktf
```

this will generate `awsvpc` and `awsvpn` stacks.

Creation process detail walkthrough:

1. AWS VPC is created in customer dedicated Sourcegraph AWS Account.
1. Cloud Router with GCP-side BGP ASN number is created in Managed Instance VPC.
1. AWS VPN Gateway with BGP AWS-side ASN number is created. It is attached to VPC and uses VPC route tables to propagate BPG sessions.
1. High Available GCP VPN Gateway is created in Managed Instance GCP VPC with 2 interfaces.
1. AWS Customer Gateway is created (uses GCP-side BGP ASN). For high available options 2 gateways are created.
1. AWS VPN connection is created with 2 tunnel options (every tunnel uses different 32B PreSharedKey). For high available options 2 VPC connections are created.
1. GCP External VPN Gateway is created with interfaces correspoding for each AWS VPN connection tunnel. For high available options External VPN Gateway has 4 interfaces, for non high-available 2.
1. For each External VPN Gateway interface GCP VPN Tunnel is created. It uses dedicated PreSharedKey (same in on AWS VPN Connection(s)).
1. For each GCP VPN Tunnel interface is added to Cloud Router. It uses IP range correspoding to AWS VPN Connection tunnel internal address.
1. For each AWS VPN Connection tunnel internal address BGP peer is added to Cloud Router.

For more details, go to [Google documentation](https://cloud.google.com/network-connectivity/docs/vpn/tutorials/create-ha-vpn-connections-google-cloud-aws)

### VPN Verification

For each customer using private code host, additional section to our generated operation dashboard is added.
Upon enabling the private code host support, follow the process [to update the dashboard](https://github.com/sourcegraph/cloud/blob/main/prod.dashboard.md#update-all-generated-dashboards)

## [Optional] Private code host domain

Every Cloud customer using private DNS domain for code hosts requires additional dns-proxy sidecar.
DNS-proxy is rewriting from private customer domain to public, resolvable A record
i.e. `private.CUSTOMER.com` -> `public.CUSTOMER.com` where:

- `private.CUSTOMER.com` is customer code host address, only resolvable inside customer network
- `public.CUSTOMER.com` - is resolvable to customer code host private IP
  Dns-proxy sidecar modifies pod `/etc/resolv.conf` file.

1. Configure config.yaml

```yaml
spec:
  infrastructure:
    privateDNS:
      routes:
        - source: private.CUSTOMER.com
          destination: public.CUSTOMER.com
          # resolverIP: IP_OF_CUSTOM_RESOLVER - optional
    gke:
      blue:
        additionalOauthScopes: # required for each node pool to pull dns-proxy private image from artifact registry
          - https://www.googleapis.com/auth/devstorage.read_only
```

1. Generate and apply cdktf

```sh
mi2 generate cdktf
# modify GKE node pool and add artifact registry repository with permissions
mi2 instance tfc deploy -auto-approve
```

1. Generate dns-proxy sidecars

```sh
# generate sidecars and copy dns-proxy from control plane project to customer project artifact registry repository
mi2 generate kustomize
cd kubernetes
kustomize build --load-restrictor LoadRestrictionsNone --enable-helm . | kubectl apply -f -
```

### AWS Private Link playbook

**TBD**
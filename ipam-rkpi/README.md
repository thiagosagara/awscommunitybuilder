# VPC IPAM BGP route protection for BYOIP

Infrastructure and step-by-step commands to protect a BYOIP range on AWS with
**VPC IPAM delegated RPKI**: AWS creates and renews the ROAs for your prefixes,
monitors your routes for overlapping announcements from other ASNs, and alarms
before a ROA expires.

Companion repository for the article *How to Protect Your IP Range on AWS from
BGP Hijacking* ([link]).

## What this protects you from (and what it doesn't)

In 2008, Pakistan Telecom announced a /24 inside YouTube's /22. Longest prefix
match did the rest and YouTube went dark for a large part of the world for
about 80 minutes.

IPAM keeps a valid, Strict ROA for your prefix and tells you when someone else
announces something inside it. Networks that run ROV (Route Origin Validation)
use that ROA to drop the fake announcement. **IPAM does not block anything
itself**: protection only happens in networks that validate, and it does not
respond to an incident for you.

## Repository layout

```
.
├── README.md
└── template.yaml   # IPAM Advanced, BYOIP pool, RIR association, ROA alarm
```

## What runs where

CloudFormation does not cover the whole flow. The RIR step is manual by
nature, and some IPAM operations have no CloudFormation resource yet.

| Step | What happens | How |
| --- | --- | --- |
| 1 | IPAM Advanced, BYOIP pool, RIR association | `template.yaml` |
| 2 | Get the child request XML | CLI or console |
| 3 | Delegate RPKI at your RIR portal | Manual, RIR portal |
| 4 | Enable the association with the parent response | CLI or console |
| 5 | Provision the BYOIP CIDR (ROA is created here) | CLI |
| 6 | Verify routes, ROAs and findings | CLI or console |
| 7 | ROA expiration alarm | `template.yaml` (stack update) |
| 8 | ROAs for prefixes kept on-premises (RPR) | CLI |

## Prerequisites

- An IP block of your own, registered at a RIR, and access to that RIR's portal.
- AWS CLI v2 with permissions for EC2 IPAM, CloudFormation, CloudWatch and SNS.
- Budget for the IPAM Advanced Tier (see Cost).

Delegated RPKI is supported for ARIN, RIPE NCC, APNIC and LACNIC. AFRINIC gets
route discovery and findings only. GovCloud (US) and the China Regions are not
supported.

## Step by step

All commands assume `us-east-1`. Replace the example IDs with the stack
outputs.

### 1. Deploy the stack

```bash
aws cloudformation deploy \
  --template-file template.yaml \
  --stack-name ipam-bgp-protection \
  --parameter-overrides Rir=LACNIC OrgHandle=YOUR-ORG-HANDLE \
  --region us-east-1

aws cloudformation describe-stacks --stack-name ipam-bgp-protection \
  --query "Stacks[0].Outputs" --region us-east-1
```

Wait until `AssociationState` reads `pending-enable`.

### 2. Get the child request XML

```bash
aws ec2 describe-ipam-internet-registry-associations --region us-east-1 \
  --ipam-internet-registry-association-ids <AssociationId> \
  --query "IpamInternetRegistryAssociations[0].ChildRequestXml" --output text
```

### 3. Delegate at the RIR

Paste the child request XML in your RIR portal (on Registro.br: Numeração,
Configurar RPKI) and download the **parent response XML**.

If you already have ROAs in hosted RPKI, check what happens to them when you
switch to delegated. Some RIRs revoke the hosted CA on the switch.

### 4. Enable the association

The console is easier here: **Enable**, paste the parent response XML, **Parse
XML**. With the CLI you pass the five fields from the XML yourself:

```bash
aws ec2 enable-ipam-internet-registry-association --region us-east-1 \
  --ipam-internet-registry-association-id <AssociationId> \
  --rpki-version "..." \
  --service-uri "..." \
  --child-handle "..." \
  --parent-handle "..." \
  --parent-bpki-ta "..."
```

Wait for `enable-complete`.

### 5. Provision the CIDR

With delegation active, ownership is verified through the association, so
there is no signed message and no WHOIS or DNS TXT check.

```bash
aws ec2 provision-ipam-pool-cidr --region us-east-1 \
  --ipam-pool-id <PoolId> \
  --cidr 203.0.113.0/24 \
  --cidr-authorization-context Message="CoveredByInternetRegistryAssociation",Signature=""
```

IPAM creates a **Strict** ROA (max-length equal to the prefix length) for the
pool's locale and ASN.

### 6. Verify

```bash
aws ec2 get-ipam-route-origin-authorizations --region us-east-1 \
  --ipam-internet-registry-association-id <AssociationId>

aws ec2 get-ipam-route-protection-findings --region us-east-1 \
  --ipam-id <IpamId> \
  --filters "Name=rpki-status,Values=invalid,unknown,valid"
```

In the console: **IPAM > Monitoring > Route monitoring**. Look at
`RouteOverlaps`: a more specific prefix from another ASN inside your range is
the 2008 pattern.

### 7. Turn on the ROA expiration alarm

An expired ROA moves your route from Valid to Unknown without any error, and
ROV does not drop Unknown. Once you know the ROA's ASN and max-length:

```bash
aws cloudformation deploy \
  --template-file template.yaml \
  --stack-name ipam-bgp-protection \
  --parameter-overrides CreateRoaAlarm=true RoaCidr=203.0.113.0/24 \
      RoaAsn=64512 RoaMaxLength=24 AlertEmail=you@example.com \
  --region us-east-1
```

Confirm the SNS subscription email. There is no native notification for route
overlaps; poll `get-ipam-route-protection-findings` on a schedule if you need
one.

### 8. ROAs for prefixes that stay on-premises

```bash
aws ec2 create-ipam-routing-policy-registration --region us-east-1 \
  --ipam-id <IpamId> \
  --ipam-internet-registry-association-id <AssociationId> \
  --cidr 198.51.100.0/24 --asns 64500
```

The default is exact match. If you register a /22 and announce /24s from it,
those /24s become Invalid. Register what you actually announce, or use
`--permit-more-specific-announcements` with the max-length you need.

## Cleanup

Order matters: the association cannot be deleted while it has routing policy
registrations, and the pool cannot be deleted while it has CIDRs.

```bash
export AWS_DEFAULT_REGION=us-east-1
ASSOC_ID=<AssociationId>
POOL_ID=<PoolId>

# 1. Routing policy registrations, one per prefix
aws ec2 delete-ipam-routing-policy-registration \
  --ipam-internet-registry-association-id $ASSOC_ID --cidr 198.51.100.0/24

# 2. Provisioned CIDR
aws ec2 deprovision-ipam-pool-cidr --ipam-pool-id $POOL_ID --cidr 203.0.113.0/24

# 3. RIR association (kept by the stack on purpose, see below)
aws ec2 delete-ipam-internet-registry-association \
  --ipam-internet-registry-association-id $ASSOC_ID

# 4. Stack: IPAM, pool, alarm and SNS topic
aws cloudformation delete-stack --stack-name ipam-bgp-protection
```

The association has `DeletionPolicy: Retain` because deleting it removes every
ROA AWS manages under it. A careless `delete-stack` would take your ROAs with
it, so you remove it deliberately in step 3.

Finally, remove the delegation at the RIR portal if you want to undo it
completely. On Registro.br, "Desabilitar RPKI" disables RPKI for the whole
holder, not only the AWS delegation, and can affect your BGP announcements.

## Cost

The Advanced Tier charges US$ 0.00027 per active IP per hour, for **every
active IP the IPAM sees**, not per BYOIP prefix. A fully used /24 (256 IPs) is
about US$ 49.77 per 30 days; a scope with 10,000 active IPs is about
US$ 1,944. Check the current price on the
[VPC pricing page](https://aws.amazon.com/vpc/pricing/).

## Known issues

- **NIC.BR in CloudFormation.** The CLI accepts `--rir NICBR`, but the
  CloudFormation schema only accepts RIPE, APNIC, ARIN and LACNIC. If your
  block is at Registro.br, create the association with the CLI and remove
  `RirAssociation` from the template, or validate that `LACNIC` works for your
  block before relying on it.
- **No pre-created ROAs for LACNIC and NIC.BR.** The ROA is created when you
  provision the CIDR, not when the association is enabled.
- **Provisioning is CLI only.** `AWS::EC2::IPAMPoolCidr` has no property for
  the CIDR authorization context used by BYOIP.

## References

- [Monitor BGP route protection](https://docs.aws.amazon.com/vpc/latest/ipam/monitor-bgp-route-security.html)
- [Tutorial: Set up delegated RPKI for BYOIP prefixes](https://docs.aws.amazon.com/vpc/latest/ipam/tutorials-byoip-bgp-security.html)
- [AWS::EC2::IpamInternetRegistryAssociation](https://docs.aws.amazon.com/AWSCloudFormation/latest/TemplateReference/aws-resource-ec2-ipaminternetregistryassociation.html)
- [YouTube Hijacking: A RIPE NCC RIS case study](https://www.ripe.net/about-us/news/youtube-hijacking-a-ripe-ncc-ris-case-study/)

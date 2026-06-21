# n8n Automation Scripts

One-click CloudFormation template to self-host [n8n](https://n8n.io) on AWS with sensible defaults.

## What you get

- Single-EC2 deployment with Docker
- Auto-generated encryption key stored in Secrets Manager
- Persistent EBS volume that survives stack deletes (auto-snapshotted)
- SSM Session Manager for shell access — no SSH keys to manage
- n8n built-in user management (invite-only, no public sign-up)
- Public REST API disabled, telemetry off, community packages disabled

**Cost:** ~$17/mo on `t3.small` + ~$2/mo storage + $0.40/mo for the secret. Free tier eligible for first year on `t3.micro`.

## Deploy

### Option 1 — AWS Console (easiest)

1. Open the [CloudFormation Console](https://console.aws.amazon.com/cloudformation/home).
2. **Create stack → With new resources (standard)**.
3. Upload `n8n-cloudformation.yaml`.
4. Stack name: `n8n` (or anything). Defaults are fine.
5. Check the IAM acknowledgement box at the bottom.
6. **Create stack** — wait ~3 minutes.
7. Open the **Outputs** tab → click **`ClaimOwnerAccountURL`** to claim the owner account.

### Option 2 — AWS CLI

```bash
aws cloudformation create-stack \
  --stack-name n8n \
  --template-body file://n8n-cloudformation.yaml \
  --capabilities CAPABILITY_IAM
```

## After deploy

The stack outputs include:

| Output | What it is |
|--------|-----------|
| `N8nURL` | `http://<EIP>:5678` |
| `ClaimOwnerAccountURL` | Setup page — claim it before anyone else |
| `EncryptionKeySecretArn` | ARN of the encryption secret in Secrets Manager |
| `GetEncryptionKeyCommand` | Ready-to-paste CLI to retrieve the key |
| `ConnectViaSessionManager` | Shell access without SSH keys |
| `ElasticIP` | Static public IP |

## Architecture

```
Internet
   ↓
Internet Gateway
   ↓
Public Subnet (10.0.1.0/24)
   ↓
Security Group (port 5678 only)
   ↓
EC2 instance (Amazon Linux 2023, Docker)
   ↓
n8n container (with /opt/n8n-data persistent volume)
```

## Tradeoffs / known limits

- **HTTP only.** No HTTPS out of the box. For OAuth (Google, etc.) you'll need to add a domain + ALB/ACM or front it with Cloudflare Tunnel. SMTP-based Gmail (port 587 + App Password) works fine on plain HTTP.
- **Single AZ, single instance.** No queue mode, no workers, no Multi-AZ HA. Good for personal/team use up to a few hundred workflows/day.
- **No RDS / Redis.** SQLite on the EBS volume. Fine for small scale, switch to Postgres if you grow.

## Customizing

The CloudFormation template uses a `.env` file (written by the UserData script) to configure n8n. To change settings on a running instance:

```bash
# Connect via SSM
aws ssm start-session --target <INSTANCE_ID>

# Edit env vars
sudo nano /opt/n8n/.env

# Restart container
cd /opt/n8n && sudo docker compose up -d
```

Common ones to add:

```bash
GENERIC_TIMEZONE=America/New_York
N8N_DEFAULT_LOCALE=en
EXECUTIONS_DATA_PRUNE=true
EXECUTIONS_DATA_MAX_AGE=336   # hours, ~14 days
```

## Things I learned the hard way

- n8n 1.0+ removed basic auth. The first visitor to `/setup` becomes the owner. Claim it within minutes of deploy.
- Setting `N8N_SECURE_COOKIE=false` is required when serving over plain HTTP, otherwise login loops forever.
- Inline env vars in `docker-compose.yml` are a YAML indentation trap — using a separate `.env` file via `env_file:` is much safer.
- AWS Marketplace has paid n8n AMIs but they bundle license fees on top of EC2. This template is the same thing, free.

## Roadmap (PRs welcome)

- [ ] HTTPS via ALB + ACM + Route53
- [ ] Cloudflare Tunnel option (free HTTPS, no domain needed)
- [ ] Queue mode with worker auto-scaling
- [ ] RDS Postgres + ElastiCache Redis variant
- [ ] Automated S3 backups of workflows

## License

MIT — do whatever you want with it.

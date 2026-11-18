# Deploying an OpenClaw Agent to AWS EC2

A reusable guide for deploying **any OpenClaw agent** from a local machine to an AWS EC2 instance.

## 1. Create an EC2 instance

In AWS Console, open **EC2 → Launch instance**.

  Setting         Recommendation (for testing purpose)

  --------------- ---------------------------------------------------------------

  Region          Closest suitable AWS region
  AMI             Ubuntu 24.04 LTS or another OpenClaw-supported Ubuntu release
  Architecture    64-bit x86
  Instance type   `t3.small` (2 GiB RAM) as a practical starting point
  Storage         15--30 GiB gp3
  Public IP       Enable auto-assignment
  Key pair        Create and download a `.pem` file

**Security group** 

- Allow SSH (TCP 22) only from your current public IP (`/32`). 
- Do not expose the OpenClaw Gateway port (`18789`) publicly. 
- Add other inbound rules only when genuinely required.

## 2. Connect to EC2

On your local machine:

```bash
chmod 400 ~/Downloads/<key-pair>.pem

ssh -i ~/Downloads/<key-pair>.pem \
  ubuntu@<EC2_PUBLIC_IP>
```

If the instance is stopped and started again, its auto-assigned public IP may change. Check the current IP in the EC2 Console.

## 3. Prepare Ubuntu

Update the operating system:

```bash
sudo apt update && sudo apt upgrade -y
```

For smaller instances, you can optionally configure 2 GiB of swap:

```bash
sudo fallocate -l 2G /swapfile
sudo chmod 600 /swapfile
sudo mkswap /swapfile
sudo swapon /swapfile

echo '/swapfile none swap sw 0 0' | sudo tee -a /etc/fstab
```

## 4. Install OpenClaw

Follow the current official installation instructions:

https://docs.openclaw.ai/

Verify the installation:

```bash
openclaw --version
```

Complete the initial setup and ensure the Gateway is running.

## 5. Transfer the agent workspace

On your local machine, package the agent workspace. Example:

```bash
tar -czf /tmp/my-agent-workspace.tar.gz \
  --exclude='.git' \
  -C "$HOME/Projects" \
  my-agent
```

Transfer it to EC2:

```bash
scp -i ~/Downloads/<key-pair>.pem \
  /tmp/my-agent-workspace.tar.gz \
  ubuntu@<EC2_PUBLIC_IP>:~/
```

On EC2, extract it:

```bash
mkdir -p "$HOME/Projects"

tar -xzf ~/my-agent-workspace.tar.gz \
  -C "$HOME/Projects"
```

**Transfer considerations** 

- Preserve workspace instructions, configuration, templates, skills, and relevant memory files. 
- Exclude unnecessary files such as `.git`, caches, and temporary files. 
- Do not blindly copy the entire `~/.openclaw` directory. It may contain credentials, private configuration, and databases. 
- Transfer agent state or conversation history separately only when needed, after reviewing compatibility and security implications.

## 6. Register the agent

Register the agent with OpenClaw:

```bash
openclaw agents add <agent-id> \
  --workspace "$HOME/Projects/my-agent" \
  --non-interactive
```

Verify the registration:

```bash
openclaw agents list
```

Confirm that the agent points to the intended workspace.

## 7. Configure the model and authentication

Configure the desired model and authenticate the relevant provider.

For example, GitHub Copilot:

```bash
openclaw models auth login-github-copilot --agent <agent-id>
```

Other providers may require API keys or different authentication methods.

If an authentication change is blocked because the Gateway owns the OpenClaw state, stop the Gateway first:

```bash
openclaw gateway stop
```

Perform the authentication step, then restart it:

```bash
openclaw gateway start
```

Verify that the agent has access to its intended model and credentials. Also verify the agent's model configuration; registering an agent does not necessarily configure its model automatically.

## 8. Start and test the agent

Start its TUI using its session key:

```bash
openclaw tui agent:<agent-id>:main
```

Test with a read-only request:

```text
Introduce yourself and inspect your workspace.
Report your role, project structure, and missing configuration.
Do not modify or install anything.
```

Verify: 

- Correct agent identity. 
- Correct workspace. 
- Expected model. 
- Successful response. 
- No unexpected file modifications.

Then test the agent's actual intended workflow.

## 9. Access the Gateway dashboard remotely (optional)

Keep the Gateway bound to localhost. From your local machine, establish an SSH tunnel:

```bash
ssh -i ~/Downloads/<key-pair>.pem \
  -N -L 18789:127.0.0.1:18789 \
  ubuntu@<EC2_PUBLIC_IP>
```

Open this address in your browser:

http://localhost:18789/

Do not expose the Gateway port publicly through the EC2 security group.

## 10. Manage the EC2 instance

For occasional use, stop the instance when finished.

  -----------------------------------------------------------------------

  Action                              What happens

  ----------------------------------- -----------------------------------

  Stop EC2                            Compute charges stop

  Start EC2                           The Gateway service can start
                                      automatically if configured as a
                                      service

  Reboot EC2                          The instance restarts

  Stop and start                      An auto-assigned public IP may
                                      change

  Keep stopped                        EBS storage charges continue

The `.pem` key remains valid when the instance is stopped or its instance type changes.

## Deployment checklist

- [ ] Create EC2 instance.
- [ ] Configure security group.
- [ ] Connect through SSH.
- [ ] Prepare Ubuntu and optional swap.
- [ ] Install OpenClaw.
- [ ] Transfer the agent workspace.
- [ ] Register the agent.
- [ ] Configure the model and authentication.
- [ ] Start and test the agent.
- [ ] Verify workspace access and its intended workflow.
- [ ] Verify persistence after restart.
- [ ] Stop EC2 when not in use.

## Deployment architecture

```text
Local Machine
     |
     | SSH / SCP
     v
AWS EC2 (Ubuntu)
     |
     +-- OpenClaw Gateway
     |
     +-- Agent A
     |     +-- Workspace A
     |     +-- Model / Authentication
     |
     +-- Agent B
     |     +-- Workspace B
     |     +-- Model / Authentication
     |
     +-- Agent C
           +-- Workspace C
           +-- Model / Authentication
```

**Key principle:** Install OpenClaw once on EC2. You can register multiple agents, each with its own workspace, identity, configuration, and model settings. You do not need a separate EC2 instance for every agent.

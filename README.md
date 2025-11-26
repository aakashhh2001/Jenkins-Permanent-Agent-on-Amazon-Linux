# Jenkins Permanent Agent on Amazon Linux 2023 (Step-by-step)

This README documents, in precise detail, how to create a permanent Jenkins agent on an **Amazon Linux 2023 EC2** instance and connect it via the Jenkins GUI (Manage Jenkins → Nodes → New Node). The flow uses SSH authentication with an EC2 key pair and the Jenkins `SSH Username with private key` credentials provider.

> Target audience: developers or DevOps engineers who want a reliable, GUI-driven Jenkins agent on Amazon Linux. This README is ready to be pushed to a GitHub repository as `README.md`.

---

## Table of contents

1. Prerequisites
2. Launch an Amazon Linux 2023 EC2 instance
3. Create or import an SSH key pair
4. Create a security group that allows SSH from the Jenkins controller
5. Connect to the instance (SSH)
6. Install Java on the EC2 instance
7. Prepare a directory for the Jenkins agent (optional)
8. Add the SSH credential in Jenkins (exact field values)
9. Add a new permanent node in Jenkins (exact field values)
10. Post-connection verification and tests
11. Troubleshooting
12. Optional enhancements and hardening
13. Example `user-data` (bootstrap) script

---

## 1. Prerequisites

* An AWS account and permission to create EC2 instances and security groups.
* A running Jenkins controller with administrative access to the Jenkins GUI.
* Basic terminal/SSH knowledge.
* Your local machine (or CI) able to access the Jenkins controller and the EC2 instance network.

---

## 2. Launch an Amazon Linux 2023 EC2 instance (console quick steps)

1. Open the AWS console → EC2 → Launch Instance.
2. Name the instance: `jenkins-agent-amazon-linux`.
3. AMI: **Amazon Linux 2023 (AL2023)**.
4. Instance type: `t3.micro` (or `t3.small` if you expect heavier workloads).
5. Key pair: choose an existing key or create a new one. (If creating new, download and secure the `.pem` file.)
6. Network: enable **Auto-assign Public IP** (if you plan to SSH from the public internet).
7. Security group: add SSH (port 22) rule limited to the public IP address of your Jenkins controller. See section 4 for details.
8. Advanced: optionally add user-data (bootstrap script). See section 13 for an example.
9. Launch the instance.

Record the instance's **Public IPv4** address — you will need it when adding the node in Jenkins.

---

## 3. Create or import an SSH key pair

You can create the key pair either in the AWS console or locally and import the public key.

**Console method (recommended for quick setup)**

* EC2 → Key Pairs → Create key pair → choose `RSA` or `ED25519` → Download `.pem` file.

**Local method (recommended if you manage keys locally)**

```bash
ssh-keygen -t ed25519 -f ~/.ssh/jenkins-ec2 -C "jenkins-agent-key"
# Upload the public key (jenkins-ec2.pub) to AWS: EC2 -> Key Pairs -> Import key pair
```

Store and protect your private key. Example permission fix:

```bash
chmod 400 mykey.pem
```

---

## 4. Create a Security Group for SSH

Create or modify a security group with the following inbound rule:

* Type: **SSH**
* Protocol: TCP
* Port range: 22
* Source: **<JENKINS_CONTROLLER_PUBLIC_IP>/32**

Important: do not set Source to `0.0.0.0/0` for production. Restrict SSH access to only the Jenkins controller IP (or your management IP range).

---

## 5. Connect to the instance via SSH

From your local terminal, connect to the EC2 instance (replace placeholders):

```bash
ssh -i /path/to/mykey.pem ec2-user@<EC2_PUBLIC_IP>
```

If you cannot connect, verify:

* The security group allows port 22 from your client IP.
* The EC2 instance has a public IP (or reachable via private networking/VPN).
* The key file permissions are `chmod 400`.

---

## 6. Install Java on the EC2 instance

Jenkins agent requires Java to run `agent.jar`.

On Amazon Linux 2023, install Amazon Corretto (Java 17 recommended):

```bash
sudo dnf update -y
sudo dnf install -y java-17-amazon-corretto
java -version
```

Confirm the output includes a Java 17 runtime.

---

## 7. Prepare a directory for the Jenkins agent (optional)

Jenkins will create the remote root directory automatically when connecting over SSH in most cases. If you want to precreate it and set permissions, run:

```bash
sudo mkdir -p /home/ec2-user/jenkins-agent
sudo chown ec2-user:ec2-user /home/ec2-user/jenkins-agent
```

This is optional but can reduce permission-related issues on first connection.

---

## 8. Add the SSH Credential in Jenkins (exact field values)

From the Jenkins GUI:

**Jenkins → Manage Jenkins → Credentials → System → Global credentials (unrestricted) → Add Credentials**

Fill the form exactly as below:

* **Domain**: *Global credentials (unrestricted)* (default)
* **Kind**: `SSH Username with private key`
* **Scope**: `Global (Jenkins, nodes, items, all child items)`
* **Username**: `ec2-user`
* **Treat username as secret?**: Leave unchecked
* **Private Key**: `Enter directly` → paste the entire contents of your private key file (`.pem`) into the text box. Include the `-----BEGIN...` and `-----END...` lines.
* **Passphrase / Password**: leave blank (not used for key pair)
* **ID**: `ec2-ssh-key` (suggested; optional but recommended)
* **Description**: `SSH key for Amazon Linux EC2 Jenkins agent`

Click **Create**.

> Note: if the Jenkins controller cannot paste the key (policy or UI restriction), consider using `From the Jenkins master ~/.ssh` or create the credential via Jenkins CLI or Job DSL with appropriate secure storage.

---

## 9. Add a new permanent node in Jenkins (exact field values)

From the Jenkins GUI:

**Manage Jenkins → Manage Nodes and Clouds → New Node**

Create a node using the following precise values.

### Node general fields

* **Name**: `amazon-linux-agent` (no spaces recommended)
* **Description**: `Permanent Jenkins agent on Amazon Linux EC2`
* **# of executors**: `1`
* **Remote root directory**: `/home/ec2-user/jenkins-agent`
* **Labels**: `amazon-linux ec2 permanent-agent` (space-separated labels)
* **Usage**: `Use this node as much as possible`

### Launch method

* **Launch method**: `Launch agents via SSH`

### SSH settings

* **Host**: `<EC2_PUBLIC_IP>` (replace with your instance public IPv4)
* **Credentials**: select the credential you created earlier: `ec2-user (SSH key for Amazon Linux EC2 Jenkins agent)` or the ID `ec2-ssh-key`.
* **Host Key Verification Strategy**: `Non verifying Verification Strategy` (recommended initial choice to avoid fingerprint mismatches)

### Availability

* **Availability**: `Keep this agent online as much as possible`

Leave all Node Properties (Environment variables, Tool Locations, Disk monitoring, etc.) blank unless you have specific needs.

Click **Save**.

---

## 10. Post-connection verification and tests

After clicking **Save**, Jenkins will attempt to connect and start the agent. Verify the following in the Jenkins GUI:

1. Node shows **green** and **online** status.
2. Node logs (click the node, then `Log`) show successful SSH session and `agent.jar` start.
3. Run a simple job targeting this node label (create a freestyle or pipeline that uses `node('amazon-linux') { sh 'uname -a; java -version' }`).
4. Confirm the job runs successfully on the agent and returns expected output.


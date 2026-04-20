# IAM Automation Lab

## Repository contents

```
.
├── iam-lab-stack.yaml          # CloudFormation template
├── deployment-config.yaml      # GitSync deployment configuration
└── README.md                   # This file
```

---

## Architecture overview

```
GitHub repo (main branch)
       │
       │  GitSync (auto-deploy on push)
       ▼
CloudFormation Stack
       │
       ├── Secrets Manager ──────────────────────── Auto-generated temp password
       │                                             (shared by all 3 users)
       │
       ├── EC2LabGroup ──────────────────────────── ec2:Describe*, ec2:RunInstances
       │       ├── ec2-user1  (member, no overrides)
       │       └── ec2-user2  (member + inline Deny on RunInstances)
       │
       └── S3LabGroup ───────────────────────────── s3:ListAllMyBuckets, s3:GetObject
               └── s3-user    (member, no overrides)
```

---

## Task 1 — Deployment via GitSync

### Step 1 — Push files to GitHub

Create a GitHub repository and push all three files to the `main` branch.

```bash
git init
git remote add origin https://github.com/<your-username>/<your-repo>.git
git add iam-lab-stack.yaml deployment-config.yaml README.md
git commit -m "Initial IAM lab stack"
git push -u origin main
```

### Step 2 — Connect GitSync in AWS CloudFormation

1. Open the **AWS CloudFormation console** → **Stacks** → **Create stack** → **With new resources**.
2. Under *Specify template*, select **Sync from Git**.
3. Click **Connect to Git** → choose **GitHub** → authenticate via GitHub App or personal access token (PAT) with `repo` scope.
4. Select your repository, branch (`main`), and set:
   - **Template file path:** `iam-lab-stack.yaml`
   - **Deployment file path:** `deployment-config.yaml`
5. Name the stack `iam-lab-stack`.
6. For the GitSync IAM service role, attach a policy granting at minimum:
   - `cloudformation:*`
   - `iam:*`
   - `secretsmanager:*`
7. Click **Create stack** and wait for `CREATE_COMPLETE`.

Any future push to `main` will automatically trigger a re-deployment — this is GitSync's core value.

### Step 3 — Retrieve the temporary password

After the stack reaches `CREATE_COMPLETE`, run the following CLI command (also shown in stack Outputs):

```bash
aws secretsmanager get-secret-value \
  --secret-id "iam-lab/temp-console-password" \
  --region <your-region> \
  --query SecretString \
  --output text
```

The returned JSON will contain:
```json
{"username": "iam-lab-users", "password": "Ab3#xKp9..."}
```

Use the `password` value when logging in as each IAM user.

### Step 4 — Find the console sign-in URL

```bash
aws sts get-caller-identity --query Account --output text
```

The sign-in URL is:
```
https://<account-id>.signin.aws.amazon.com/console
```

---

## Resources created

| Resource | Type | Purpose |
|---|---|---|
| `iam-lab/temp-console-password` | Secrets Manager Secret | Auto-generated shared temp password |
| `EC2LabGroup` | IAM Group | Grants EC2 list + describe + run permissions |
| `S3LabGroup` | IAM Group | Grants S3 list + read-only permissions |
| `ec2-user1` | IAM User | EC2 group member; can list and create instances |
| `ec2-user2` | IAM User | EC2 group member; **denied** from creating instances |
| `s3-user` | IAM User | S3 group member; read-only S3 access |

All users are created with `PasswordResetRequired: true`, forcing a password change on first login.

---

## Task 2 — Console verification and screenshots

Log in to the AWS Console as each IAM user and capture the following screenshots. Each user requires **2 screenshots minimum** (one success, one failure). The login/password-reset screen is a recommended bonus screenshot.

---

### ec2-user1

**How to log in:**
1. Go to `https://<account-id>.signin.aws.amazon.com/console`
2. Enter username: `ec2-user1`
3. Enter the password retrieved from Secrets Manager
4. You will be prompted to set a new password — do so

**Screenshot 1 — EC2 success**
- Navigate to **Services → EC2 → Instances**
- You should see a list of instances (DescribeInstances is allowed)
- Capture the instance list page showing instances loaded successfully

**Screenshot 2 — S3 failure**
- Navigate to **Services → S3 → Buckets**
- You should see an `Access Denied` error (no S3 permissions)
- Capture the error message from the console

Expected error text:
```
User: arn:aws:iam::<account-id>:user/ec2-user1 is not authorized to perform: s3:ListAllMyBuckets
```

---

### ec2-user2

**How to log in:**
1. Sign out of the previous session or use an incognito window
2. Sign in with username: `ec2-user2` and the same temp password

**Screenshot 1 — EC2 list success**
- Navigate to **Services → EC2 → Instances**
- The instance list should load (DescribeInstances is allowed via group)
- Capture the instances page

**Screenshot 2 — EC2 create denied**
- Click **Launch instances** from the EC2 Instances page
- Fill in the AMI, instance type, and proceed to launch
- You should receive an authorization error when AWS attempts to call RunInstances
- Capture the error dialog or banner

Expected error text:
```
You are not authorized to perform this operation.
Error: UnauthorizedOperation — explicit deny in identity-based policy
```

> The phrase "explicit deny" in the error confirms your inline Deny policy is working correctly. This is a key IAM concept to explain during the live review.

**Screenshot 3 — S3 failure (bonus)**
- Navigate to **Services → S3 → Buckets**
- Capture the Access Denied response

---

### s3-user

**How to log in:**
1. Sign out and sign in as `s3-user` with the temp password

**Screenshot 1 — S3 success**
- Navigate to **Services → S3 → Buckets**
- You should see a list of S3 buckets (ListAllMyBuckets is allowed)
- Click into a bucket to confirm object listing works (ListBucket + GetObject)
- Capture the bucket list or the object list inside a bucket

**Screenshot 2 — EC2 failure**
- Navigate to **Services → EC2 → Instances**
- You should see an authorization error (no EC2 permissions)
- Capture the error message

Expected error text:
```
User: arn:aws:iam::<account-id>:user/s3-user is not authorized to perform: ec2:DescribeInstances
```

---

### Screenshot summary table

| User | Action | Expected result | Screenshot # |
|---|---|---|---|
| ec2-user1 | View EC2 instances | Success — instances listed | 1 |
| ec2-user1 | View S3 buckets | Failure — Access Denied | 2 |
| ec2-user2 | View EC2 instances | Success — instances listed | 3 |
| ec2-user2 | Launch EC2 instance | Failure — explicit deny (RunInstances) | 4 |
| s3-user | View S3 buckets | Success — buckets listed | 5 |
| s3-user | View EC2 instances | Failure — Access Denied | 6 |

---

## IAM concepts demonstrated

**Group-based permissions** — permissions are attached to groups, not individual users. Both EC2 users inherit their access from `EC2LabGroup`. Changing the group policy instantly affects all members without touching user-level config.

**Explicit Deny overrides Allow** — in AWS IAM policy evaluation, an explicit `Effect: Deny` always wins over any `Effect: Allow`, regardless of where the Allow comes from (group, managed policy, or resource policy). `ec2-user2` is in a group that allows `RunInstances`, but the inline Deny on the user itself blocks it. The console error surfaces this as "explicit deny in identity-based policy."

**Principle of least privilege** — each group grants only what is needed. `S3LabGroup` does not grant `s3:PutObject`, `s3:DeleteObject`, or bucket management actions. `EC2LabGroup` does not grant `s3:*` at all.

**Secrets Manager for credential management** — the temp password is never hardcoded in the template or visible in stack events. CloudFormation resolves the `{{resolve:secretsmanager:...}}` dynamic reference at deploy time. Rotating the secret would allow redeploying fresh credentials without touching the template.

**Password reset on first login** — `PasswordResetRequired: true` in the `LoginProfile` block forces each user to set their own password immediately. This is a security best practice so the shared temp password cannot persist.

**GitSync as infrastructure version control** — changes to the CloudFormation template go through Git. The commit history is the audit trail. GitSync re-deploys automatically on push, eliminating manual stack updates and the configuration drift that comes with them.

---

## Deliverables checklist

- [x] GitHub repository with `iam-lab-stack.yaml`, `deployment-config.yaml`, and `README.md`
- [x] GitSync enabled and stack successfully deployed
- [x] Secrets Manager secret created and temp password retrievable
- [x] Three IAM users created with group assignments and forced password reset
- [x] ec2-user2 denied from creating EC2 instances (explicit deny policy)
- [x] 6 screenshots (2 per user) demonstrating access success and failure

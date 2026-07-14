# Deploying housing_regression_MLE to AWS ECS Fargate

Documentation of the cloud deployment for the housing regression project: FastAPI service + Streamlit dashboard, both running on ECS Fargate behind a shared Application Load Balancer. Written up from implementation notes, including the reasoning behind each AWS service choice, the actual errors hit along the way, and what's still outstanding.

**Region used throughout:** `us-east-2` (Ohio)
**Account:** IAM user `Rishav2` created with Management Console + CLI access

---

## Architecture at a glance

```
Internet → ALB (port 80)
             ├── default rule → housing-api-tg (port 8000) → FastAPI container
             └── /dashboard* → housing-streamlit-tg (port 8501) → Streamlit container

Both containers run as ECS Fargate tasks inside housing-api-cluster-ecs,
pull their images from ECR, and read/write model artifacts from S3.
```

---

## Step 1 — IAM user and CLI setup

**Why an IAM user instead of using root:** root has no permission boundaries i.e if its credentials ever leaked, the entire account is compromised, billing included. IAM users can be scoped, rotated, and deleted independently of the account itself. This is the standard first step for any AWS work, not project-specific.

**Steps taken:**
1. Create a new IAM user (`Rishav2`), granted access to the Management Console
2. Install AWS CLI v2 via PowerShell (as Administrator):
   ```powershell
   Start-Process msiexec.exe -Wait -ArgumentList '/i https://awscli.amazonaws.com/AWSCLIV2.msi /qn'
   ```
   or install from the official documentation directly.
3. Configure the CLI:
   ```powershell
   aws configure
   ```
   Prompts: IAM user → Create access key → AWS Access Key ID (terminal) → AWS Secret Access Key → default region
4. Confirm Docker Desktop is running:
   ```powershell
   docker ps
   ```

### Error hit: region mismatch (`us-east-2` vs `eu-west-2`)
The original reference material (a tutorial repo) used `eu-west-2` (London) throughout — bucket, ECR, task defs. This deployment intentionally standardized on `us-east-2` instead. This meant every region reference in every later step had to be manually re-checked and swapped, including inside the task definition JSON files, which is easy to miss since it's not just a CLI flag, it's baked into `image` URIs and `logConfiguration` blocks too.

**Lesson:** decide on a region *before* touching any service, and grep every config file for the wrong one before registering anything.

### Error hit: `docker ps` — Docker Desktop engine not running
```
failed to connect to the docker API at npipe:////./pipe/dockerDesktopLinuxEngine
```
**Fix:** Docker Desktop app wasn't actually launched (CLI installed fine, engine wasn't running). Just starting the Docker Desktop app and waiting for the tray icon to finish initializing resolved it.

---

## Step 2 — S3 bucket

**Why S3 here:** the containers need to read a trained model and processed dataset at runtime, and write nothing back except maybe predictions/logs. S3 is the standard choice for this, cheap, durable, and IAM-controllable at the object level, so the container's permissions can be scoped tightly (see IAM section below) rather than mounting a filesystem or baking data into the image (which would make the image huge and force a rebuild every time the model changes).

**Steps:**
```powershell
aws s3 mb s3://housing-regression-data --region us-east-2
```
`mb` = **make bucket**, one of several Unix-inspired shorthand commands in the `aws s3` CLI (alongside `cp`, `mv`, `rm`, `ls`, `sync`, `rb`).

Upload data and models:
```powershell
aws s3 cp data/processed/ s3://housing-regression-data-rishau/processed/ --recursive
aws s3 cp models/ s3://housing-regression-data/models/ --recursive
```
`--recursive` is required because `cp` by default expects a single file → single destination; without it, pointing at a folder does nothing useful. It walks the local folder tree and mirrors the structure into the bucket.

### Error hit: bucket name already taken
`housing-regression-data` alone was already claimed globally (S3 bucket names are unique across *all* of AWS, not just this account). **Fix:** appended a suffix (`-rishav`) and used that name consistently in every later reference (task def env vars, IAM policy ARNs).

### Error hit: typo'd prefix (`mmodels` instead of `models`)
Fixed by copying to the correct prefix and deleting the old one — S3 has no true "rename," since prefixes aren't real folders, just parts of the object key:
```powershell
aws s3 mv s3://housing-regression-data/mmodels/ s3://housing-regression-data/models/ --recursive
```
`mv` on S3 is really copy-then-delete done as one command.

---

## Step 3 — ECR repositories and Docker images

**Why ECR:** described in the notes as "a tool like Docker Hub that is a fully managed, highly scalable AWS service that allows developers to securely store, manage, and deploy container images" (Amazon Elastic Container Repository). The key reason to use ECR specifically instead of Docker Hub: ECS/Fargate can pull from it using IAM roles directly, with no separate registry credentials to manage inside AWS — auth flows through the same IAM system as everything else.

**Steps:**
```powershell
aws ecr create-repository --repository-name housing-api --region us-east-2
aws ecr create-repository --repository-name housing-streamlit --region us-east-2
```

**Authenticate Docker to ECR:**
```powershell
aws ecr get-login-password --region us-east-2 | docker login --username AWS --password-stdin "$ACCOUNT_ID.dkr.ecr.us-east-2.amazonaws.com"
```
How this works: ECR requires authentication before push/pull unlike a public registry. The first command asks AWS for a temporary auth token, printed to stdout. The pipe (`|`) feeds that token into `docker login` as the password, using the literal username `AWS` (not a placeholder — that's fixed by ECR). The `docker login` command then logs into that specific ECR registry endpoint using the piped token.

**Build and tag the image:**
```powershell
docker build -t housing-api -f Dockerfile .
docker tag housing-api:latest "$ACCOUNT_ID.dkr.ecr.us-east-2.amazonaws.com/housing-api:latest"
docker push "$ACCOUNT_ID.dkr.ecr.us-east-2.amazonaws.com/housing-api:latest"
```
Docker images are referenced by **registry + name + tag**. Locally the image is just `housing-api:latest`. ECR won't accept a push to that name because it doesn't know which registry it belongs to. `docker tag` creates a second name pointing at the same image (no rebuild, no duplication of data); this new name is the full path ECR expects. Same process repeated for the Streamlit image, swapping the Dockerfile and repo name.

### Error hit: `docker login` — 400 Bad Request
```
Error response from daemon: login attempt to https://...amazonaws.com/v2/ failed with status: 400 Bad Request
```
This is a known issue piping `get-login-password` directly into `docker login --password-stdin` on **Windows PowerShell**  long token strings can get mangled by the pipe. **Fix:** store the token in a variable first, then pass it explicitly, rather than piping:
```powershell
$pw = aws ecr get-login-password --region us-east-2
docker login --username AWS --password $pw "$ACCOUNT_ID.dkr.ecr.us-east-2.amazonaws.com"
```

### Error hit: `docker push` — 403 Forbidden
```
unknown: unexpected status from HEAD request ... 403 Forbidden
```
Root cause: the ECR auth token from `docker login` is temporary (~12 hours) and had expired after a long session with pauses in between. **Fix:** re-run the `docker login` steps immediately before retrying the push.

### Error hit: `Tags: None` on `aws ecr describe-images`
After fixing a bug in the image and rebuilding, the push appeared to succeed but the repository showed no `latest` tag. Root cause: `docker tag` was skipped/not rerun after the rebuild, so `docker push` had nothing properly tagged to push. **Fix:** always run `docker build → docker tag → docker push` as one complete sequence after any code change — never assume a previous tag still applies to a freshly built image.

---

## Step 4 — IAM roles

Two separate roles were needed, deliberately kept apart because they serve two different moments in the container lifecycle:

### `ecsTaskExecutionRole`
**Why this exists:** ECS itself needs permission to act *before your application code even starts* pulling the image from ECR and writing container logs to CloudWatch. Without it, tasks fail immediately at startup.

Steps:
```
IAM → Roles → Create role
Trusted entity type: AWS service
Use case: Elastic Container Service → Elastic Container Service Task
```
The "ECS Task" choice matters as it sets up the **trust policy** controlling who can assume the role. Picking this means only ECS tasks can assume it, not EC2 instances or Lambda.
```
Attach policy: AmazonECSTaskExecutionRolePolicy (AWS-managed)
Name: ecsTaskExecutionRole → Create
```

### `ecs_s3_access`
**Why a second, separate role:** once the container is *running*, the application code (inside `src/api/main.py`) needs its own permission to actually read/write S3 objects. A completely different concern from "can ECS start the container at all." Conflating these two would mean either over-granting execution-time permissions to your app, or under-granting startup permissions.

Custom policy (`housing-s3-policy`):
```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": ["s3:GetObject", "s3:PutObject"],
      "Resource": "arn:aws:s3:::housing-regression-data-rishau/*"
    },
    {
      "Effect": "Allow",
      "Action": "s3:ListBucket",
      "Resource": "arn:aws:s3:::housing-regression-data-rishau"
    }
  ]
}
```
Two statements are needed because S3's permission model treats "list what's in a bucket" as a *bucket-level* action (ARN with no trailing `/*`), separate from "get/put individual objects" (ARN with `/*`, scoped to objects inside).

Role creation: same trusted-entity choice as above (ECS Task), attach `housing-s3-policy`, name it `ecs_s3_access`.

**Note on ARNs:** `executionRoleArn` → `ecsTaskExecutionRole`; `taskRoleArn` → `ecs_s3_access`. Mixing these two up in the task definition is one of the most common causes of tasks that either fail to start at all (wrong execution role) versus tasks that start fine but can't reach S3 (wrong task role).

---

## Step 5 — ECS cluster

**What a cluster actually is:** not a server or machine a logical grouping/namespace for services and tasks. It's an organizational boundary AWS uses to group related services, track resource usage, and scope IAM permissions/monitoring. With Fargate, there's no infrastructure being provisioned here (unlike EC2-backed ECS, which used to require actually managing EC2 instances).

```
ECS → Cluster → Create cluster
Name: housing-api-cluster-ecs
Infrastructure: AWS Fargate (serverless)
```

---

## Step 6 — Security groups

**Why two groups instead of one:** the ALB and the containers need different exposure levels. The ALB is the only thing that should be reachable from the public internet; the containers should only ever be reachable *through* the ALB.

### `alb-sg`
```
Inbound rule: Type HTTP, Port 80, Source Anywhere (0.0.0.0/0)
```

### `ecs-tasks-sg`
```
Inbound rule 1: Custom TCP, Port 8000, Source = alb-sg (the security group itself, not an IP range)
Inbound rule 2: Custom TCP, Port 8501, Source = alb-sg
```
**Why 8000 and 8501 specifically:** not a security convention these are exactly the ports the application processes bind to internally (FastAPI/uvicorn on 8000 per the Dockerfile CMD; Streamlit's own default port, 8501). Every layer downstream (task def containerPort, target group port, this security group rule) has to agree on these same numbers, since they all describe one continuous path from ALB to listening process.

**Why "source = security group" instead of a fixed IP range:** AWS evaluates this dynamically "allow traffic from any network interface that currently has `alb-sg` attached," rather than resolving to a static CIDR block once. This means the rule keeps working automatically even if the ALB's underlying IPs change (which they can, across AZs), without ever needing manual updates.

**Why not just open 8000/8501 to the internet directly:** doing so would let anyone bypass the ALB entirely — skipping its health checks, path-based routing, and any logging at that layer.

---

## Step 7 — Target groups

**Why they exist:** a target group is where the ALB actually sends traffic. A pool of IPs plus a health check config. It's a separate concept from the load balancer itself because one ALB can route to multiple target groups based on path (API vs. dashboard, in this case).

```
EC2 → Target Groups → Create target group
Target type: IP addresses   (required for Fargate — tasks get dynamic ENI IPs, not fixed EC2 instance IDs)
housing-api-tg:        Protocol/Port HTTP/8000, health check path /health
housing-streamlit-tg:  Protocol/Port HTTP/8501, health check path /dashboard
```

---

## Step 8 — Application Load Balancer

**Why an ALB at all:** the two containers are private. Fargate tasks don't get a stable, bookmarkable public IP. The ALB is the single public-facing door: a permanent DNS name that internally routes requests to whichever container should handle them, based on the URL path, and continuously health-checks containers to pull unhealthy ones out of rotation automatically.

```
EC2 → Load Balancer → Create load balancer → Application Load Balancer
Name: housing-api-ALB
Scheme: Internet-facing
VPC: default
Mapping: select at least 2 subnets in different Availability Zones
Security group: remove default, select alb-sg
Listener: HTTP:80 → default action: forward to housing-api-tg
Create
```

**Listener rule for the dashboard** - this is what makes `/dashboard*` go to Streamlit while everything else falls through to the API's default rule:
```
ALB → Listeners tab → click HTTP:80 listener → Manage rules (Add Rule)
Condition: Path is /dashboard*
Action: Forward to housing-streamlit-tg
Priority: any number
```

### Error hit: empty target group dropdown when creating the ALB listener
No targets appeared as selectable options when configuring the listener. Root cause: **VPC mismatch** - the target groups had been created against a different VPC than the one selected for the ALB. Fixed by confirming both were pointed at the same default VPC before proceeding. (General debugging approach: `aws elbv2 describe-target-groups --query 'TargetGroups[].{Name:...,VpcId:...}'` to compare VPC IDs directly rather than guessing from the console UI.)

### Error hit: target stuck in `unused` state, reason `Target.NotInUse`
```json
"State": "unused",
"Reason": "Target.NotInUse",
"Description": "Target is in an Availability Zone that is not enabled for the load balancer"
```
Root cause: the ALB was only configured to span 2 of the VPC's 3 subnets (`us-east-2a`, `us-east-2b`), but Fargate placed a task in the third AZ (`us-east-2c`), which the ALB had no path to reach at all. This masqueraded earlier as an "unhealthy - Request timed out" event, since a health check can't succeed if the ALB has no route to the target's AZ in the first place.

**Fix:** expand the ALB's subnet coverage to include all AZs the ECS service could place tasks into:
```powershell
aws elbv2 set-subnets --load-balancer-arn $albArn --subnets subnet-1 subnet-2 subnet-3 --region us-east-2
```
**Lesson:** the ALB's subnet selection and the ECS service's `--network-configuration` subnet list must cover the *same* set of AZs if the ECS service can place a task somewhere the ALB doesn't span, that target will silently sit unreachable regardless of container health.

---

## Step 9 — Task definitions

Both task def JSONs need: `image` pointing at the correct ECR URI (matching region!), `executionRoleArn` → `ecsTaskExecutionRole`, `taskRoleArn` → `ecs_s3_access`, `logConfiguration.options.awslogs-region` matching the deployment region, and (for Streamlit specifically) `API_URL` pointing at the ALB's real DNS name — copied directly from the ALB detail page after Step 8, since Streamlit needs to know where to send prediction requests.

```powershell
aws ecs register-task-definition --cli-input-json file://housing-api-task-def.json --region us-east-2
aws ecs register-task-definition --cli-input-json file://streamlit-task-def.json --region us-east-2
```

### Error hit: malformed JSON — `logConfiguration` nested inside `environment`
While manually adding a `logConfiguration` block to `streamlit-task-def.json` (missing from the original template), it was pasted as if it were another entry inside the `environment` array, rather than as a sibling property of the container definition object. This produced invalid JSON (missing comma, wrong nesting level) that would have failed registration outright.
**Fix:** `logConfiguration` sits at the same level as `name`, `image`, `portMappings`, and `environment`, not inside any of them.

### Error hit: task placement failure - missing CloudWatch log group
```
ResourceInitializationError: ... CreateLogStream ... ResourceNotFoundException: The specified log group does not exist.
```
The task definition referenced `/ecs/housing-streamlit` as a log group, but nothing actually creates that log group automatically — registering a task definition doesn't provision the CloudWatch resources it references. **Fix:**
```powershell
aws logs create-log-group --log-group-name /ecs/housing-streamlit --region us-east-2
```
then force a redeployment. **Lesson:** any log group named in `logConfiguration` needs to be created explicitly before a task using that config can start.

### Error hit: updating the task def revision doesn't update the running service
After fixing the image reference and registering a new revision (`:2`), the service kept failing with the same old error. Root cause: `--force-new-deployment` alone restarts tasks using whatever revision the service is **currently pinned to**, it does not automatically pick up a newer task definition revision. **Fix:** explicitly point the service at the new revision:
```powershell
aws ecs update-service --cluster housing-api-cluster-ecs --service housing-streamlit-service --task-definition housing-streamlit:2 --force-new-deployment --region us-east-2
```

---

## Step 10 — ECS services

**What "creating a service" actually means, vs. just running a task:** a bare Fargate task (`aws ecs run-task`) is one-shot i.e if it crashes or stops, nothing restarts it. A service is a wrapper around tasks that adds self-healing (automatically launches a replacement to maintain desired count if a task dies) and load balancer integration (automatically registers/deregisters tasks with a target group as they start/stop), plus rolling deployments (starting new tasks and draining old ones gracefully on redeploy, instead of killing everything at once).

```
ECS → Cluster → housing-api-cluster-ecs → Service tab → Create
Compute options: Launch type Fargate
Task definition family: housing-api-task-ecs, revision: latest
Service name: housing-api-service
Desired tasks: 1
Networking: default VPC, all subnets, existing ecs-tasks-sg, Public IP: turned on
Load balancing: Application Load Balancer → existing housing-api-ALB
Container to load balance: housing-api:8000
Target group: existing housing-api-tg
```
Repeated for Streamlit, swapping task def family, service name, container port (8501), and target group.

Equivalent CLI, run in one block to avoid stale shell variables:
```powershell
$vpcId = (aws ec2 describe-vpcs --filters "Name=isDefault,Values=true" --query "Vpcs[0].VpcId" --output text --region us-east-2)
$subnets = (aws ec2 describe-subnets --filters "Name=vpc-id,Values=$vpcId" --query "Subnets[].SubnetId" --output text --region us-east-2) -split "\s+"
$ecsSgId = (aws ec2 describe-security-groups --filters "Name=group-name,Values=ECS-tasks-SG" --query "SecurityGroups[0].GroupId" --output text --region us-east-2)
$apiTgArn = (aws elbv2 describe-target-groups --names housing-api-tg --query "TargetGroups[0].TargetGroupArn" --output text --region us-east-2)

aws ecs create-service `
  --cluster housing-api-cluster-ecs `
  --service-name housing-api-service `
  --task-definition housing-api-task-ecs `
  --desired-count 1 `
  --launch-type FARGATE `
  --network-configuration "awsvpcConfiguration={subnets=[$($subnets[0]),$($subnets[1])],securityGroups=[$ecsSgId],assignPublicIp=ENABLED}" `
  --load-balancers "targetGroupArn=$apiTgArn,containerName=housing-api,containerPort=8000" `
  --region us-east-2
```

### Error hit: `Cannot index into a null array`
PowerShell variables like `$subnets`, `$ecsSgId` only live for the life of a terminal session. After closing/reopening PowerShell or leaving a long gap, referencing them again returned null, and indexing into a null array (`$subnets[0]`) threw this error. **Fix:** re-run the full lookup block (`$vpcId`, `$subnets`, `$ecsSgId`, target group ARNs) in the same paste/session immediately before any command that uses them.

### Error hit: security group lookup returned `None`
```powershell
aws ec2 describe-security-groups --filters "Name=group-name,Values=ecs-tasks-sg"
```
returned nothing, despite the group existing. Root cause: **case sensitivity** — the actual group names were `ALB-SG` and `ECS-tasks-SG` (capitalized), but the filter searched for lowercase `ecs-tasks-sg`. AWS security group name filters are case-sensitive. **Fix:** confirmed actual names via `describe-security-groups --output table` first, then matched the filter exactly.

### Error hit: `Creation of service was not idempotent`
Attempting to `create-service` a second time for a service that had already been created successfully in an earlier attempt. This isn't a real failure — AWS correctly refusing to create a duplicate. **Fix (verification approach):**
```powershell
aws ecs describe-services --cluster housing-api-cluster-ecs --services <service-name> --query "services[0].{Status:status,Running:runningCount,Desired:desiredCount,Events:events[0:3]}" --output table
```
If `Status: ACTIVE`, `Running == Desired`, and the events show "reached a steady state," the service is already fine no action needed.

### Error hit (the big one): Streamlit target stuck unhealthy misdiagnosed twice before finding the real cause
This was the longest debugging chain in the whole deployment, worth documenting step by step since the first two hypotheses were both plausible but wrong.

**Symptom:** `housing-streamlit-service` showed `Running: 1 / Desired: 1` and `Status: ACTIVE`, but `describe-target-health` kept returning `unhealthy`, cycling through different reasons over time — `Target.Timeout` → `Target.FailedHealthChecks` — and the service kept restarting tasks in a loop, eventually hitting `deployment failed: tasks failed to start` from the circuit breaker. Visiting the dashboard through the ALB returned a **504 Gateway Timeout**.

**Hypothesis 1: AZ/subnet mismatch (partially correct, and a real bug that needed fixing):** an earlier round of this same investigation found a target sitting in `Target.NotInUse` / `"Description": "Target is in an Availability Zone that is not enabled for the load balancer"` the ALB only spanned 2 of 3 subnets, and Fargate had placed a task in the third, unreachable AZ. Fixed via `aws elbv2 set-subnets` to add the missing subnet. This was a genuine, separate bug but fixing it didn't resolve the unhealthy state; it just changed the failure mode from `unused` to `Target.Timeout`.

**Hypothesis 2 health check too strict for Streamlit's response time (reasonable, but not the actual cause):** the target group's health check had a 5-second timeout against the `/dashboard` path, which serves Streamlit's full rendered page rather than a lightweight ping. Two changes were made based on this theory:
- Loosened the timeout/threshold: `aws elbv2 modify-target-group --health-check-timeout-seconds 10 --health-check-interval-seconds 30 --healthy-threshold-count 3`
- Switched the health check path to Streamlit's dedicated lightweight endpoint: `/dashboard/_stcore/health` (returns a plain `ok` without running the app script — the equivalent of FastAPI's `/health` for Streamlit specifically)

Neither change fixed it. The target kept failing regardless of path or timeout a strong signal the problem wasn't about response *speed* at all.

**Actual root cause, found by testing connectivity directly instead of tuning health check settings further:**
```powershell
aws ec2 describe-network-interfaces --network-interface-ids <eni-id> --query "NetworkInterfaces[0].Groups"
```
This revealed the running Streamlit task's network interface had **`ALB-SG` attached instead of `ECS-tasks-SG`**. `ALB-SG` only allows inbound traffic on port 80 — it has no rule permitting port 8501 at all. So every health check to port 8501, from the very first deployment, was being silently dropped at the network layer, before it ever reached the container. The app itself had been working correctly the entire time (confirmed via clean "Uvicorn server started" / "You can now view your Streamlit app" logs on every task attempt) this was never an application-level problem.

**Likely origin of the bug:** in Step 10 (`create-service` for Streamlit), the `--network-configuration` almost certainly had `securityGroups=[$albSgId]` instead of `securityGroups=[$ecsSgId]` an easy variable mix-up given how many similarly-named `$...SgId`/`$...TgArn` PowerShell variables were juggled across a long session, compounded by variables repeatedly going null between terminal sessions (see the "Cannot index into a null array" errors below).

**Fix:**
```powershell
aws ecs update-service `
  --cluster housing-api-cluster-ecs `
  --service housing-streamlit-service `
  --network-configuration "awsvpcConfiguration={subnets=[subnet-01958bc2d75a541fd,subnet-025201f6710e3b658],securityGroups=[sg-07f92acdc0e202b69],assignPublicIp=ENABLED}" `
  --force-new-deployment `
  --region us-east-2
```
Hardcoding the actual subnet/security-group IDs directly (rather than relying on shell variables) avoided yet another null-array failure at this exact moment. After this, the newly launched task showed `ECS-tasks-SG` attached, and `describe-target-health` returned `healthy` within about a minute. Confirmed end-to-end via browser at `http://<alb-dns>/dashboard`.

**Lesson — the general one:** when a target is unhealthy and the *application logs look completely clean*, stop tuning health check paths/timeouts and go straight to verifying actual network reachability (security groups on the specific ENI, not just "the security group I think I attached"). A clean app log combined with a persistently unreachable target is a strong signal the problem is beneath the application layer entirely — routing/firewall, not app behavior. This would have been found faster by checking `describe-network-interfaces … Groups` as a first step alongside checking target health, rather than after two rounds of health-check-setting changes.

### Error hit (recurring): `Cannot index into a null array` on `--network-configuration`
Happened multiple times across this session whenever `$subnets`, `$ecsSgId`, etc. hadn't been re-populated in the current PowerShell session (see Step 10 above for the general explanation). By the end of the deployment, the reliable workaround adopted was to **stop relying on shell variables for security-group/subnet IDs in one-off fix commands** and just hardcode the literal IDs directly, confirmed via a fresh `describe-*` lookup immediately beforehand.

---

## Debugging approach used throughout

A general pattern emerged for chasing down deployment failures, worth reusing:

1. **Service level:** `aws ecs describe-services` → `status`, `runningCount` vs `desiredCount`, recent `events` (often names the exact problem, e.g. "unhealthy... Request timed out", "unable to place a task... ResourceInitializationError")
2. **Task level:** `aws ecs list-tasks --desired-status STOPPED` → `aws ecs describe-tasks` → `stoppedReason`, `exitCode` (note: ECS only retains detailed stopped-task records for roughly an hour after that, `describe-tasks` returns `"reason": "MISSING"`)
3. **Target group level:** `aws elbv2 describe-target-health` → `State` (`healthy`, `unhealthy`, `initial`, `unused` each implies a different cause; `unused` in particular usually means an AZ/subnet mismatch, not an application problem)
4. **Application logs:** `aws logs tail /ecs/<log-group> --since 1h` or `get-log-events` on a specific stream the actual ground truth of what the application was doing, useful for distinguishing "container never started" from "container ran fine and was cycled by a normal deployment"
5. **Network interface security groups (the layer most often skipped):** `aws ec2 describe-network-interfaces --network-interface-ids <eni-id> --query "NetworkInterfaces[0].Groups"` confirms which security group is *actually* attached to the running task, as opposed to which one you *intended* to attach via `--network-configuration`. This is the check that finally found the real Streamlit bug (see Step 10) after two rounds of health-check-setting changes that didn't help.

**General pattern that emerged:** if application logs are clean (app started, no errors) but the target group still won't go healthy no matter what health-check settings are changed, stop tuning the health check and go straight to verifying actual network reachability it's very likely a security group or routing issue sitting beneath the application layer entirely, not anything the app or the health check config can fix.

**Worth flagging:** an AI troubleshooting tool (Amazon Q) attributed an earlier, separate deployment rollback (on the API service) to "missing container-level health check" based on CPU/memory graph dips. Reading the actual application logs directly (`aws logs get-log-events`) told a different story the container was serving `200 OK` on every health check right up until a clean SIGTERM (exit code 143), consistent with a normal deployment replacing an old task, not an application failure. **Lesson:** CloudWatch metric-level inference is a starting hypothesis, not a conclusion always check the actual stopped-task reason and application logs before accepting a root-cause claim, especially from automated tooling.

---

## What's left / not yet done

- **CI/CD (GitHub Actions) is not covered in this document.** The repo already has a `.github/workflows` directory from the original tutorial reference. It should be reviewed before writing a custom pipeline, since it may already encode the expected build/push/deploy steps (image tags, which Dockerfile maps to which ECR repo). Required secrets: `AWS_ACCESS_KEY_ID`, `AWS_SECRET_ACCESS_KEY`, `AWS_REGION` (`us-east-2`) — set as **Secrets**, not Variables, since the access key pair grants `AdministratorAccess`-level control over the account.
- **Streamlit deployment is now confirmed fully working end-to-end** both target groups show `healthy`, and the dashboard loads and serves predictions through `http://<alb-dns>/dashboard`. The actual blocker turned out to be the wrong security group attached to the service (see Step 10 for the full debugging chain), not the AZ mismatch or health check settings that were fixed along the way those were real but secondary issues.
- **Health check path for Streamlit is now `/dashboard/_stcore/health`** (Streamlit's built-in lightweight health endpoint) rather than the original `/dashboard` worth keeping this in mind if the health check config is ever recreated from scratch, since `/dashboard` alone renders the full app and is a slower, heavier check than necessary.
- **Container-level `healthCheck` block** was suggested by Amazon Q but not conclusively shown to be necessary the ALB target group health check is already doing this job. Worth revisiting only if target-group-level health checks prove insufficient in practice.
- **Model/data caching in `app.py` not yet verified** if Streamlit reloads the model/data from S3 on every script rerun rather than using `@st.cache_resource`/`@st.cache_data`, real users could see the same kind of slowness that was initially (incorrectly) suspected as the health check problem. Worth checking now that the actual bug is fixed, since it's a real potential performance issue independent of what caused the deployment failures.
- **Dangling/duplicate IAM access keys** at least one access key pair was generated without its secret being saved, leaving an orphaned "Active" key with no known secret. Should be cleaned up (`aws iam delete-access-key`) once a working key pair is confirmed in use, since unused live credentials are a minor standing security risk.
- **`AdministratorAccess` on `Rishav2`** is intentionally broad for this learning project but should eventually be scoped down to only the specific ECS/ECR/S3/IAM/ELB actions actually used, especially before this same user/key pair is reused in a CI/CD pipeline.
- **No automated tests or `pytest` step** confirmed as part of the deployment flow — worth checking whether the existing GitHub Actions workflow already runs tests before build/push.
- **No custom domain / HTTPS** — the ALB currently serves plain HTTP on port 80 with its raw AWS-generated DNS name; adding a Route 53 domain + ACM certificate + HTTPS listener is a natural next step but out of scope here.
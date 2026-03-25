# Lambda Local + Remote Debugging (Breakpoints) — 2026-02-11

## Goal
Find a practical way to debug Lambda execution with breakpoints for `filescience-dev-valkey-queue-processor`, including both local and cloud-backed workflows.

## Key Findings
- AWS now supports **remote cloud debugging** for Lambda through **AWS Toolkit for VS Code** (breakpoints, step-through, variable inspection).
- Classic **local debugging** via **AWS SAM CLI** (`sam local invoke` / `sam local start-api` + `--debug-port`) still works and is useful for tight local loops.
- This repo currently deploys with `make build` + Terragrunt/OpenTofu and **does not contain a SAM template**, so SAM local debugging requires adding a small template or a local runner shim.

## Recommended Primary Path (Fastest for this repo)
Use **AWS Toolkit remote debugging** first.

Why:
- No need to repackage this repo into SAM format first.
- Lets you debug real cloud behavior (IAM/VPC/runtime/layers) that local emulation can miss.
- Works for Python Lambda runtimes on Amazon Linux 2023.

## Workflow A — Remote Debug in VS Code (Cloud Function, Local Breakpoints)
1. Install/update AWS Toolkit for VS Code (version 3.69.0+).
2. Ensure IAM permissions include:
   - `lambda:GetFunction`, `lambda:GetFunctionConfiguration`, `lambda:UpdateFunctionConfiguration`, `lambda:GetLayerVersion`, `lambda:InvokeFunction`
   - `iot:OpenTunnel`, `iot:RotateTunnelAccessToken`, `iot:CloseTunnel`, `iot:ListTunnels`
3. In AWS Explorer, open Lambda list and select `filescience-dev-valkey-queue-processor`.
4. Choose **Invoke remotely**, enable **Remote debugging**, and set **Local Root Path** to this repo root.
5. Set breakpoints in `bases/filescience/valkey_queue_processor/handler.py`.
6. Invoke with a realistic DynamoDB stream payload.
7. Use VS Code Run and Debug panel for call stack/variables/step controls.
8. End session with disconnect or **Remove Debug Setup** (debug layer/tunnel are auto-cleaned).

Notes:
- Remote debug temporarily adds a debug layer (~40 MB) and increases timeout (up to 900s) during the session.
- Lambda still has 5-layer and 250 MB combined package+layers limits.
- `us-east-1` is in the supported regions list.

## Workflow B — Local Debug with SAM (Emulated Runtime)
1. Create a minimal SAM template for the queue-processor handler.
2. Build with `sam build --use-container`.
3. Run local invoke in debug mode:
   - `sam local invoke <LogicalId> -e events/dynamodb-stream.json -d 5678`
4. Attach VS Code debugger to port `5678` with path mapping to `/var/task`.
5. Step through handler code with breakpoints.

Useful flags for this repo:
- `--env-vars` for Lambda env parity
- `--docker-network` if local dependencies are containerized
- `--container-host host.docker.internal` on macOS container-host edge cases

## Repo-Specific Observations
- Existing `.vscode/launch.json` + `.vscode/tasks.json` already use `debugpy` for local discover flows, so a similar debug profile can be added for Lambda-focused local runs.
- `docs/lambda-deployment.md` covers build/deploy and log-tail well, but not breakpoint debugging yet.

## Suggested Next Actions
1. Validate AWS Toolkit permissions/profile and run one remote debug session on dev.
2. Save one known-failing DynamoDB stream event as a reusable debug payload fixture.
3. If local-only breakpoint loops are desired, add a small SAM template and VS Code attach config.
4. Optionally extend `docs/lambda-deployment.md` with a "Debugging with breakpoints" section.

## Sources
- AWS Lambda Developer Guide: Remotely debug Lambda functions with VS Code  
  https://docs.aws.amazon.com/lambda/latest/dg/debugging.html
- AWS Toolkit for VS Code User Guide: Lambda remote debugging  
  https://docs.aws.amazon.com/toolkit-for-vscode/latest/userguide/lambda-remote-debug.html
- AWS SAM Developer Guide: Locally debug functions with AWS SAM  
  https://docs.aws.amazon.com/serverless-application-model/latest/developerguide/serverless-sam-cli-using-debugging.html
- AWS SAM CLI Command Reference: `sam local invoke`  
  https://docs.aws.amazon.com/serverless-application-model/latest/developerguide/sam-cli-command-reference-sam-local-invoke.html

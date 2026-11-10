# **Claude Power BI Integration via TrueFoundry**

## **1\. Executive Summary & Objective**

Connecting Claude agents and developer tools to Microsoft Power BI while using TrueFoundry on AWS as your central proxy requires bridging two separate control loops strictly inside your AWS perimeter: **LLM inference routing** (sending all prompts to Amazon Bedrock via TrueFoundry) and **tool execution routing** (connecting your client to Power BI via the Model Context Protocol, or MCP).

To guarantee **zero prompt data or Power BI payloads ever traverse Anthropic’s commercial SaaS cloud** (claude.ai), this architecture prohibits Anthropic-hosted Web/Cowork SaaS sessions and claude.ai OAuth callbacks. Instead, users interact exclusively through **Bedrock-routed clients**—such as **Claude Code**, an **Internal Enterprise Agent UI** (e.g., LibreChat or OpenWebUI deployed on EKS), or custom **Agent SDKs**—configured to route 100% of LLM inference and MCP tool execution through TrueFoundry’s **LLM Gateway** and **MCP Gateway** inside your AWS VPC.

## **2\. Architecture & Integration Options**

### **2.1 Power BI Connector Options**

Your Bedrock-backed Claude clients use the **Model Context Protocol (MCP)** to interact with Power BI. You have three enterprise-compliant architectural options depending on your Power BI licensing and governance posture (third-party SaaS MCP bridges are excluded to preserve strict AWS VPC data residency):

| Option | Target Use Case | How It Works | Governance & Auth Model |
| :---- | :---- | :---- | :---- |
| **1\. TrueFoundry MCP Gateway \+ Microsoft Fabric Remote MCP** | **Native Cloud Analytics** (Recommended if on Fabric F2+ / Premium P1+) | Connects TrueFoundry's MCP Gateway directly to Microsoft's managed Power BI MCP endpoint (api.fabric.microsoft.com) over NAT/ExpressRoute. | Proxied via TrueFoundry using Entra ID On-Behalf-Of (OBO) flow; respects Power BI Row-Level Security (RLS). |
| **2\. TrueFoundry MCP Gateway \+ Custom EKS Power BI MCP** | **Enterprise Cloud Production** (Required for Power BI Pro or strict row-truncation) | Deploy a containerized FastMCP Power BI REST/XMLA server on AWS EKS behind TrueFoundry’s MCP Gateway. | Centralized OAuth2 / Microsoft Entra ID OBO flow; enforces RLS plus Python-level row truncation and PII masking. |
| **3\. Local Power BI Modeling MCP Server** | **Report Authoring & DAX Design** | Runs Microsoft's official powerbi-modeling-mcp.exe locally alongside Power BI Desktop (.pbip / TMDL files). | Local stdio execution on the analyst's machine for local file edits; LLM inference still routes to AWS Bedrock. |

### **2.2 End-to-End System Architecture (100% AWS Bedrock \+ TrueFoundry \+ Power BI)**

To keep all LLM reasoning and tool governance within your AWS boundary, every client is configured to use your internal TrueFoundry AI Gateway for both model tokens and MCP tool calls.

**System Data Flow:**

> 1. **User Identity & Client Layer:** An analyst opens a Bedrock-compliant client (**Claude Code**, an internal EKS-hosted Agent Workspace, or an IDE agent) over corporate VPN / AWS Direct Connect and authenticates via Microsoft Entra ID using a **Public Client App Registration** (tfy-ai-gateway-client-public). On Entra-joined workstations, an OS-native SSO broker (WAM on Windows / Keychain on macOS) acquires and refreshes short-lived v2.0 Entra JWTs for both LLM inference (via apiKeyHelper) and MCP tool calls.  
> 2. **Ingress Layer (AWS VPC):** Requests hit an internal AWS Application Load Balancer (ALB) secured by AWS WAF and TLS termination (with idle timeout configured to 300 seconds for streaming SSE/HTTP), routing into your Amazon EKS cluster running the **TrueFoundry AI Gateway**.  
> 3. **LLM Inference Loop (100% AWS Bedrock):** When the user asks a question (*"Summarize Q3 variance in the Executive Sales dashboard"*), the client sends the prompt to ANTHROPIC\_BASE\_URL pointing to the TrueFoundry **LLM Gateway**. TrueFoundry validates the user's Entra JWT, enforces rate limits/PII guardrails, assumes an AWS IAM Role via IRSA (IAM Roles for Service Accounts), and invokes **Claude on Amazon Bedrock** via an AWS PrivateLink VPC Endpoint (com.amazonaws.\<region\>.bedrock-runtime). Zero traffic leaves AWS for Anthropic's cloud.  
> 4. **Tool Discovery & Execution Loop:** Claude on Bedrock recognizes it needs Power BI data and emits a tool call (Get Semantic Model Schema, Execute Query). The request routes through the **TrueFoundry MCP Gateway**.  
> 5. **Identity Propagation (Entra ID Two-App OBO):** TrueFoundry's MCP Gateway (acting as the **Confidential Client App**, tfy-ai-gateway-powerbi-obo) exchanges the user's incoming Entra JWT for a downstream Power BI access token using the **On-Behalf-Of (OBO)** flow via an outbound AWS NAT Gateway to login.microsoftonline.com. This guarantees Power BI queries run under the *human user's* identity, preserving Power BI Row-Level Security (RLS) without exposing any Client Secret to user machines.  
> 6. **Power BI Execution:** The Power BI MCP Server (either Microsoft's remote endpoint or your EKS FastMCP pod) calls the Power BI API, returns the DAX result to TrueFoundry (which enforces hard payload size limits and logs the audit trail), and passes the context back to **Amazon Bedrock** to generate the final answer.

## **3\. Implementation & Deployment Guide**

### **3.1 Provision the AWS & TrueFoundry Compute Plane**

> 1. **Deploy EKS, NAT Gateway & TrueFoundry Agent:** Provision an Amazon EKS cluster across private subnets in your AWS VPC. Configure an AWS NAT Gateway (or egress firewall) to allow outbound HTTPS (443) to login.microsoftonline.com, api.powerbi.com, and api.fabric.microsoft.com. Install the TrueFoundry compute plane Helm chart and enable Karpenter for node autoscaling.  
> 2. **Configure Internal ALB for MCP Streaming:** Configure your AWS ALB Ingress with an **Idle Timeout of 300 seconds** (to prevent premature disconnects during complex DAX queries and LLM streaming) and ensure custom mcp-\* headers (specifically mcp-Session-Id) are preserved. If running stateful multi-replica MCP pods on EKS, enable target group session stickiness.  
> 3. **Configure Bedrock PrivateLink:** Create Interface VPC Endpoints for com.amazonaws.\<region\>.bedrock-runtime and com.amazonaws.\<region\>.bedrock so model prompts and Power BI results never traverse the public internet.  
> 4. **Bind IAM Roles (IRSA):** Create an IAM Role with bedrock:InvokeModel, bedrock:InvokeModelWithResponseStream, bedrock:ApplyGuardrail, and KMS decryption permissions (if using customer-managed KMS keys for Guardrails) for your approved Claude models and Cross-Region Inference Profiles. Bind this role to the TrueFoundry AI Gateway Kubernetes Service Account via OIDC.

### **3.2 Configure Microsoft Entra ID & Power BI Tenant Settings**

> 1. **Register Two Entra ID Applications (Strict Public/Confidential Separation):**  
   * **App 1 (Confidential Gateway Resource): tfy-ai-gateway-powerbi-obo** — Holds the Client Secret/Certificate in AWS Secrets Manager, exposes the api://\<TFY\_GATEWAY\_APP\_ID\>/access\_as\_user scope, and holds delegated Power BI permissions for the OBO exchange.  
   * **App 2 (Public Desktop/CLI Client): tfy-ai-gateway-client-public** — Used by end-user workstations (Claude Code / local MSAL proxy) with PKCE/WAM broker and **zero Client Secrets**. Authorized to request tokens for App 1's scope.  
> 2. **Assign Delegated Permissions on App 1:** Under **API Permissions** on tfy-ai-gateway-powerbi-obo, add **Power BI Service** delegated permissions: Dataset.Read.All, Report.Read.All, and Workspace.Read.All. Grant Admin Consent.  
> 3. **Enable Power BI Tenant MCP Setting:** If using Microsoft's native remote MCP endpoint, open the Microsoft Fabric / Power BI Admin Portal under **Tenant settings** and enable *Users can use the Power BI Model Context Protocol server endpoint*. *(Note: Do **not** enable "Allow service principals to use Power BI APIs" unless needed for unrelated automation; OBO uses delegated user tokens, and keeping service principal access disabled prevents app-only abuse)*.

### **3.3 Deploy and Register the Power BI MCP Server in TrueFoundry**

> 1. **Choose Your Upstream MCP Target:** Either point directly to Microsoft's remote endpoint (\[https\://api.fabric.microsoft.com/v1/mcp/powerbi\](https\://api.fabric.microsoft.com/v1/mcp/powerbi)) or deploy a custom Python FastMCP container inside your TrueFoundry EKS workspace.  
> 2. **Register in MCP Gateway:** In the TrueFoundry UI, navigate to **AI Gateway \-\> MCP Servers** and register the Power BI MCP server.  
> 3. **Configure OAuth / OBO Authentication:** Select **OAuth2 (microsoft\_entra)** with grant type **jwt\_bearer**. Input the Client ID and Client Secret (or private\_key\_jwt certificate assertion if supported by your TrueFoundry version) of tfy-ai-gateway-powerbi-obo stored in AWS Secrets Manager.  
> 4. **Set RBAC & Guardrails:** Map your corporate SSO groups to the MCP server under **Collaborators**. Enable strict tool allowlisting (tool\_policy) to block Generate Query (preventing Microsoft from routing sub-prompts to Azure OpenAI), and configure gateway-level rate limits and payload truncation.

### **3.4 Connect Bedrock-Routed Clients (Claude Code & Internal Agent UI)**

> 1. **Configure Internal Agent UI (Web Users):** Deploy your internal chat UI (e.g., LibreChat / OpenWebUI / TrueFoundry Chat UI) on EKS, configured with SSO against Entra ID and pointing its model provider to TrueFoundry's LLM Gateway (Bedrock) and tool provider to TrueFoundry's MCP Gateway.  
> 2. **Configure Claude Code / IDE Agents (Power Users):** Configure user workstations via MDM to set ANTHROPIC\_BASE\_URL to your internal TrueFoundry LLM Gateway URL, configure apiKeyHelper to dynamically fetch fresh Entra v2.0 tokens from the OS keychain, and attach the local async MSAL token bridge (tfy\_entra\_mcp\_bridge.py) so both LLM inference and MCP tool calls authenticate seamlessly via Entra ID and route exclusively through AWS.

### **3.5 Optional: Configure Local Authoring for BI Developers**

If BI developers need Claude Code (running on AWS Bedrock) to write and validate DAX measures directly against unsaved local .pbip models in Power BI Desktop before publishing to the cloud:

> 1. Enable **Power BI Project (.pbip)** and **TMDL format** under *File \-\> Options \-\> Preview features* in Power BI Desktop.  
> 2. Install the official **Power BI Modeling MCP Server** extension from the VS Code Marketplace and locate powerbi-modeling-mcp.exe.  
> 3. Register the local binary in the user's .claude.json / MCP configuration with "args": \["--start", "--read-only"\] (omitting \--read-only only when automated local schema edits are desired).

## **4\. Security & Administrative Governance**

> * **Strict Prohibition of Anthropic SaaS Routing:** Standard consumer/commercial Claude Web (claude.ai) and unproxied Claude Desktop GUI sessions route prompts to Anthropic’s commercial cloud and require registering \[https\://claude.ai/api/mcp/auth\_callback\](https\://claude.ai/api/mcp/auth\_callback). To guarantee 100% AWS Bedrock residency, **do not register claude.ai redirect URIs** in your Entra App Registrations, and restrict client access to internal Bedrock-routed interfaces (Claude Code configured with ANTHROPIC\_BASE\_URL or your EKS-hosted Internal Agent UI).  
> * **Enforce Row-Level Security (RLS) via OBO:** Never configure the Power BI MCP server in production with a static master Service Principal (client\_credentials). Always use TrueFoundry's **On-Behalf-Of (jwt\_bearer)** flow so Power BI natively enforces the exact dataset and RLS permissions assigned to the logged-in human user.  
> * **Zero Client Secret Distribution & Unified Short-Lived JWTs:** Never distribute the confidential OBO Client Secret to end-user devices, and never hardcode static Entra JWTs in shell profiles (which expire after 60 minutes). End-user devices must authenticate using the separate Public Client App ID (tfy-ai-gateway-client-public) via PKCE / OS-native SSO broker (WAM/Keychain), sharing the encrypted token cache across both Claude Code's apiKeyHelper (LLM Gateway) and the MCP proxy.  
> * **Hard Payload Truncation & Token Governance:** Prompt instructions (TOPN(200, ...)) in tool\_metadata are soft guidance that LLMs can bypass or omit during complex debugging, and DAX queries against poorly indexed tables can return up to 100,000 rows (15 MB JSON) that rapidly exhaust Bedrock context windows. Enforce **hard byte-size / token truncation limits** on tool responses in TrueFoundry’s AI Gateway (or deploy Option 2's custom FastMCP wrapper to truncate rows at 50 KB in Python before returning) alongside per-team token budgets.  
> * **Fabric Capacity Isolation & Concurrency Throttling:** Autonomous agents frequently execute 3–6 sequential or parallel DAX queries per user turn, which can trigger interactive throttling on shared Microsoft Fabric F-SKU capacities and degrade executive dashboard performance. Isolate AI/MCP workloads onto a dedicated Fabric capacity (or enable Fabric Surge Protection) and enforce a strict per-user MCP rate limit (e.g., 10 tool calls/minute) in TrueFoundry.

## **5\. Infrastructure Setup: AWS IAM & Terraform Configuration**

To connect TrueFoundry’s AI Gateway on Amazon EKS to Amazon Bedrock over AWS PrivateLink using IAM Roles for Service Accounts (IRSA), three core infrastructure components are required:

> 1. **Least-Privilege IAM Policy & IRSA Role:** Scoped strictly to Anthropic Claude foundation models, system and application cross-region inference profiles, multi-region Bedrock Guardrails, and optional KMS customer-managed keys, bound to TrueFoundry's Kubernetes Service Account via OIDC.  
> 2. **AWS PrivateLink (bedrock-runtime & bedrock):** Interface VPC Endpoints with Private DNS enabled so all SDK/Gateway calls to Bedrock resolve to private ENI IP addresses inside your VPC.  
> 3. **VPC Endpoint Policy:** A network-level perimeter policy attached to the PrivateLink endpoints restricting use to your AWS Account and approved Bedrock actions.

**AWS Bedrock Cross-Region Inference Note:** Claude models on Bedrock (such as Claude 3.5 Sonnet v2, Claude 3.7 Sonnet, and Claude 4.x profiles) use **Cross-Region Inference Profiles** (us.anthropic.\* or eu.anthropic.\*). When Bedrock routes an inference request across regions inside a geographic load-balancing group, both foundation-model/\* and guardrail/\* evaluations occur in the destination inference region—meaning the IAM policy **must** use \* for the region field on Foundation Models, Inference Profiles, and Guardrails. Furthermore, system-defined inference profiles must include both account-scoped and accountless (::) ARN patterns to ensure compatibility across all AWS SDK invocation paths.

**IAM Permissions Policy (truefoundry-bedrock-invoke-policy.json):**
```
{  
  "Version": "2012-10-17",  
  "Statement": [  
    {  
      "Sid": "AllowClaudeFoundationModelsAndInferenceProfiles",  
      "Effect": "Allow",  
      "Action": [  
        "bedrock:InvokeModel",  
        "bedrock:InvokeModelWithResponseStream"  
      ],  
      "Resource": [  
        "arn:aws:bedrock:*::foundation-model/anthropic.claude-*",  
        "arn:aws:bedrock:*::inference-profile/*.anthropic.claude-*",  
        "arn:aws:bedrock:*:*:inference-profile/*.anthropic.claude-*",  
        "arn:aws:bedrock:*:*:application-inference-profile/*"  
      ]  
    },  
    {  
      "Sid": "AllowBedrockGuardrailsExecutionMultiRegion",  
      "Effect": "Allow",  
      "Action": [  
        "bedrock:ApplyGuardrail"  
      ],  
      "Resource": "arn:aws:bedrock:*:*:guardrail/*"  
    },  
    {  
      "Sid": "AllowBedrockModelDiscoveryForTrueFoundryGateway",  
      "Effect": "Allow",  
      "Action": [  
        "bedrock:ListFoundationModels",  
        "bedrock:GetFoundationModel",  
        "bedrock:ListInferenceProfiles",  
        "bedrock:GetInferenceProfile"  
      ],  
      "Resource": "*"  
    }  
  ]  
}
```

**IRSA Trust Policy (truefoundry-irsa-trust-policy.json):**
```
{  
  "Version": "2012-10-17",  
  "Statement": [  
    {  
      "Effect": "Allow",  
      "Principal": {  
        "Federated": "arn:aws:iam::<AWS_ACCOUNT_ID>:oidc-provider/oidc.eks.<AWS_REGION>.amazonaws.com/id/<OIDC_ID>"  
      },  
      "Action": "sts:AssumeRoleWithWebIdentity",  
      "Condition": {  
        "StringEquals": {  
          "oidc.eks.<AWS_REGION>.amazonaws.com/id/<OIDC_ID>:aud": "sts.amazonaws.com",  
          "oidc.eks.<AWS_REGION>.amazonaws.com/id/<OIDC_ID>:sub": "system:serviceaccount:truefoundry:tfy-llm-gateway"  
        }  
      }  
    }  
  ]  
}
```
### **5.1 Terraform Blueprint Snippets**

#### **variables.tf**
```
variable "aws_region" {  
  description = "Primary AWS region where EKS and TrueFoundry are deployed"  
  type        = string  
  default     = "us-east-1"  
}

variable "vpc_id" {  
  description = "VPC ID where the EKS cluster resides"  
  type        = string  
}

variable "private_subnet_ids" {  
  description = "List of private subnet IDs for the Bedrock VPC Interface Endpoints"  
  type        = list(string)  
}

variable "eks_cluster_name" {  
  description = "Name of the existing EKS cluster running TrueFoundry"  
  type        = string  
}

variable "tfy_namespace" {  
  description = "Kubernetes namespace where TrueFoundry AI Gateway runs"  
  type        = string  
  default     = "truefoundry"  
}

variable "tfy_service_account_name" {  
  description = "Kubernetes ServiceAccount used by TrueFoundry LLM Gateway pods"  
  type        = string  
  default     = "tfy-llm-gateway"  
}

variable "guardrail_kms_key_arns" {  
  description = "Optional list of Customer Managed KMS Key ARNs used to encrypt Bedrock Guardrails"  
  type        = list(string)  
  default     = []  
}
```

#### **main.tf**
```
terraform {  
  required_version = ">= 1.5.0"  
  required_providers {  
    aws = {  
      source  = "hashicorp/aws"  
      version = "~> 5.0"  
    }  
  }  
}

provider "aws" {  
  region = var.aws_region  
}

# ------------------------------------------------------------------------------  
# 1. Data Sources (Account, VPC, EKS Cluster, & OIDC Provider)  
# ------------------------------------------------------------------------------  
data "aws_caller_identity" "current" {}

data "aws_vpc" "eks_vpc" {  
  id = var.vpc_id  
}

data "aws_eks_cluster" "this" {  
  name = var.eks_cluster_name  
}

locals {  
  oidc_issuer_url   = replace(data.aws_eks_cluster.this.identity[0].oidc[0].issuer, "https://", "")  
  oidc_provider_arn = "arn:aws:iam::$data.aw{s}_{c}alle{r}_{i}dentity.current.accoun{t}_{i}d:oidc-provider/${local.oidc_issuer_url}"  
}

# ------------------------------------------------------------------------------  
# 2. Security Group for Bedrock PrivateLink Endpoints  
# ------------------------------------------------------------------------------  
resource "aws_security_group" "bedrock_vpce_sg" {  
  name        = "${var.eks_cluster_name}-bedrock-vpce-sg"  
  description = "Allow TLS 443 inbound from EKS VPC to Bedrock PrivateLink Endpoints"  
  vpc_id      = var.vpc_id

  ingress {  
    description = "HTTPS from EKS VPC CIDR"  
    from_port   = 443  
    to_port     = 443  
    protocol    = "tcp"  
    cidr_blocks = [data.aws_vpc.eks_vpc.cidr_block]  
  }

  egress {  
    description = "Allow all outbound"  
    from_port   = 0  
    to_port     = 0  
    protocol    = "-1"  
    cidr_blocks = ["0.0.0.0/0"]  
  }

  tags = {  
    Name = "${var.eks_cluster_name}-bedrock-vpce-sg"  
  }  
}

# ------------------------------------------------------------------------------  
# 3. IAM Role & Policy for TrueFoundry Gateway (IRSA)  
# ------------------------------------------------------------------------------  
data "aws_iam_policy_document" "tfy_irsa_assume_role" {  
  statement {  
    effect  = "Allow"  
    actions = ["sts:AssumeRoleWithWebIdentity"]

    principals {  
      type        = "Federated"  
      identifiers = [local.oidc_provider_arn]  
    }

    condition {  
      test     = "StringEquals"  
      variable = "${local.oidc_issuer_url}:aud"  
      values   = ["sts.amazonaws.com"]  
    }

    condition {  
      test     = "StringEquals"  
      variable = "${local.oidc_issuer_url}:sub"  
      values   = ["system:serviceaccount:$var.tf{y}_{n}amespace:${var.tfy_service_account_name}"]  
    }  
  }  
}

resource "aws_iam_role" "tfy_bedrock_irsa_role" {  
  name               = "${var.eks_cluster_name}-tfy-bedrock-irsa"  
  assume_role_policy = data.aws_iam_policy_document.tfy_irsa_assume_role.json

  tags = {  
    Component = "TrueFoundry-LLM-Gateway"  
  }  
}

data "aws_iam_policy_document" "tfy_bedrock_permissions" {  
  statement {  
    sid    = "AllowClaudeModelAndProfileInvocation"  
    effect = "Allow"  
    actions = [  
      "bedrock:InvokeModel",  
      "bedrock:InvokeModelWithResponseStream"  
    ]  
    resources = [  
      "arn:aws:bedrock:*::foundation-model/anthropic.claude-*",  
      "arn:aws:bedrock:*::inference-profile/*.anthropic.claude-*",  
      "arn:aws:bedrock:*:${data.aws_caller_identity.current.account_id}:inference-profile/*.anthropic.claude-*",  
      "arn:aws:bedrock:*:${data.aws_caller_identity.current.account_id}:application-inference-profile/*"  
    ]  
  }

  statement {  
    sid    = "AllowBedrockGuardrailsMultiRegion"  
    effect = "Allow"  
    actions = [  
      "bedrock:ApplyGuardrail"  
    ]  
    # Wildcard region (*) required so Cross-Region Inference Profiles can evaluate guardrails in peer regions  
    resources = [  
      "arn:aws:bedrock:*:${data.aws_caller_identity.current.account_id}:guardrail/*"  
    ]  
  }

  dynamic "statement" {  
    for_each = length(var.guardrail_kms_key_arns) > 0 ? [1] : []  
    content {  
      sid    = "AllowKMSForCustomerManagedGuardrails"  
      effect = "Allow"  
      actions = [  
        "kms:Decrypt",  
        "kms:GenerateDataKey"  
      ]  
      resources = var.guardrail_kms_key_arns  
    }  
  }

  statement {  
    sid    = "AllowBedrockModelDiscovery"  
    effect = "Allow"  
    actions = [  
      "bedrock:ListFoundationModels",  
      "bedrock:GetFoundationModel",  
      "bedrock:ListInferenceProfiles",  
      "bedrock:GetInferenceProfile"  
    ]  
    resources = ["*"]  
  }  
}

resource "aws_iam_policy" "tfy_bedrock_policy" {  
  name        = "${var.eks_cluster_name}-tfy-bedrock-policy"  
  description = "Least-privilege Bedrock policy for TrueFoundry AI Gateway"  
  policy      = data.aws_iam_policy_document.tfy_bedrock_permissions.json  
}

resource "aws_iam_role_policy_attachment" "tfy_bedrock_attach" {  
  role       = aws_iam_role.tfy_bedrock_irsa_role.name  
  policy_arn = aws_iam_policy.tfy_bedrock_policy.arn  
}

# ------------------------------------------------------------------------------  
# 4. AWS PrivateLink Interface Endpoints (bedrock-runtime & bedrock)  
# ------------------------------------------------------------------------------  
data "aws_iam_policy_document" "bedrock_vpce_policy" {  
  statement {  
    sid    = "RestrictToCurrentAWSAccount"  
    effect = "Allow"  
    actions = [  
      "bedrock:InvokeModel",  
      "bedrock:InvokeModelWithResponseStream",  
      "bedrock:ApplyGuardrail",  
      "bedrock:ListFoundationModels",  
      "bedrock:GetFoundationModel",  
      "bedrock:ListInferenceProfiles",  
      "bedrock:GetInferenceProfile"  
    ]  
    resources = ["*"]

    principals {  
      type        = "AWS"  
      identifiers = ["*"]  
    }

    condition {  
      test     = "StringEquals"  
      variable = "aws:PrincipalAccount"  
      values   = [data.aws_caller_identity.current.account_id]  
    }  
  }  
}

resource "aws_vpc_endpoint" "bedrock_runtime" {  
  vpc_id              = var.vpc_id  
  service_name        = "com.amazonaws.${var.aws_region}.bedrock-runtime"  
  vpc_endpoint_type   = "Interface"  
  subnet_ids          = var.private_subnet_ids  
  security_group_ids  = [aws_security_group.bedrock_vpce_sg.id]  
  private_dns_enabled = true  
  policy              = data.aws_iam_policy_document.bedrock_vpce_policy.json

  tags = {  
    Name = "${var.eks_cluster_name}-bedrock-runtime-vpce"  
  }  
}

resource "aws_vpc_endpoint" "bedrock_control_plane" {  
  vpc_id              = var.vpc_id  
  service_name        = "com.amazonaws.${var.aws_region}.bedrock"  
  vpc_endpoint_type   = "Interface"  
  subnet_ids          = var.private_subnet_ids  
  security_group_ids  = [aws_security_group.bedrock_vpce_sg.id]  
  private_dns_enabled = true  
  policy              = data.aws_iam_policy_document.bedrock_vpce_policy.json

  tags = {  
    Name = "${var.eks_cluster_name}-bedrock-control-vpce"  
  }  
}

# ------------------------------------------------------------------------------  
# 5. Outputs  
# ------------------------------------------------------------------------------  
output "truefoundry_bedrock_irsa_role_arn" {  
  description = "Annotate the TrueFoundry Gateway K8s ServiceAccount with this IAM Role ARN"  
  value       = aws_iam_role.tfy_bedrock_irsa_role.arn  
}

output "bedrock_runtime_vpce_id" {  
  description = "VPC Endpoint ID for Bedrock Runtime"  
  value       = aws_vpc_endpoint.bedrock_runtime.id  
}
```

### **5.2 IRSA Role Binding & TrueFoundry Configuration**

> 1. **Annotate the TrueFoundry Service Account:** Attach the truefoundry\_bedrock\_irsa\_role\_arn output to the Gateway Service Account:  
>    kubectl annotate serviceaccount tfy-llm-gateway \  
>      -n truefoundry \  
>      eks.amazonaws.com/role-arn=arn:aws:iam::<AWS_ACCOUNT_ID>:role/<CLUSTER_NAME>-tfy-bedrock-irsa \  
>      --overwrite

>    Restart the TrueFoundry AI Gateway pods so the EKS Pod Identity Webhook injects AWS\_ROLE\_ARN and AWS\_WEB\_IDENTITY\_TOKEN\_FILE.  
> 2. **Configure Provider Integration in TrueFoundry UI:**  
   * Go to **TrueFoundry UI \-\> AI Gateway \-\> Providers \-\> Add Provider \-\> AWS Bedrock**.  
   * For **Authentication Method**, choose **IAM Role / Assumed Service Account (IRSA)**.  
   * Add your target Claude Cross-Region Inference Profile IDs (e.g., us.anthropic.claude-3-7-sonnet-20250219-v1:0).  
> 3. **Verify PrivateLink Resolution & STS Identity from Inside the Pod:**  
>    # 1. Verify DNS resolves to private VPC CIDR IPs (e.g., 10.x.x.x)  
>    nslookup bedrock-runtime.us-east-1.amazonaws.com

>    # 2. Verify the pod assumed the IRSA role via WebIdentity  
>    aws sts get-caller-identity

## **6\. Authentication Architecture: Microsoft Entra ID Two-App OBO Flow**

To adhere to OAuth 2.0 security best practices and avoid exposing confidential client secrets on user devices, the On-Behalf-Of (jwt\_bearer) architecture separates the **Confidential Gateway App** from the **Public Desktop/CLI Client App**:

> * **App 1 (tfy-ai-gateway-powerbi-obo — Confidential Resource & OBO Executor):** Validates inbound tokens at the TrueFoundry Gateway and uses its private credential in AWS Secrets Manager to exchange the user's token for a **Power BI Service** token (aud: \[https\://analysis.windows.net/powerbi/api\](https\://analysis.windows.net/powerbi/api)).  
> * **App 2 (tfy-ai-gateway-client-public — Public Native Client):** Installed on user workstations for PKCE / OS-broker authentication with zero secrets.

### **6.1 Step 1: Register App 1 — TrueFoundry Confidential OBO App**

> 1. **Register the Application:**  
   * In **Microsoft Entra admin center**, navigate to **Identity \-\> Applications \-\> App registrations \-\> New registration**.  
   * **Name:** tfy-ai-gateway-powerbi-obo  
   * **Supported account types:** *Accounts in this organizational directory only (Single tenant)*.  
   * **Redirect URI:** Leave blank (or configure **Web** redirect URI only if linking an internal EKS-hosted Web UI). **Never** add claude.ai redirect URIs.  
   * Click **Register** and copy the **Application (client) ID** (\<TFY\_GATEWAY\_APP\_ID\>) and **Directory (tenant) ID** (\<TENANT\_ID\>).  
> 2. **Enable v2.0 Access Tokens in the App Manifest (Critical):**  
   * Click **Manifest** (under Manage) and set requestedAccessTokenVersion to 2 inside the api object (or "accessTokenAcceptedVersion": 2 if using the legacy Azure AD Graph app manifest format):  
     "api": {  
       "requestedAccessTokenVersion": 2  
     }

   * Click **Save**.  
> 3. **Expose the Inbound API Scope:**  
   * Go to **Expose an API** \-\> next to **Application ID URI**, click **Add** and save api://\<TFY\_GATEWAY\_APP\_ID\>.  
   * Click **Add a scope**:  
     * **Scope name:** access\_as\_user  
     * **Who can consent:** *Admins and users*  
     * **Admin/User consent display name:** Access TrueFoundry AI Gateway  
     * **State:** *Enabled*  
> 4. **Grant Delegated Power BI Service Permissions (For OBO Exchange):**  
   * Go to **API permissions \-\> Add a permission \-\> Microsoft APIs \-\> Power BI Service \-\> Delegated permissions**:  
     * Dataset.Read.All  
     * Workspace.Read.All  
     * Report.Read.All  
   * Click **Add permissions**, then click **Grant admin consent for \[Your Tenant\]**.  
> 5. **Create the Confidential Client Credential (Stored Strictly in AWS Secrets Manager):**  
   * Under **Certificates & secrets**, generate a **Client Secret** (with automated rotation via AWS Secrets Manager) or upload an **X.509 Certificate** (if your TrueFoundry version supports private\_key\_jwt assertions). Store this credential exclusively in TrueFoundry Secrets (backed by AWS Secrets Manager). Never distribute this secret to end users.

### **6.2 Step 2: Register App 2 — Public Desktop/CLI Client App**

> 1. **Register the Public Client Application:**  
   * Navigate to **App registrations \-\> New registration**.  
   * **Name:** tfy-ai-gateway-client-public  
   * **Redirect URI:** Select **Public client/native (mobile & desktop)** and add:  
     * http\://localhost:3334/oauth/callback  
     * http\://localhost (used by MSAL Python InteractiveBrowserCredential)  
     * \[https\://login.microsoftonline.com/common/oauth2/nativeclient\](https\://login.microsoftonline.com/common/oauth2/nativeclient)  
     * ms-appx-web://microsoft.aad.brokerplugin/\<PUBLIC\_CLIENT\_APP\_ID\> (used by Windows Web Account Manager / WAM broker for silent OS-level SSO)  
   * Copy the **Application (client) ID** (\<PUBLIC\_CLIENT\_APP\_ID\>).  
> 2. **Authorize Public Client on App 1:**  
   * Return to App 1 (tfy-ai-gateway-powerbi-obo) \-\> **Expose an API** \-\> **Authorized client applications** \-\> **Add a client application**.  
   * Enter \<PUBLIC\_CLIENT\_APP\_ID\> (and optionally Azure CLI 04b07795-8ddb-461a-bbee-02f9e1bf7b46 for CLI testing) and check the api://\<TFY\_GATEWAY\_APP\_ID\>/access\_as\_user scope.  
   * In App 1's **Manifest**, also add \<PUBLIC\_CLIENT\_APP\_ID\> to the knownClientApplications array so Entra bundles consent smoothly.

### **6.3 Step 3: Inbound Identity Provider Registration in TrueFoundry**

In the TrueFoundry UI, navigate to **Settings \-\> Security & Access \-\> Identity Providers** and apply the configuration to validate tokens issued for App 1:
```
name: entra-obo-inbound  
type: jwt  
issuer: https://login.microsoftonline.com/<TENANT_ID>/v2.0  
jwks_uri: https://login.microsoftonline.com/<TENANT_ID>/discovery/v2.0/keys  
audiences:  
  - <TFY_GATEWAY_APP_ID>  
  - api://<TFY_GATEWAY_APP_ID>  
subject_claim: preferred_username
```

Map resolved identities or Entra group claims to a TrueFoundry **Team** or **Virtual Account**.

### **6.4 Step 4: Runtime Handshake Verification & Testing**

Verify the end-to-end OBO exchange from your terminal using the Azure CLI before rolling out client configurations:

# 1. Request an inbound user token scoped to the TrueFoundry Gateway App  
```
INBOUND_TOKEN=$(az account get-access-token \  
  --scope "api://<TFY_GATEWAY_APP_ID>/access_as_user" \  
  --query accessToken -o tsv)
```

# 2. Call the TrueFoundry MCP Gateway tools/list endpoint over your private network  
```
curl -X POST "https://ai-gateway.internal.yourcompany.com/mcp/powerbi-microsoft-hosted" \  
  -H "Authorization: Bearer $INBOUND_TOKEN" \  
  -H "Content-Type: application/json" \  
  -d '{  
    "jsonrpc": "2.0",  
    "id": 1,  
    "method": "tools/list"  
  }'
```

## **7\. Client Configuration: 100% Bedrock-Routed Clients**

To ensure **zero data leaks to Anthropic's commercial cloud**, all user interactions must use clients where **both** the LLM Inference endpoint (ANTHROPIC\_BASE\_URL) and the MCP Server endpoint route to your internal TrueFoundry AI Gateway on AWS.

### **7.1 Method 1: Claude Code & IDE Agents (100% AWS VPC / Bedrock)**

Configure user machines (via MDM / system profile) to route all LLM inference to AWS Bedrock via TrueFoundry and authenticate both LLM and MCP calls using the Public Client App (\<PUBLIC\_CLIENT\_APP\_ID\>).

#### **Step A: Unified Async MSAL Bridge & Dynamic Token Helper (tfy\_entra\_mcp\_bridge.py)**

Because Entra ID access tokens expire after 60–90 minutes, setting a static ANTHROPIC\_AUTH\_TOKEN environment variable in a shell profile will cause LLM inference calls to fail after an hour. Instead, deploy a single unified Python script (tfy\_entra\_mcp\_bridge.py) that serves two roles:

> 1. **\--print-token mode (called by Claude Code's apiKeyHelper):** Pre-warms the encrypted OS keychain cache (using Windows WAM / macOS Keychain broker where available) and outputs a fresh Entra JWT for Bedrock LLM Gateway calls.  
> 2. **Default MCP proxy mode:** Runs the FastMCP streamable HTTP proxy using async\_auth\_flow reading from the exact same encrypted OS keychain cache, preventing headless stdio startup timeouts.

**tfy\_entra\_mcp\_bridge.py:**
```
import asyncio  
import sys  
import httpx  
from azure.identity import InteractiveBrowserCredential, TokenCachePersistenceOptions  
from fastmcp import FastMCP  
from fastmcp.client import Client  
from fastmcp.client.transports import StreamableHttpTransport

TENANT_ID = "<TENANT_ID>"  
PUBLIC_CLIENT_ID = "<PUBLIC_CLIENT_APP_ID>"  
GATEWAY_APP_ID = "<TFY_GATEWAY_APP_ID>"  
SCOPE = f"api://{GATEWAY_APP_ID}/access_as_user"  
TFY_MCP_URL = "https://ai-gateway.internal.yourcompany.com/mcp/powerbi-microsoft-hosted"

# Persist MSAL token cache in encrypted OS keychain (DPAPI on Windows / Keychain on macOS)  
cache_options = TokenCachePersistenceOptions(  
    enabled=True,  
    name="tfy_powerbi_mcp_token_cache",  
    allow_unencrypted_storage=False  
)

def build_credential():  
    # Prefer OS-native SSO broker (WAM on Windows) if azure-identity-broker is installed  
    try:  
        from azure.identity.broker import InteractiveBrowserBrokerCredential  
        return InteractiveBrowserBrokerCredential(  
            tenant_id=TENANT_ID,  
            client_id=PUBLIC_CLIENT_ID,  
            cache_persistence_options=cache_options,  
            use_default_broker_account=True  
        )  
    except ImportError:  
        return InteractiveBrowserCredential(  
            tenant_id=TENANT_ID,  
            client_id=PUBLIC_CLIENT_ID,  
            cache_persistence_options=cache_options  
        )

credential = build_credential()

class AsyncEntraBearerAuth(httpx.Auth):  
    async def async_auth_flow(self, request: httpx.Request):  
        # Run synchronous MSAL token acquisition/cache lookup in a worker thread to avoid blocking asyncio  
        token_obj = await asyncio.to_thread(credential.get_token, SCOPE)  
        request.headers["Authorization"] = f"Bearer {token_obj.token}"  
        yield request

def main():  
    # Mode 1: Dynamic token helper for Claude Code apiKeyHelper & pre-warming auth  
    if "--print-token" in sys.argv:  
        token_obj = credential.get_token(SCOPE)  
        sys.stdout.write(token_obj.token)  
        sys.stdout.flush()  
        return

    # Mode 2: Non-blocking MCP stdio-to-StreamableHTTP proxy  
    transport = StreamableHttpTransport(url=TFY_MCP_URL, auth=AsyncEntraBearerAuth())  
    proxy_client = Client(transport)  
    mcp_proxy = FastMCP.as_proxy(proxy_client, name="PowerBI-TrueFoundry-Gateway")  
    mcp_proxy.run(transport="stdio")

if __name__ == "__main__":  
    main()
```

#### **Step B: Route All LLM Inference to AWS Bedrock via \~/.claude/settings.json**

Instead of hardcoding a static ANTHROPIC\_AUTH\_TOKEN in the shell, configure \~/.claude/settings.json with apiKeyHelper so Claude Code dynamically acquires and refreshes the user's Entra JWT before invoking TrueFoundry's LLM Gateway:

```
{  
  "apiKeyHelper": "uv run --with fastmcp --with azure-identity --with azure-identity-broker --with msal-extensions --with httpx C:\\ProgramData\\CompanyAI\\tfy_entra_mcp_bridge.py --print-token",  
  "env": {  
    "ANTHROPIC_BASE_URL": "https://ai-gateway.internal.yourcompany.com/anthropic",  
    "ANTHROPIC_MODEL": "bedrock-claude-3-7-sonnet"  
  }  
}
```

#### **Step C: Register the MCP Server in .claude.json (Claude Code MCP Configuration)**

```
{  
  "mcpServers": {  
    "powerbi-truefoundry": {  
      "command": "uv",  
      "args": [  
        "run",  
        "--with", "fastmcp",  
        "--with", "azure-identity",  
        "--with", "azure-identity-broker",  
        "--with", "msal-extensions",  
        "--with", "httpx",  
        "C:\\ProgramData\\CompanyAI\\tfy_entra_mcp_bridge.py"  
      ]  
    }  
  }  
}
```

#### **Step D: Alternative Zero-Code Setup Using mcp-remote (Public Client PKCE)**

If you prefer using npx mcp-remote instead of the Python MSAL script for MCP transport, pass the **Public Client ID** (\<PUBLIC\_CLIENT\_APP\_ID\>) with no client secret:

```
{  
  "mcpServers": {  
    "powerbi-truefoundry": {  
      "command": "npx",  
      "args": [  
        "-y",  
        "mcp-remote",  
        "https://ai-gateway.internal.yourcompany.com/mcp/powerbi-microsoft-hosted",  
        "--static-oauth-client-info",  
        "{\"client_id\":\"<PUBLIC_CLIENT_APP_ID>\",\"scope\":\"api://<TFY_GATEWAY_APP_ID>/access_as_user offline_access\"}"  
      ]  
    }  
  }  
}
```

### **7.2 Method 2: Internal Enterprise Web UI on EKS (For Non-Technical Analysts)**

Because Anthropic's hosted Web UI (claude.ai) routes prompts through Anthropic's cloud, provide business analysts with a browser-based conversational experience by deploying an **Internal Enterprise Agent UI** (TrueFoundry Playground, OpenWebUI, or LibreChat) inside your EKS cluster:

> 1. Configure the Web UI to authenticate users via OIDC against Microsoft Entra ID (tfy-ai-gateway-powerbi-obo), requesting the api://\<TFY\_GATEWAY\_APP\_ID\>/access\_as\_user scope.  
> 2. Configure the Web UI backend to forward the user's Entra JWT in the Authorization: Bearer header to both TrueFoundry’s **LLM Gateway** (which routes inference to AWS Bedrock over PrivateLink) and TrueFoundry’s **MCP Gateway** (which executes the OBO exchange to Power BI).

## **8\. Verification & Troubleshooting Guidelines**

> * **Error AADSTS50013: Assertion failed signature validation:** Your Entra Gateway App (tfy-ai-gateway-powerbi-obo) is still issuing v1.0 tokens. Verify "requestedAccessTokenVersion": 2 is set in the Entra App **Manifest**, as v1.0 tokens fail signature validation when exchanged against the v2.0 /oauth2/v2.0/token endpoint.  
> * **Error AADSTS65001: The user or administrator has not consented to use the application:** Open the Entra Admin Center \-\> tfy-ai-gateway-powerbi-obo \-\> **API permissions**, and click **Grant admin consent for \[Organization\]**. Also verify \<PUBLIC\_CLIENT\_APP\_ID\> is listed under **Expose an API \-\> Authorized client applications**.  
> * **Error AADSTS7000218: The request body must contain client\_assertion or client\_secret:** You attempted to authenticate a public desktop client directly against the Confidential App ID (\<TFY\_GATEWAY\_APP\_ID\>). Ensure desktop clients authenticate using \<PUBLIC\_CLIENT\_APP\_ID\> while requesting the scope api://\<TFY\_GATEWAY\_APP\_ID\>/access\_as\_user.  
> * **Cross-Region Guardrail AccessDeniedException:** Verify the IAM policy on your IRSA role uses \* for the region in arn:aws:bedrock:\*:\<ACCOUNT\_ID\>:guardrail/\*, includes both account-scoped and accountless (arn:aws:bedrock:\*::inference-profile/\*) inference profile ARNs, and permits kms:Decrypt if your Guardrail uses a Customer Managed KMS Key.  
> * **MCP Server Connection Timeout on Cold Start:** If powerbi-truefoundry times out during IDE startup because an interactive login window opened in the background, run tfy\_entra\_mcp\_bridge.py \--print-token once from the terminal (or ensure apiKeyHelper is configured in \~/.claude/settings.json) so the OS keychain token cache is pre-warmed prior to stdio initialization.

## **9\. Microservice Strategy & Custom FastMCP Evaluation**

Microsoft provides official **Remote Hosted Power BI MCP Servers** out of the box. Because TrueFoundry’s MCP Gateway can proxy external https\:// MCP endpoints directly, you can register Microsoft’s native endpoints in TrueFoundry and skip building a custom Python microservice on EKS unless specific platform constraints apply.

### **9.1 Native Hosted Microsoft Endpoints Overview**

Microsoft hosts remote MCP endpoints that speak the MCP Streamable HTTP protocol and accept the Entra ID On-Behalf-Of (jwt\_bearer) token (\[https\://analysis.windows.net/powerbi/api/.default\](https\://analysis.windows.net/powerbi/api/.default)):

> 1. **Power BI Consumption MCP Server (\[https\://api.fabric.microsoft.com/v1/mcp/powerbi\](https\://api.fabric.microsoft.com/v1/mcp/powerbi)):** Exposes four native tools (Get Semantic Model Schema, Get Report Metadata, Generate Query, and Execute Query) with full RLS enforcement.  
> 2. **Fabric IQ MCP Server (\[https\://api.fabric.microsoft.com/v1/mcp/fabriciq\](https\://api.fabric.microsoft.com/v1/mcp/fabriciq)):** Conversational analytics MCP endpoint across governed Fabric workspaces.  
> 3. **Power BI Authoring MCP Server (\[https\://api.fabric.microsoft.com/v1/mcp/powerbi/authoring\](https\://api.fabric.microsoft.com/v1/mcp/powerbi/authoring)):** Allows BI developers to create and refactor tables, DAX measures, and relationships in the cloud.

### **9.2 Custom Python FastMCP Decision Drivers**

Evaluate whether to use Microsoft's remote endpoint or deploy a custom FastMCP server on EKS based on these four technical drivers:

| Reason to Build Custom FastMCP | Microsoft Native Hosted MCP Limitation | How a Custom Python FastMCP Server Solves It |
| :---- | :---- | :---- |
| **1\. Licensing Constraint (Power BI Pro vs. Fabric F-SKU)** | Microsoft's hosted MCP endpoints require paid **Microsoft Fabric capacity (F2+)** or **Power BI Premium (P1+)**, plus enabling preview tenant switches. | A custom Python server calls standard Power BI REST APIs (ExecuteQueries), which works on basic **Power BI Pro** licenses without Fabric capacity. |
| **2\. "Generate Query" Loops Back to Azure OpenAI** | On Microsoft's hosted endpoint, the Generate Query tool uses Microsoft's internal Azure OpenAI Copilot service to write DAX. | Either block Generate Query via TrueFoundry's tool\_policy (Section 10.2) or use a custom FastMCP server so **Claude on AWS Bedrock** writes 100% of the DAX. |
| **3\. Hard Context Window & Row Truncation Guardrails** | An ungrouped DAX EVALUATE query against a large table can return up to 100,000 rows (15 MB JSON), exhausting your Bedrock token budget if the LLM ignores soft prompt instructions. | A custom Python FastMCP server enforces hard AST/regex TOPN(500, ...) injection or truncates responses at 50 KB \+ masks sensitive PII columns in Python *before* payloads reach Bedrock. |
| **4\. Power BI Report Server (On-Premises / AWS EC2)** | Microsoft's hosted MCP endpoints only work with the Power BI Cloud Service. | A custom FastMCP server on EKS can query on-premises **Power BI Report Server** or SSAS over private AWS Direct Connect. |

## **10\. Endpoint Tooling & Manifest Configuration**

To proxy Microsoft’s hosted Power BI MCP endpoint (\[https\://api.fabric.microsoft.com/v1/mcp/powerbi\](https\://api.fabric.microsoft.com/v1/mcp/powerbi)) through TrueFoundry while keeping **100% of DAX generation and LLM reasoning inside AWS Bedrock**, you must block Microsoft's Generate Query tool using TrueFoundry’s tool\_policy allowlist.

### **10.1 Tool Governance Matrix**

| Tool Name | What It Does | Action for Strict AWS Bedrock Compliance |
| :---- | :---- | :---- |
| **Get Semantic Model Schema** | Returns tables, columns, DAX measures, relationships, and hierarchies. | **ENABLE (Allowlist)** — Safe, read-only metadata discovery. |
| **Get Report Metadata** | Returns report pages, visuals, slicers, and field bindings. | **ENABLE (Allowlist)** — Helps Claude on Bedrock understand existing visual calculations. |
| **Execute Query** | Executes a DAX query against the semantic model under the signed-in user's RLS permissions. | **ENABLE \+ OVERRIDE DESCRIPTION \+ GATEWAY PAYLOAD CAP** — Instruct Claude via tool\_metadata to wrap queries in TOPN(200, ...) and enforce hard response truncation \+ rate limits at the gateway. |
| **Generate Query** | Calls Microsoft's internal Copilot service (Azure OpenAI) to translate natural language into DAX. | **DISABLE (Block)** — Prevents sub-prompts from leaving AWS for Azure OpenAI; forces **Claude on AWS Bedrock** to author DAX directly. |

### **10.2 Production TrueFoundry MCP Server Manifest Specification**

Save the following manifest as powerbi-mcp-server.yaml and apply it via tfy apply \-f powerbi-mcp-server.yaml:

```
name: powerbi-microsoft-hosted  
type: mcp-server-remote  
description: "Microsoft Hosted Power BI Consumption MCP Server proxied via TrueFoundry OBO"  
url: https://api.fabric.microsoft.com/v1/mcp/powerbi

# 1. Entra ID JWT-Bearer On-Behalf-Of (OBO) Authentication  
auth_data:  
  type: oauth2  
  provider: microsoft_entra  
  grant_type: jwt_bearer  
  token_url: https://login.microsoftonline.com/<TENANT_ID>/oauth2/v2.0/token  
  client_id: <TFY_GATEWAY_APP_ID>  
  # Reference your rotated Entra Client Secret (or certificate assertion if supported) in AWS Secrets Manager  
  client_secret: tfy-secret://your-org:aws-secrets-store:entra-obo-client-secret  
  scopes:  
    - https://analysis.windows.net/powerbi/api/.default

# 2. Strict Tool Allowlist Policy (Blocks 'Generate Query' so zero prompts go to Azure Copilot)  
tool_policy:  
  enable_tools_by_default: false  
  enabled_tools:  
    - "Get Semantic Model Schema"  
    - "Get Report Metadata"  
    - "Execute Query"

# 3. Tool Metadata & Guardrail Overrides (Injects TOPN limits into Bedrock Claude's tool definition)  
# Note: Pair this soft prompt instruction with hard response size truncation and per-user rate limits  
# in TrueFoundry AI Gateway Policies to prevent ungrouped DAX queries from exhausting Bedrock context.  
tool_metadata:  
  "Get Semantic Model Schema":  
    description: "Fetch tables, columns, measures, and relationships for a Power BI semantic model. Always call this before writing a DAX query."  
    annotations:  
      readOnlyHint: true  
      destructiveHint: false  
  "Get Report Metadata":  
    description: "Fetch pages, visuals, and field bindings for a Power BI report to understand how metrics are displayed."  
    annotations:  
      readOnlyHint: true  
      destructiveHint: false  
  "Execute Query":  
    description: >  
      Execute a read-only DAX query against a Power BI semantic model.  
      CRITICAL GUARDRAIL: Never execute an ungrouped EVALUATE on a raw table.  
      Always aggregate using SUMMARIZECOLUMNS/ROW or cap results using TOPN(200, ...)  
      to avoid exceeding context window limits.  
    annotations:  
      readOnlyHint: true  
      destructiveHint: false

# 4. RBAC Access Control  
collaborators:  
  - subject: team:bi-analysts  
    role: mcp-user  
  - subject: team:platform-admins  
    role: mcp-admin
```

*(Note: After connecting the server in the TrueFoundry UI and clicking **Sync / Test Connection**, inspect the **Tools** tab to verify exact casing/spacing of tool names returned by your endpoint version and match enabled\_tools accordingly)*.

### **10.3 Authoring Control & Approval Workflows**

If you also connect Microsoft's **Power BI Authoring MCP Server** (\[https\://api.fabric.microsoft.com/v1/mcp/powerbi/authoring\](https\://api.fabric.microsoft.com/v1/mcp/powerbi/authoring)) for BI developers:

> 1. **Block Destructive Authoring Tools Automatically:** Use disabled\_tools inside tool\_policy to hide delete\_table, delete\_measure, and delete\_relationship.  
> 2. **Require Human-in-the-Loop Approval for Schema Edits:** In **AI Gateway \-\> Policies \-\> MCP Tool Approval**, select your Power BI Authoring MCP server and set **Approval Scope** to destructive (or specific write tools) so schema changes require approval in Microsoft Teams or Slack before executing.

## **11\. Final Deployment Verification & Operations Checklist**

### **11.1 End-to-End Readiness Checklist**

| Layer | Verification Item | Expected Outcome |
| :---- | :---- | :---- |
| **1\. AWS Networking & VPC** | Verify bedrock-runtime PrivateLink DNS resolution inside EKS pods (nslookup) and NAT Gateway egress to login.microsoftonline.com & api.fabric.microsoft.com. | bedrock-runtime resolves to private VPC CIDR IPs (10.x.x.x); outbound HTTPS to Entra/Fabric succeeds. |
| **2\. AWS ALB Ingress** | Check ALB timeout and header preservation settings. | ALB Idle Timeout is \>= 300s; mcp-Session-Id header is preserved on requests and responses. |
| **3\. AWS IAM & IRSA** | Run aws sts get-caller-identity inside the tfy-llm-gateway pod and test a Cross-Region Inference Profile with a Guardrail attached. | Pod assumes \<cluster\>-tfy-bedrock-irsa; cross-region inference and guardrail (including KMS decryption) succeed without AccessDeniedException. |
| **4\. Microsoft Entra ID** | Verify App 1 (tfy-ai-gateway-powerbi-obo) has v2.0 tokens enabled, Admin Consent granted for Power BI delegated scopes, and App 2 (tfy-ai-gateway-client-public) listed under Authorized Client Applications. | az account get-access-token issues a v2.0 JWT (ver: "2.0"); zero claude.ai redirect URIs exist on either app. |
| **5\. TrueFoundry Gateways** | Test tools/list via curl with an inbound Entra token, verify synced tools, and check Gateway rate-limit/payload-truncation rules. | OBO exchange succeeds; only Get Semantic Model Schema, Get Report Metadata, and Execute Query are exposed (Generate Query is blocked); responses \>50 KB are truncated. |
| **6\. Client Bedrock Enforcement** | Verify ANTHROPIC\_BASE\_URL and apiKeyHelper are set in \~/.claude/settings.json and inspect network traffic during a Power BI prompt. | 100% of LLM inference hits TrueFoundry \-\> AWS Bedrock PrivateLink using auto-refreshed Entra JWTs; zero packets route to api.anthropic.com or claude.ai. |

### **11.2 Key Pre-Flight Reminders**

> * **Stateful Session Header (mcp-Session-Id):** Microsoft's hosted Power BI MCP endpoints maintain semantic model connection state across calls using the mcp-Session-Id HTTP header. Ensure your TrueFoundry Gateway ingress/ALB does not strip custom mcp-\* headers on requests or responses.  
> * **Private Link Endpoint for Fabric IQ:** If your organization enforces Azure Private Links on Microsoft Fabric and exposes the Fabric IQ MCP server, point TrueFoundry to \[https\://api.fabric.microsoft.com/v1/mcp/fabriciq\](https\://api.fabric.microsoft.com/v1/mcp/fabriciq) rather than fabriciq.svc.cloud.microsoft.

---


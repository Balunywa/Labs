
# End-to-End Architecture: Sales RFP Agent with Power Automate Integration

This document outlines two deployment patterns for integrating email-driven RFP processing into the Sales RFP Agent solution, highlighting how data moves from a customer email to the Copilot Studio agent, and detailing the security controls in each scenario.

## Overview
The Sales RFP Agent automates intake and processing of Request for Proposal (RFP) documents sent by customers, enabling sales teams to quickly respond with generated proposals inside Microsoft Teams. The solution uses Outlook, Microsoft Graph, Power Automate, SharePoint, Dataverse, and Copilot Studio.

Two architectural scenarios are considered:

- **Scenario 1 — Azure Front Door (AFD)/WAF with Private Origin**: Maximum control over ingress traffic with Azure Front Door Premium, Web Application Firewall, and a Private Link–secured backend.

- **Scenario 2 — Direct M365 Integration (Non-AFD)**: Simplified architecture using only M365 native triggers in Power Automate.

## Common Components
## Common Components

Both scenarios use the same downstream processing and outputs:

- **Power Automate** — orchestrates the workflow from email to generated proposal.
- **SharePoint** — stores raw RFPs from customers and generated proposals from the agent.
- **Dataverse** — stores structured knowledge source files and metadata for Copilot.
- **Copilot Studio** — hosts the Sales RFP Agent in Microsoft Teams.
- **Connectors** — enable Power Automate to interact with SharePoint, Dataverse, and Teams.

## Scenario 1 — AFD/WAF + Private Origin (Recommended)

### Flow

1. **Customer sends RFP email**
   - Delivered to an Exchange Online mailbox monitored for new messages.

2. **Microsoft Graph change notification**
   - Subscription configured to `/messages` resource on the mailbox.
   - `notificationUrl` set to AFD public endpoint (e.g., `https://rfp-ingest.contosoafdedgenet/api/webhook`).

3. **Azure Front Door + WAF**
   - Terminates TLS.
   - Applies WAF rules: geo/IP restrictions, bot protection, rate limiting.
   - Forwards validated requests to Private Endpoint.

4. **Function App / Logic App (Private Origin)**
   - Private Link–secured; no public IP.
   - On subscription validation: echoes `validationToken` within 10s.
   - On notification: uses Managed Identity to request Graph access token from Entra ID, then calls `GET /messages/{id}` and `/attachments` to pull full content.

5. **Trigger Power Automate Flow:**
   - **Option A (HTTP Trigger)** — NAT Gateway egress IP allowlisted in Flow; HMAC signature in headers validated by Flow before processing.
   - **Option B (Private Trigger)** — enqueue message to Service Bus or update Dataverse row; Flow triggered privately, authenticated with MI + RBAC.

6. **Power Automate processing**
   - Saves raw RFP to SharePoint (private site, Sites.Selected permission).
   - Updates Dataverse with structured knowledge data.
   - Publishes proposal to Copilot Agent in Teams channel.

### Security Considerations for Scenario 1

| Area | Control |
|------|---------|
| **Ingress** | Azure Front Door Premium with WAF policy (geo/IP filtering, bot protection, rate limiting). |
| **Origin** | Function App/Logic App private endpoint (Private Link). |
| **Auth to Graph** | Managed Identity or App Registration with Mail.Read (application permission), admin consent. |
| **Flow Trigger Security** | HTTP Trigger — NAT static IP allowlist, HMAC signature validation. Private Trigger — MI with RBAC on Service Bus/Dataverse. |
| **Data at Rest** | SharePoint private site with Sites.Selected, Dataverse security roles. |
| **Connector Auth** | Service Account in Entra ID for SharePoint/Teams connectors. |
| **Monitoring** | Entra sign-in logs, M365 Unified Audit Log, AFD access logs, Function App AppInsights. |

## Scenario 2 — Direct M365 Integration (Non-AFD)

### Flow

1. **Customer sends RFP email**
   - Delivered to Exchange Online mailbox.

2. **Power Automate Outlook trigger**
   - Cloud Flow uses the built-in Outlook connector trigger "When a new email arrives".
   - Trigger runs entirely within M365; no public endpoints required.

3. **Power Automate processing**
   - Extracts RFP email and attachments.
   - Saves raw RFP to SharePoint (private site).
   - Updates Dataverse with knowledge data.
   - Publishes generated proposal to Copilot Agent in Teams channel.

### Security Considerations for Scenario 2

| Area | Control |
|------|---------|
| **Ingress** | All within Microsoft 365; no Azure public ingress points. |
| **Auth to mailbox** | Outlook connector in Flow runs under a Service Account with delegated mailbox access. |
| **Connector Auth** | Service Account in Entra ID for SharePoint/Teams/Dataverse connectors. |
| **Data at Rest** | SharePoint private site with Sites.Selected, Dataverse security roles. |
| **Monitoring** | M365 Unified Audit Log, Flow run history, SharePoint audit logs. |

## Scenario Comparison

| Aspect | Scenario 1 — AFD/WAF | Scenario 2 — Direct M365 |
|--------|----------------------|---------------------------|
| **Public ingress** | Yes, via AFD public endpoint (WAF-protected) | None |
| **Control over ingress** | High — WAF rules, Private Link | N/A — M365 internal |
| **Processing trigger** | Graph webhook → Function App | Outlook connector |
| **Flow trigger security** | NAT IP allowlist + HMAC or private MI trigger | N/A — internal connector |
| **Flexibility** | High — can integrate with any external system | Limited to M365 services |
| **Complexity** | Higher — Azure + M365 integration | Lower — all in M365 |

## Best Practice Recommendations

- **For sensitive data or external-facing mailbox ingestion** → Use Scenario 1 for maximum ingress control and visibility.
- **For internal M365-only workflows with low external exposure risk** → Use Scenario 2 for simplicity.
- Always bind Flow connectors to a dedicated Service Account in Entra ID, never personal accounts.
- For any Azure→M365 call, prefer Managed Identity over storing secrets.
- Monitor all activity in Entra, M365 Unified Audit Log, and App Insights.
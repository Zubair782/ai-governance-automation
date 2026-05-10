# DORA Incident Response Agent

Automated major incident classification and notification workflow for financial entities under the EU Digital Operational Resilience Act (DORA).

## 🎯 Purpose

This n8n workflow automates the end-to-end response process for cybersecurity incidents in financial institutions:

1. **Ingestion** — Receives incident data from SIEM (e.g., Microsoft Sentinel)
2. **Context enrichment** — Looks up affected services in CMDB, calculates economic impact
3. **Classification** — AI agent evaluates the incident against DORA Article 18 criteria (7 thresholds)
4. **Notification draft** — Generates compliant initial notification (Article 19 §1) in French
5. **Record keeping** — Logs to incident registry (Google Sheets)
6. **Alert** — Sends email to compliance team with 4-hour deadline reminder

## 📋 Regulatory Framework

- **DORA** (Regulation EU 2022/2554) — Digital Operational Resilience Act
- **Commission Delegated Regulation (EU) 2024/1772** — RTS on major incident classification
- **Commission Implementing Regulation (EU) 2024/2956** — ITS on incident reporting format
- Target authorities: ACPR / Banque de France (France), EBA, ESMA, EIOPA

## 🏗️ Architecture

```
SIEM Alert → Mock Incident Data → Context Enrichment (CMDB lookup)
                                         ↓
                                  AI Classification Agent
                                  (Claude Sonnet 4.5)
                                         ↓
                              ┌──────────┴──────────┐
                              ↓                     ↓
                        Major Incident      Non-Major Incident
                              ↓                     ↓
                   Notification Drafting    (End - no action)
                   (AI Agent - French)
                              ↓
                    ┌─────────┴─────────┐
                    ↓                   ↓
            Google Sheets Registry   Gmail Alert
```

## 🔧 Prerequisites

### Required n8n Version
- n8n v1.0+ (tested on v1.x)
- Self-hosted or n8n Cloud

### Required Credentials
Configure these in n8n before importing:

1. **Anthropic API** (Claude)
   - Get key at: https://console.anthropic.com/
   - Add as "Anthropic" credential in n8n
   - Model used: `claude-sonnet-4-5-20250929`

2. **Google Sheets API**
   - OAuth2 or Service Account
   - Permissions: Read/Write on target spreadsheet
   - Create a Google Sheet with columns matching the workflow schema (see below)

3. **Gmail API** (or SMTP)
   - OAuth2 authentication
   - Required scope: `https://www.googleapis.com/auth/gmail.send`

### Google Sheet Structure

Create a spreadsheet with these columns (exact header names):

| Column Name | Description |
|-------------|-------------|
| `timestamp_processed` | ISO datetime when workflow ran |
| `incident_id` | Internal reference (e.g., INC-2025-0042) |
| `detected_at` | Detection timestamp from SIEM |
| `classification` | major / significant / non_major |
| `service_id` | Affected service ID |
| `service_name` | Affected service name |
| `is_critical_function` | Boolean - DORA Art. 8 critical function |
| `clients_affected` | Number of impacted clients |
| `duration_minutes` | Downtime in minutes |
| `countries` | Comma-separated EU member states |
| `economic_impact_eur` | Calculated financial impact |
| `justification` | Classification rationale (French) |
| `initial_notification_due` | 4-hour deadline (ISO datetime) |
| `intermediate_report_due` | 72-hour deadline |
| `final_report_due` | 30-day deadline |
| `notification_draft` | Full markdown notification text |
| `status` | draft_to_review / submitted / closed |

## 📥 Installation

### 1. Import the workflow

**Method A — From file:**
```bash
# Download the workflow.json from this repo
# In n8n: Workflows → Import from File → Select workflow.json
```

**Method B — From URL:**
```
n8n: Workflows → Import from URL
URL: https://raw.githubusercontent.com/Zubair782/ai-governance-automation/main/workflows/dora-incident-response/workflow.json
```

**Method C — Copy-paste:**
```
Open workflow.json → Copy all (Ctrl+A, Ctrl+C)
In n8n canvas → Click → Paste (Ctrl+V)
```

### 2. Configure credentials

After import, each node with a red ⚠️ warning needs credentials:
- **Anthropic Chat Model** → Select your Anthropic credential
- **Registre Incidents** (Google Sheets node) → Select your Google credential
- **Email Cellule Conformite** (Gmail node) → Select your Gmail credential

### 3. Update placeholders

Replace these values in the workflow:

**Google Sheets node** ("Registre Incidents"):
- `documentId` → Replace `REPLACE_WITH_YOUR_GOOGLE_SHEET_ID` with your actual Sheet ID
  (Found in the URL: `https://docs.google.com/spreadsheets/d/YOUR_SHEET_ID_HERE/edit`)

**Gmail node** ("Email Cellule Conformite"):
- `sendTo` → Replace `REPLACE_WITH_RECIPIENT_EMAIL@example.com` with compliance team email

### 4. Customize CMDB (optional)

Edit the **"Enrichissement contexte Incident"** (Code node) to match your service inventory:

```javascript
const cmdb = {
  "YOUR-SERVICE-ID": {
    name: "Your Service Name",
    criticality: "critical", // or "non_critical"
    is_critical_function: true,
    business_owner: "Your Department",
    rto_minutes: 60
  }
  // Add more services...
};
```

### 5. Test the workflow

1. Click **"Execute workflow"** (manual trigger)
2. Check execution logs — all nodes should turn green ✅
3. Verify outputs:
   - Google Sheet updated with new row
   - Email received by compliance team
4. Review the generated notification for accuracy

## 🎮 Usage

### Manual Trigger (Demo Mode)
- Current setup uses **mock incident data** in the "Mock Incident SIEM" node
- Modify the JSON in this node to test different scenarios

### Production Integration
Replace the "When clicking 'Execute workflow'" trigger with one of:

**Option A — Webhook (recommended):**
```
Delete manual trigger → Add "Webhook" node
Configure POST endpoint
Send SIEM alerts to: https://your-n8n.domain/webhook/dora-incidents
```

**Option B — Schedule polling:**
```
Delete manual trigger → Add "Schedule Trigger"
Set to run every X minutes
Add API call to SIEM to fetch new incidents
```

**Option C — Email trigger:**
```
Delete manual trigger → Add "Email Trigger (IMAP)"
Monitor security@yourdomain inbox
Parse SIEM alert emails
```

## 🔍 Classification Logic

The AI agent evaluates 7 DORA Article 18 criteria:

1. **Clients affected** — Threshold: > 10,000 or > 5% of client base
2. **Reputational impact** — Media coverage, complaints, customer loss
3. **Duration** — Threshold: > 2 hours for critical services
4. **Geographical spread** — Threshold: ≥ 2 EU Member States
5. **Data losses** — Material impact on Confidentiality, Integrity, Availability, Authenticity
6. **Service criticality** — DORA Art. 8 critical/important function (YES/NO)
7. **Economic impact** — Threshold: > €100,000 or > 0.1% annual revenue

**Classification outcome:**
- **MAJOR** → Criterion 6 met + ≥1 other threshold met
- **SIGNIFICANT** → High likelihood of becoming major
- **NON_MAJOR** → Below thresholds

## 📊 Sample Output

**Classification JSON:**
```json
{
  "classification": "major",
  "criteria_assessment": {
    "clients_affected": { "value": 145000, "threshold_met": true },
    "service_criticality": { "is_critical_function": true },
    "economic_impact_eur": { "value": 7280000, "threshold_met": true }
  },
  "justification": "Incident majeur : fonction critique impactée...",
  "deadlines_utc": {
    "initial_notification_due": "2026-05-07T12:15:00Z",
    "intermediate_report_due": "2026-05-10T08:15:00Z",
    "final_report_due": "2026-06-06T08:15:00Z"
  }
}
```

**Notification draft** (excerpt, in French):
```markdown
# Notification initiale d'incident TIC majeur (DORA Art. 19 §1)

## 1. Identification de l'entité déclarante
- Entité : [À COMPLÉTER PAR LA CONFORMITÉ]
- Code LEI : [À COMPLÉTER]

## 2. Identification de l'incident
- Référence interne : INC-2025-0042
- Type d'incident : Accès non autorisé / exfiltration de données
- Date et heure de détection (UTC) : 2026-05-07T08:15:00Z
...
```

## ⚠️ Important Notes

### Data Privacy
- **Mock data only** in this public version
- In production, ensure GDPR compliance for incident data storage
- Google Sheets: set appropriate access controls
- Consider encryption at rest for sensitive incident records

### Limitations
- This is a **proof-of-concept** workflow
- AI classification requires human review before regulatory submission
- Economic impact calculation is simplified (production systems need integration with financial reporting)
- French-language output only (for ACPR/Banque de France); adapt prompts for other jurisdictions

### Production Readiness Checklist
- [ ] Replace mock SIEM data with real integration
- [ ] Validate CMDB accuracy
- [ ] Legal review of notification templates
- [ ] Stress-test with edge cases (borderline incidents)
- [ ] Set up monitoring/alerting for workflow failures
- [ ] Implement approval workflow before actual submission to authorities
- [ ] Archive completed incidents per retention policy

## 🤝 Contributing

Improvements welcome:
- Support for other EU financial regulators (BaFin, DNB, etc.)
- Multi-language notification templates
- Integration examples with other SIEM platforms
- Enhanced economic impact models

## 📄 License

MIT License - See LICENSE file in repo root

## 🔗 Resources

- [DORA Full Text (EUR-Lex)](https://eur-lex.europa.eu/eli/reg/2022/2554/oj)
- [EBA Guidelines on ICT Risk Management](https://www.eba.europa.eu/regulation-and-policy/internal-governance/guidelines-on-ict-and-security-risk-management)
- [n8n Documentation](https://docs.n8n.io/)
- [Anthropic Claude API](https://docs.anthropic.com/)

---

**Author:** Zubair782  
**Status:** Proof of Concept  
**Last Updated:** May 2026

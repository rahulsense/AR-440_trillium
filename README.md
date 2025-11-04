# AR-440: Fixing Repeat Calls from Grace After Screening Completion

## Problem Description

**JIRA Ticket:** [AR-440](https://sensehq.atlassian.net/browse/AR-440)

### Examples of the Issue

**Example 1:** Candidate screened at 9:07 AM, received follow-up attempts at 9:10 AM and 1:08 PM

![Example 1](https://github.com/user-attachments/assets/28bd8bd5-60ef-44ea-86f1-b35525af049b)

**Example 2:** Candidate screened twice on Friday and twice again the following Monday

![Example 2](https://github.com/user-attachments/assets/f24fd563-aeff-4441-9db7-70f6f9425659)

**Example 3:** Candidate screened at 9:12 AM, follow-up attempt at 1:12 PM

![Example 3](https://github.com/user-attachments/assets/50b9e524-c300-4890-aa5b-bef946643c9a)

**Example 4:** Candidate screened at 9:13 AM, repeat call 4 minutes later, then again at 1:15 PM

![Example 4](https://github.com/user-attachments/assets/24dc051b-c67c-4c89-9ed9-e55e530b6417)

**Example 5:** Candidate screened three times in one day

![Example 5](https://github.com/user-attachments/assets/4c8afcf1-6c58-4f49-9bda-2ebeaefcb4b0)

## Root Cause

The issue was traced to an incorrect Note Writeback (NWB) configuration in the agency configs. Calls were not being properly marked as completed in the system, causing the workflow to continue attempting to contact candidates who had already been screened.

## Debugging Process

### Step 1: Access the Database
1. Navigate to Slack and request access to `chatbot-db-cluster-ro`
2. Connect via SDM: `chatbot-db-cluster-ro`
3. Open Sequel Ace and connect to the `trillium` database

### Step 2: Query Call Records
Run the following query to find call IDs for a specific recipient (replace `4336077` with the actual ATS ID):

```sql
SELECT id 
FROM voice_ai_call 
WHERE recipient_ats_id = 4336077 
ORDER BY time_created DESC;
```

### Step 3: Check Call Audit Logs
Run this query to retrieve detailed call logs (started, initiated, failed, voicemail):

```sql
SELECT * 
FROM voice_abt_audit 
WHERE conversation_id IN (
    SELECT id 
    FROM voice_ai_call 
    WHERE recipient_ats_id = 4336077 
    ORDER BY time_created DESC
);
```

### Step 4: Analysis
Upon reviewing the audit logs, we discovered that **none of the calls were marked as completed** in the system, which explained why candidates continued to receive follow-up calls.

![Database Audit Logs](https://github.com/user-attachments/assets/53aa1cb8-6762-454b-870c-7f3d6c4efe20)

### Step 5: Identify Workflow Issue
1. Navigated to the associated workflow: [Automation Workflow #114](https://trilliumstaffing.sensehq.com/automation-workflow/114)
2. Found that Path 1 was using an incorrect NWB action type: **"AI Recruiter Screening"**
3. This was an automated Note Writeback from the Voice AI team that wasn't properly signaling call completion

## Solution

Updated the NWB action type from **"AI Recruiter Screening"** to **"Sense Voice AI"** in the workflow configuration.

![Workflow Configuration Fix](https://github.com/user-attachments/assets/c76cf33e-1916-483c-8f30-967f86f10afb)

This change ensures that completed calls are properly marked in the system, preventing the workflow from re-triggering follow-up attempts for already-screened candidates.

## Status

**Resolved** - Workflow updated and candidates should no longer receive repeat calls after successful screening completion.

---

## Related Resources
- **JIRA Ticket:** [AR-440](https://sensehq.atlassian.net/browse/AR-440)
- **Workflow:** [Automation Workflow #114](https://trilliumstaffing.sensehq.com/automation-workflow/114)
- **Database:** `chatbot-db-cluster-ro` → `trillium`

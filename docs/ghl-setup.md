# GoHighLevel Setup Guide

## Step 1: Enable Call Transcription
Go to **Settings → Phone Numbers** and enable transcription for each number.

## Step 2: Create the Workflow
1. Automation → Workflows → Create New → Start from Scratch
2. Trigger: **Transcript Generated**
3. Filter: Call Duration > 10 seconds, Direction = Inbound

## Step 3: Add Webhook Action
POST to your n8n endpoint with these GHL custom values:
- `{{contact.id}}`
- `{{contact.fullNameLowerCase}}`
- `{{contact.phone}}`
- `{{call.duration}}`
- `{{call.type}}`
- `{{call.transcript}}`
- `{{call.summary}}`
- `{{location.id}}`

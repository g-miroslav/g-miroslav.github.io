---
layout: post
title: How to send approval requests via email that were created in MS Teams
subtitle: It's 2026, and people still expect emails!
cover-img: /assets/img/ms_teams_background.avif
thumbnail-img: /assets/img/ms_teams_approvals_logo.png
share-img: /assets/img/ms_teams_background.avif
gh-repo: username/repo
tags: [Power Automate, Approvals, MS Teams]
comments: true
mathjax: true
author: Miroslav Gencur
---

## Introduction
The Approvals app in MS Teams is the most straighforward tool for tracking of approval requests. It allows users to submit a request to one or several people for approval. Approval templates are used to predefine a list of approvers, approval stages, questions, or requirements for attachments.

<img src="{{ '/assets/img/ms_teams_approvals.jpg' | relative_url }}" alt="MS Teams Approvals app">

## Problem
All approval requests, created in the MS Teams Approvals app, only allow approvers to respond in MS Teams. No emails are sent are sent as a result of approval requests submitted via the Approvals app. Moreover, there are no settings in MS Teams that would give us the option to enable email notifications.

## Solution
Email notifications can be turned on with a simple Power Automate flow that leverages the dataverse connector. This connector is premium, therefore a Power Automate license is required.

<img src="{{ '/assets/img/email_notification_flow.png' | relative_url }}" alt="Email Notification Flow">

### Trigger
The trigger of the flow pickes up all approvals initiated from the Approvals app. This means that approval request started by Power Automate or SharePoint aren't affected.

<img src="{{ '/assets/img/email_notification_trigger.png' | relative_url }}" alt="Email Notification Flow trigger">

### Action
Select the approval from the trigger:

<img src="{{ '/assets/img/email_notification_action_param.png' | relative_url }}" alt="Email Notification Flow approval id parameter">

Change the value of 'Send Email notification' field to 'Yes':

<img src="{{ '/assets/img/email_notification_action_value.png' | relative_url }}" alt="Email Notification Flow approval value">


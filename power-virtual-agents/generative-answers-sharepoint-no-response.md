---
title: "Generative answers don't return a response"
description: "Troubleshoot when generative answers pointing to SharePoint or OneDrive sources don't return results."
author: adilei
ms.date: 03/11/2024
ms.topic: troubleshooting
ms.custom: guidance
ms.author: adileibowitz
ms.reviewer: erickinser
---

# Generative answers pointing to SharePoint or OneDrive sources don't return results

Generative answers allow makers to create copilots that respond to questions grounded in data sources, like public websites or SharePoint, by pointing the copilot at those data sources. However, sometimes the copilot doesn't provide a response and instead returns something like **'I’m not sure how to help with that. Can you try rephrasing?'** (the actual message depends on the implementation).

:::image type="content" source="media/generative-answers/genanswers-no-response.png" alt-text="Screenshot showing no response from Generative Answers.":::

## Why doesn't the 'Create generative answers' node respond?

When a SharePoint or OneDrive data source is configured, there could be several different factors preventing generative answers from returning a response, such as the following potential factors:

1. [Search results are missing](#search-results-are-missing)

1. [The user accessing the copilot doesn't have sufficient permissions on the data source](#missing-user-permissions)

1. [Files are larger than the 3 MB size limitation](#file-size-limitation)

1. [The app registration or the copilot are misconfigured](#the-app-registration-or-copilot-are-misconfigured)

1. [Content blocked by content moderation](#content-blocked-by-content-moderation)

> [!Note]
> Before continuing, please make sure you have followed the instructions on how to [set up generative answers over SharePoint or OneDrive](nlu-boost-node.md).

## Search results are missing

Generative answers for a SharePoint or OneDrive data source rely on making calls to the Graph API search endpoint. Only the top three results coming back from Graph API are used to summarize and generate a response. If no results come back from Graph API, the generative answers node doesn't provide a response.

To diagnose whether Copilot Studio isn't returning results from the Graph API, you can make direct calls to the Graph API search endpoint. This call simulates the way Copilot Studio works behind the scenes. Calls to the Graph API search endpoint can be generated by using the following template with [Graph Explorer](/graph/graph-explorer/graph-explorer-overview). When accessing Graph Explorer, be sure to sign-in using the appropriate credentials for the SharePoint/OneDrive tenant.

The template can be used either by copying the following payload, or using this [deep link](https://developer.microsoft.com/graph/graph-explorer?request=search%2Fquery&method=POST&version=v1.0&GraphUrl=https://graph.microsoft.com&requestBody=eyJyZXF1ZXN0cyI6W3siZW50aXR5VHlwZXMiOlsiZHJpdmVJdGVtIiwibGlzdEl0ZW0iXSwicXVlcnkiOnsicXVlcnlTdHJpbmciOiJTRUFSQ0ggVEVSTVMgZmlsZXR5cGU6ZG9jeCBPUiBmaWxldHlwZTphc3B4IE9SIGZpbGV0eXBlOnBwdHggT1IgZmlsZXR5cGU6cGRmIHBhdGg6XCJET01BSU4uc2hhcmVwb2ludC5jb20vc2l0ZXMvU0lURU5BTUUifSwiZnJvbSI6MCwic2l6ZSI6MywiUXVlcnlBbHRlcmF0aW9uT3B0aW9ucyI6eyJFbmFibGVNb2RpZmljYXRpb24iOnRydWUsIkVuYWJsZVN1Z2dlc3Rpb24iOnRydWV9fV19), which opens Graph Explorer with a prepopulated query.

**POST https://graph.microsoft.com/v1.0/search/query**

```json
{
    "requests": [
        {
            "entityTypes": [
                "driveItem",
                "listItem"
            ],
            "query": {
                "queryString": "SEARCH TERMS filetype:docx OR filetype:aspx OR filetype:pptx OR filetype:pdf path:\"DOMAIN.sharepoint.com/sites/SITENAME"
            },
            "from": 0,
            "size": 3,
            "QueryAlterationOptions": {
                "EnableModification": true,
                "EnableSuggestion": true
            }
        }
    ]
}
```

### Missing results

Let’s assume that generative answers are configured to provide responses based on content stored in https://\<user-domain\>.sharepoint.com/sites/HR. However, users aren't getting responses when asking, "What is our policy regarding perks & benefits?"

Behind the scenes, users’ queries are being rewritten, so only the main keywords are being sent to Graph API, resulting in a query similar to the following example:

:::image type="content" source="media/generative-answers/perks-benefits.png" alt-text="Screenshot of a search query in Graph Explorer.":::

If no results are returned to the search endpoint, as shown in the following response, generative answers doesn't provide a response, either.

:::image type="content" source="media/generative-answers/no-results.png" alt-text="Screenshot showing no results returned from a search in Graph Explorer.":::

### How to fix

1. Ensure that your Create generative answers node points to a SharePoint or OneDrive location with relevant content.

1. Only documents in [supported formats](nlu-boost-node.md#supported-content) are used to generate responses.

    > [!Note]
    > Only modern SharePoint pages are supported.

1. It's possible that documents were only recently uploaded to SharePoint or OneDrive, but have yet to be indexed. It's also possible that there are settings that prevent some sites from appearing in search results. For more information, see [Search results missing in SharePoint Online](/sharepoint/troubleshoot/search/search-results-missing).

## Missing user permissions

Generative answers over SharePoint and OneDrive rely on [delegated permissions](nlu-boost-node.md#authentication) when making calls to Graph API. At a minimum, a user must have read permissions on the relevant sites and files, or the call to Graph API doesn't return any results.

If the user is missing permissions, no results are returned from Graph API, nor any errors or exceptions. For a user with no permissions, it appears as if no documents were found.

### How to fix

Amend permissions so users can access the relevant sites and files. For more information, see [Sharing and permissions in the SharePoint modern experience](/sharepoint/modern-experience-sharing-permissions).

## The app registration or copilot are misconfigured

When admins configure generative answers over SharePoint and OneDrive, admins are expected to set up authentication with a Microsoft Entra ID, and configure [extra scopes](nlu-boost-node.md#authentication). If scopes are missing from the app registration or from the copilot authentication settings, or if consent wasn't granted to the required scopes, no results are returned, nor any errors or exceptions. For an end user, it appears as if no documents were found.

### How to fix

Add the necessary scopes to the App Registration and/or the copilot’s authentication settings, and grant consent.

The following example is a reference to a well configured app registration:

:::image type="content" source="media/generative-answers/app-registration.png" alt-text="Screenshot of app registration permissions.":::

The following example shows the required authentication settings in Copilot Studio:

:::image type="content" source="media/generative-answers/copilot-auth.png" alt-text="Screenshot showing Copilot Studio authentication settings.":::

## File size limitation

Currently, generative answers can only process files up to 3 MB in size. Larger files can be stored in SharePoint and **are returned** by a Graph API search, but aren't processed by generative answers.

> [!Note]
> This limitation doesn't apply to customers eligible for [M365 Semantic Indexing](/microsoftsearch/semantic-index-for-copilot).

### How to fix

If files relevant for your conversational AI experience exceed the 3 MB limitation, you might want to explore alternative architectures, such as using [Microsoft 365 Semantic Indexing](/microsoftsearch/semantic-index-for-copilot) or [connect your data to Azure Open AI for Generative Answers](nlu-generative-answers-azure-openai.md).

## Content blocked by content moderation

When generating responses, Copilot Studio moderates content that's harmful, malicious, noncompliant, or in breach of copyrights. When content gets moderated, generative answers don't provide a response or an indication that content was moderated. However, moderation events are logged when Copilot Studio is configured to [send telemetry data to Azure Applications Insights](advanced-bot-framework-composer-capture-telemetry.md#connect-your-microsoft-copilot-studio-bot-to-application-insights).

After connecting your copilot to Azure App Insights, you can use the following Kusto Query Language (KQL) query to find out if content was filtered:

```
customEvents
| extend cd = todynamic(customDimensions)
| extend conversationId = tostring(cd.conversationId)
| extend topic = tostring(cd.TopicName)
| extend message = tostring(cd.Message)
| extend result = tostring(cd.Result)
| extend SerializedData = tostring(cd.SerializedData)
| extend Summary = tostring(cd.Summary)
| extend feedback = tostring(todynamic(replace_string(SerializedData,"$","")).value)
| where name == "GenerativeAnswers" and result contains "Filtered"
| where cloud_RoleInstance == "myCopilot"
| project cloud_RoleInstance, name, timestamp, conversationId, topic, message, result, feedback, Summary
| order by timestamp desc
```

In the following example, the KQL query highlights an attempt to use generative answers filtered by content moderation:

:::image type="content" source="media/generative-answers/content-filtered.png" alt-text="Screenshot of Azure Application Insights.":::

### How to fix

1. Try to adjust [content moderation](nlu-boost-conversations.md#content-moderation), but keep in mind that lower content moderation settings might result in answers that are less accurate or relevant.

1. If you think your content shouldn't be moderated, [raise a case with customer support](/power-platform/admin/get-help-support).

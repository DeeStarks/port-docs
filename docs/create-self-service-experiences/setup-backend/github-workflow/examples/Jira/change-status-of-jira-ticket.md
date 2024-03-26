# Change status of Jira ticket

## Overview
This self-service guide facilitates transitioning the status of a Jira ticket from Port using Port's self service actions. With this, you can manage ticket (issue) status without leaving Port.

## Prerequisites
* [Jira API token](https://support.atlassian.com/atlassian-account/docs/manage-api-tokens-for-your-atlassian-account/) with permissions to create new issues.
* [Port's GitHub app](https://github.com/apps/getport-io) needs to be installed.

## Steps
1. Create the following GitHub action secrets
* JIRA_API_TOKEN - [Jira API token](https://support.atlassian.com/atlassian-account/docs/manage-api-tokens-for-your-atlassian-account) generated by the user.
* JIRA_BASE_URL - The URL of your Jira organization.
* JIRA_USER_EMAIL - The email of the Jira user that owns the Jira API token.
* PORT_AUTH_CLIENT_ID - Your port [client id]([How to get the credentials](https://docs.getport.io/build-your-software-catalog/sync-data-to-catalog/api/#find-your-port-credentials)).
* PORT_AUTH_CLIENT_SECRET - Your port [client secret]([How to get the credentials](https://docs.getport.io/build-your-software-catalog/sync-data-to-catalog/api/#find-your-port-credentials)).

2. Optional - Install Port's Jira integration [learn more](https://docs.getport.io/build-your-software-catalog/sync-data-to-catalog/project-management/jira#installation)

3. Creating the action in Port
:::tip Action usage

This action is on the `Jira Issue` blueprint.

:::

<details>
<summary><b>Change status of a Jira ticket (Click to expand)</b></summary>

```json showLineNumbers
{
  "identifier": "change_jira_ticket_status",
  "title": "Change Jira ticket status",
  "icon": "Jira",
  "userInputs": {
    "properties": {
      "status": {
        "title": "Status",
        "type": "string",
        "enum": [
          "To Do",
          "In Progress",
          "Code Review",
          "Product Review",
          "Waiting For Prod",
          "Done"
        ],
        "enumColors": {
          "To Do": "lightGray",
          "In Progress": "bronze",
          "Code Review": "darkGray",
          "Product Review": "purple",
          "Waiting For Prod": "orange",
          "Done": "green"
        }
      }
    },
    "required": [
      "status"
    ],
    "order": []
  },
  "invocationMethod": {
    "type": "GITHUB",
    "repo": "<Enter GitHub repository>",
    "org": "<Enter GitHub organization>",
    "workflow": "change_jira_ticket_status.yml",
    "omitUserInputs": false,
    "omitPayload": false,
    "reportWorkflowStatus": true
  },
  "trigger": "DAY-2",
  "description": "Transition a ticket to another status.",
  "requiredApproval": false
}
```

</details>

4. Create a workflow file under `.github/workflows/change_jira_ticket_status.yml` using the workflow:

<details>
<summary><b>Change status of a Jira ticket (Click to expand)</b></summary>

```yaml showLineNumbers

name: Change Jira Ticket Status
on:
  workflow_dispatch:
    inputs:
      status:
        type: string
        required: true
      port_payload:
        required: true
        description:
          Port's payload, including details for who triggered the action and
          general context (blueprint, run id, etc...)
        type: string
    secrets:
      JIRA_BASE_URL:
        required: true
      JIRA_USER_EMAIL:
        required: true
      JIRA_API_TOKEN:
        required: true
      PORT_CLIENT_ID:
        required: true
      PORT_CLIENT_SECRET:
        required: true

jobs:
  create-entity-in-port-and-update-run:
    runs-on: ubuntu-latest
    steps:
      - name: Login
        uses: atlassian/gajira-login@v3
        env:
          JIRA_BASE_URL: ${{ secrets.JIRA_BASE_URL }}
          JIRA_USER_EMAIL: ${{ secrets.JIRA_USER_EMAIL }}
          JIRA_API_TOKEN: ${{ secrets.JIRA_API_TOKEN }}

      - name: Inform starting of changing Jira ticket status
        uses: port-labs/port-github-action@v1
        with:
          clientId: ${{ secrets.PORT_CLIENT_ID }}
          clientSecret: ${{ secrets.PORT_CLIENT_SECRET }}
          operation: PATCH_RUN
          runId: ${{ fromJson(inputs.port_payload).context.runId }}
          logMessage: |
            Changing status of Jira issue... ⛴️

      - name: Transition issue
        uses: atlassian/gajira-transition@v3
        with:
          issue: ${{ fromJson(inputs.port_payload).context.entity }}
          transition: ${{ github.event.inputs.status }}

      - name: Inform that status has been changed
        uses: port-labs/port-github-action@v1
        with:
          clientId: ${{ secrets.PORT_CLIENT_ID }}
          clientSecret: ${{ secrets.PORT_CLIENT_SECRET }}
          operation: PATCH_RUN
          link: ${{ secrets.JIRA_BASE_URL }}/browse/${{ steps.create.outputs.issue }}
          runId: ${{ fromJson(inputs.port_payload).context.runId }}
          logMessage: |
            Jira issue status changed to ${{ github.event.inputs.status }}! ✅
```

</details>

5. Trigger the action from Port's [Self Service hub](https://app.getport.io/self-serve)

6. Done! wait for the ticket's status to be changed in Jira

Congrats 🎉 You've changed a ticket status in Port 🔥
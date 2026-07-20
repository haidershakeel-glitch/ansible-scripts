# n8n AI Agent Workflow - Infrastructure Assistant

## Overview
This workflow creates an AI agent that can interact with infrastructure tools (AAP, NetBox, Passbolt) through a chat interface. Users can send natural language commands to the agent which will execute tasks automatically without needing to log into each system directly.

---

## Step 1: Add Chat Trigger Node

1. Click **"+"** to add a node
2. Search **"Chat"** → select **"On new Chat event"** under Triggers
3. In the node settings:
   - Enable **"Make Chat Publicly Available"** toggle
   - **Mode** — leave as `Hosted Chat`
   - **Authentication** — leave as `None`
   - **Initial Message** — customize as needed (default is "Hi there! My name is Nathan. How can I assist you today?")
   - Leave **"Make Available in n8n Chat Hub"** disabled
4. Note the **Chat URL** at the top — this is the public URL where users can access the chat interface once the workflow is published
5. Click **Save**

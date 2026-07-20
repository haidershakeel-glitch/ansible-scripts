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

---

## Step 2: Add AI Agent Node and Configure Language Model

1. Close the Chat Trigger node
2. Click **"+"** to add a new node → search **"AI Agent"** → select it
3. It will automatically connect to the Chat Trigger node
4. Leave default settings:
   - **Source for Prompt**: Connected Chat Trigger Node
   - **Prompt**: `{{ $json.chatInput }}`
5. Click **Save** and close the node
6. On the canvas, click the **"+"** on the **Chat Model** connector below the AI Agent node
7. Search **"OpenAI Chat Model"** and select it
8. Click **"Set up credential"**:
   - **Name**: `Fuelix OpenAI Key` (or any meaningful name)
   - **API Key**: your Fuelix API key
   - **Base URL**: `https://api.fuelix.ai/v1`
   - Click **Save**
9. In the model settings:
   - Switch model selector to **"By ID"**
   - Enter model name: `claude-sonnet-4-6`
10. Click **Settings** tab → uncheck **"Use Responses API"** — this is required as Fuelix does not support the OpenAI Responses API format
11. Click **Save**

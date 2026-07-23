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

---

## Step 3: Add Memory Node

1. On the canvas, click **"+"** on the **Memory** connector below the AI Agent node
2. Select **"Simple Memory"** under "For beginners" — no credentials required, stores in n8n memory
3. Leave all default settings:
   - **Session ID**: Connected Chat Trigger Node
   - **Context Window Length**: 5 (agent remembers last 5 interactions)
4. Click **Save**

> Note: Simple Memory is for development/demo only. For production use, switch to MongoDB, Postgres, or Redis Chat Memory for persistent storage across restarts.

---

## Step 4: Add NetBox Query Tool

1. On the canvas, click **"+"** on the **Tool** connector below the AI Agent node
2. Search **"HTTP Request"** and select it
3. Configure the node:

   **Description:**
   ```
   Use this tool to query devices from NetBox inventory. Use when the user asks about devices, servers, IP addresses, serial numbers, or any network inventory information.
   ```

   **Method:** `GET`

   **URL:**
   ```
   http://host.docker.internal:8000/api/dcim/devices/
   ```
   > Note: Use `host.docker.internal` instead of `localhost` because n8n runs inside Docker and cannot reach the host machine via `localhost`. When switching to the lab NetBox instance, replace this with the full lab URL.

   **Authentication:** `Generic Credential Type`
   - **Generic Auth Type:** `Header Auth`
   - **Header Auth account:** Create new credential:
     - **Name:** `Authorization`
     - **Value:** `Token your-netbox-api-token`

4. Click **Execute step** to verify data is returned from NetBox
5. Click **Save**

> Note: NetBox API is organized by sections — you need to specify the endpoint. Key endpoints:
> - `/api/dcim/devices/` — physical devices (servers, switches, routers)
> - `/api/ipam/ip-addresses/` — IP addresses
> - `/api/dcim/interfaces/` — device interfaces/ports
> - `/api/ipam/prefixes/` — subnets
>
> Add separate HTTP Request tool nodes for each type of query you want to support.

---

## Step 5: Add IP Address Query Tool

1. Click **"+"** on the **Tool** connector again
2. Search **"HTTP Request"** and select it
3. Configure the node:

   **Description:**
   ```
   Use this tool to query IP addresses in NetBox. Use when the user asks about a specific IP address, wants to find what device an IP belongs to, or asks about subnets and IP assignments.
   ```

   **Method:** `GET`

   **URL:**
   ```
   http://host.docker.internal:8000/api/ipam/ip-addresses/
   ```

   **Authentication:** `Generic Credential Type` → `Header Auth` → same NetBox token as Step 4

4. Click **Execute step** to verify data is returned
5. Click **Save**

**Test:** Ask the agent "What device is [IP address] assigned to?" — the agent will search NetBox IP records and return the device assignment or inform you if the IP is not documented.

---

## Step 6: Add Lookup Tools

These tools allow the agent to look up existing NetBox data before creating new records.

### Get Sites Tool
1. Click **"+"** on the **Tool** connector
2. Search **"HTTP Request"** and select it
3. Configure:
   - **Description:** `Use this tool to get a list of all available sites in NetBox. Use this before creating a device to find the correct site ID, or when a user asks about locations and sites.`
   - **Method:** `GET`
   - **URL:** `http://host.docker.internal:8000/api/dcim/sites/`
   - **Authentication:** Same NetBox token
4. Click **Save**

### Get Device Types Tool
1. Click **"+"** on the **Tool** connector
2. Search **"HTTP Request"** and select it
3. Configure:
   - **Description:** `Use this tool to get a list of all available device types in NetBox. Use this before creating a device to find the correct device type ID based on the manufacturer and model.`
   - **Method:** `GET`
   - **URL:** `http://host.docker.internal:8000/api/dcim/device-types/`
   - **Authentication:** Same NetBox token
4. Click **Save**

### Get Manufacturers Tool
1. Click **"+"** on the **Tool** connector
2. Search **"HTTP Request"** and select it
3. Configure:
   - **Description:** `Use this tool to get a list of all available manufacturers in NetBox. Use this before creating a new device type to find or verify the correct manufacturer ID.`
   - **Method:** `GET`
   - **URL:** `http://host.docker.internal:8000/api/dcim/manufacturers/`
   - **Authentication:** Same NetBox token
4. Click **Save**

---

## Step 7: Add Create Site Tool

1. Click **"+"** on the **Tool** connector
2. Search **"HTTP Request"** and select it
3. Configure:
   - **Description:** `Use this tool to create a new site in NetBox. Use this when a user wants to add a device at a site that does not exist yet. Requires a site name and status.`
   - **Method:** `POST`
   - **URL:** `http://host.docker.internal:8000/api/dcim/sites/`
   - **Authentication:** Same NetBox token
   - **Send Body:** Toggle ON
     - **Body Content Type:** `JSON`
     - **Body:**
     ```json
     {
       "name": "={{ $fromAI('site_name') }}",
       "slug": "={{ $fromAI('site_slug') }}",
       "status": "active"
     }
     ```
4. Click **Save**

> Note: `$fromAI()` fields are filled in dynamically by the agent based on what the user types in chat. Site only requires `name` and `slug` — IP addresses are assigned to devices and interfaces, not sites.

---

## Step 8: Add Create Manufacturer Tool

1. Click **"+"** on the **Tool** connector
2. Search **"HTTP Request"** and select it
3. Configure:
   - **Description:** `Use this tool to create a new manufacturer in NetBox. Use this when a user wants to create a device type but the manufacturer does not exist yet. Requires a manufacturer name and slug.`
   - **Method:** `POST`
   - **URL:** `http://host.docker.internal:8000/api/dcim/manufacturers/`
   - **Authentication:** Same NetBox token
   - **Send Body:** Toggle ON
     - **Body Content Type:** `JSON`
     - **Body:**
     ```json
     {
       "name": "={{ $fromAI('manufacturer_name') }}",
       "slug": "={{ $fromAI('manufacturer_slug') }}"
     }
     ```
4. Click **Save**

> Note: The agent calls this automatically if a user asks to add a device from a manufacturer that doesn't exist yet — the user does not need to create the manufacturer separately.

---

## Step 9: Add Create Device Type Tool

1. Click **"+"** on the **Tool** connector
2. Search **"HTTP Request"** and select it
3. Configure:
   - **Description:** `Use this tool to create a new device type in NetBox. Use this when a user wants to add a device but the model does not exist yet. Requires a manufacturer ID, model name, and slug.`
   - **Method:** `POST`
   - **URL:** `http://host.docker.internal:8000/api/dcim/device-types/`
   - **Authentication:** Same NetBox token
   - **Send Body:** Toggle ON
     - **Body Content Type:** `JSON`
     - **Body:**
     ```json
     {
       "manufacturer": { "id": "={{ $fromAI('manufacturer_id') }}" },
       "model": "={{ $fromAI('model_name') }}",
       "slug": "={{ $fromAI('model_slug') }}"
     }
     ```
4. Click **Save**

> Note: The agent will first call the Get Manufacturers tool to find the correct manufacturer ID, then pass it here when creating the device type.

---

## Step 10: Add Create Interface Tool

1. Click **"+"** on the **Tool** connector
2. Search **"HTTP Request"** and select it
3. Configure:
   - **Description:** `Use this tool to create a network interface on a device in NetBox. Use this after creating a device, before assigning an IP address. Requires a device ID and interface name. Use type "1000base-t" for standard ethernet.`
   - **Method:** `POST`
   - **URL:** `http://host.docker.internal:8000/api/dcim/interfaces/`
   - **Authentication:** Same NetBox token
   - **Send Body:** Toggle ON
     - **Body Content Type:** `JSON`
     - **Body:**
     ```json
     {
       "device": { "id": "={{ $fromAI('device_id') }}" },
       "name": "={{ $fromAI('interface_name') }}",
       "type": "1000base-t"
     }
     ```
4. Click **Save**

---

## Step 11: Add Assign IP Address Tool

1. Click **"+"** on the **Tool** connector
2. Search **"HTTP Request"** and select it
3. Configure:
   - **Description:** `Use this tool to assign an IP address to a device interface in NetBox. Use this after creating the device and interface. Requires the IP address in CIDR format (e.g. 10.0.0.1/24) and the interface ID.`
   - **Method:** `POST`
   - **URL:** `http://host.docker.internal:8000/api/ipam/ip-addresses/`
   - **Authentication:** Same NetBox token
   - **Send Body:** Toggle ON
     - **Body Content Type:** `JSON`
     - **Body:**
     ```json
     {
       "address": "={{ $fromAI('ip_address') }}",
       "assigned_object_type": "dcim.interface",
       "assigned_object_id": "={{ $fromAI('interface_id') }}",
       "status": "active"
     }
     ```
4. Click **Save**

---

## Step 12: Add Create Device Tool

1. Click **"+"** on the **Tool** connector
2. Search **"HTTP Request"** and select it
3. Configure:
   - **Description:** `Use this tool to create a new device in NetBox. Use this when a user wants to add a device to inventory. Before calling this tool, look up or create the site and device type first to get their IDs. After creating the device, create an interface and assign the IP address.`
   - **Method:** `POST`
   - **URL:** `http://host.docker.internal:8000/api/dcim/devices/`
   - **Authentication:** Same NetBox token
   - **Send Body:** Toggle ON
     - **Body Content Type:** `JSON`
     - **Body:**
     ```json
     {
       "name": "={{ $fromAI('device_name') }}",
       "device_type": { "id": "={{ $fromAI('device_type_id') }}" },
       "site": { "id": "={{ $fromAI('site_id') }}" },
       "status": "active"
     }
     ```
4. Click **Save**

> Note: The full flow when a user asks to add a device with an IP address: Get Sites → Get Device Types → Get Manufacturers → (create any that don't exist) → Create Device → Create Interface → Assign IP Address. The agent handles this automatically based on the tool descriptions.

---

## Step 13: Add Get Device Roles Tool

Required because NetBox mandates a device role when creating a device.

1. Click **"+"** on the **Tool** connector
2. Search **"HTTP Request"** and select it
3. Configure:
   - **Description:** `Use this tool to get a list of all available device roles in NetBox. Use this before creating a device to find the correct role ID.`
   - **Method:** `GET`
   - **URL:** `http://host.docker.internal:8000/api/dcim/device-roles/`
   - **Authentication:** Same NetBox token
4. Click **Save**

---

## Step 14: Update Create Device Tool — Add Role Field

Go back into the **Create Device** HTTP Request node and update:

**Description** (updated):
```
Use this tool to create a new device in NetBox. Before calling this tool, look up or create the site and device type first to get their IDs, and look up the device role ID. After creating the device, create an interface and assign the IP address.
```

**Body** (updated to include role):
```json
{
  "name": "{{ $fromAI('device_name') }}",
  "device_type": { "id": "{{ $fromAI('device_type_id') }}" },
  "site": { "id": "{{ $fromAI('site_id') }}" },
  "role": { "id": "{{ $fromAI('role_id') }}" },
  "status": "active"
}
```

---

## Fixes: Prevent Duplicate Creation

Update the descriptions on the following tools to prevent the agent from trying to create records that already exist:

**Create Site** — add to description:
> ONLY call this if the site was not found after checking with the Get Sites tool.

**Create Manufacturer** — add to description:
> ONLY call this if the manufacturer was not found after checking with the Get Manufacturers tool.

**Create Device Type** — add to description:
> ONLY call this if the device type was not found after checking with the Get Device Types tool.

---

## Important: Expression Syntax in JSON Body

When using `$fromAI()` inside the JSON body of an HTTP Request node, do **not** include the `=` prefix:

| Context | Correct syntax |
|---|---|
| Regular n8n field | `={{ $fromAI('field_name') }}` |
| JSON body textarea | `{{ $fromAI('field_name') }}` |

---

## End-to-End Test Result

**Test prompt:** "Add a new device called test-router-01, it's a Cisco Catalyst 9300, located at the Toronto site, with IP address 10.0.0.50/24"

**Agent output:**
| Field | Value |
|---|---|
| Device Name | test-router-01 |
| Device Type | Cisco Catalyst 9300 |
| Manufacturer | Cisco |
| Role | Router |
| Site | Toronto |
| Status | Active |
| Interface | GigabitEthernet0/0 |
| IP Address | 10.0.0.50/24 |

The agent automatically chained all required steps: looked up existing records, created Toronto site and Catalyst 9300 device type, created the device, created the interface, and assigned the IP — all from a single chat message.

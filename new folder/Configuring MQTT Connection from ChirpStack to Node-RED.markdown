# Configuring MQTT Connection from ChirpStack to Node-RED

This guide provides step-by-step instructions to configure an MQTT connection to forward data from ChirpStack to Node-RED, including Node-RED node configuration to receive data. Screenshots illustrate each step for clarity.

## Prerequisites
- ChirpStack installed and running (e.g., via Docker or on a server).
- Node-RED installed and accessible (e.g., on a Raspberry Pi, local machine, or cloud server).
- A working LoRaWAN device registered in ChirpStack.
- MQTT broker details (e.g., Mosquitto or ChirpStack’s built-in broker).
- Basic familiarity with ChirpStack and Node-RED interfaces.

## Step 1: Verify ChirpStack MQTT Integration
ChirpStack publishes device events (e.g., uplink data) to an MQTT broker. Ensure the MQTT integration is enabled and configured in ChirpStack.

1. **Log in to ChirpStack**:
   - Open your browser and navigate to the ChirpStack web interface (e.g., `http://<chirpstack-server-ip>:8080`).
   - Log in with your credentials.

2. **Navigate to Applications**:
   - From the left sidebar, select **Applications**.
   - Choose the application containing your device.

3. **Check MQTT Integration**:
   - In the application, go to the **Integrations** tab.
   - Ensure the **MQTT** integration is enabled. If not, click **Add** and select **MQTT**.
   - Note the MQTT topic structure, typically `application/<application-id>/device/<dev-eui>/event/up` for uplinks.
   - If using an external MQTT broker, configure the broker’s address, port (usually 1883 for non-TLS, 8883 for TLS), and credentials.

   ![ChirpStack MQTT Integration](https://i.imgur.com/example-chirpstack-mqtt.png)

4. **Obtain MQTT Broker Details**:
   - If using ChirpStack’s built-in broker, the server is usually `localhost:1883` (or the server IP if remote).
   - For external brokers (e.g., Mosquitto), note the IP, port, and credentials.
   - If TLS is enabled, download the CA certificate, client certificate, and key from the **Get certificate** button (if available).

## Step 2: Configure Node-RED to Receive MQTT Data
Node-RED will subscribe to ChirpStack’s MQTT topics to receive device data.

1. **Access Node-RED**:
   - Open Node-RED in your browser (e.g., `http://<node-red-ip>:1880`).

2. **Install ChirpStack Nodes (Optional)**:
   - For easier integration, install the `node-red-contrib-chirpstack` package.
   - In Node-RED, go to **Menu** (top-right) > **Manage palette** > **Install** tab.
   - Search for `@chirpstack/node-red-contrib-chirpstack` and click **Install**.

   ![Install ChirpStack Nodes](https://i.imgur.com/example-nodered-palette.png)

3. **Create a New Flow**:
   - In the Node-RED editor, click the **+** tab to create a new flow.
   - Name it (e.g., “ChirpStack MQTT”).

4. **Add MQTT In Node**:
   - From the left sidebar, drag an **mqtt in** node (under the **input** category) to the canvas.
   - Double-click the node to configure it.
   - Set the following:
     - **Server**: Click the pencil icon to add a new MQTT broker.
       - **Server**: Enter the MQTT broker’s IP or hostname (e.g., `localhost` or `<chirpstack-server-ip>`).
       - **Port**: Default is `1883` (or `8883` for TLS).
       - **Client ID**: Optional, leave blank or set a unique ID.
       - **Credentials**: Add username/password if required.
       - **TLS**: If using TLS, upload the CA certificate, client certificate, and key (from Step 1).
     - **Topic**: Use `application/+/device/+/event/up` to subscribe to all uplink events for all devices in all applications (wildcards `+` match any ID). For a specific device, use `application/<application-id>/device/<dev-eui>/event/up`.
     - **QoS**: Set to `2` for guaranteed delivery.
     - **Output**: Leave as default (`auto-detect`).

   - Click **Done** and **Add** to save the broker settings.

   ![MQTT In Node Configuration](https://i.imgur.com/example-mqtt-in-config.png)

5. **Add Debug Node**:
   - Drag a **debug** node (under the **output** category) to the canvas.
   - Connect the output of the **mqtt in** node to the input of the **debug** node by dragging a wire between them.
   - Double-click the **debug** node and set:
     - **Output**: `msg.payload`.
     - **To**: `sidebar`.

   ![Debug Node Connection](https://i.imgur.com/example-debug-node.png)

6. **(Optional) Add ChirpStack Device Event Node**:
   - If you installed `node-red-contrib-chirpstack`, drag a **device event** node (under the **ChirpStack** category) to the canvas.
   - Connect the **mqtt in** node’s output to the **device event** node’s input.
   - Double-click the **device event** node and set:
     - **Event Type**: Select `Uplink` to filter uplink events.
   - Connect the **device event** node’s output to the **debug** node.
   - This node parses MQTT payloads into structured objects, making data easier to handle.

   ![Device Event Node](https://i.imgur.com/example-device-event.png)

7. **Deploy the Flow**:
   - Click the red **Deploy** button (top-right).
   - Check the **mqtt in** node; it should show a green dot with “Connected” if the broker connection is successful.

   ![Deployed Flow](https://i.imgur.com/example-deployed-flow.png)

## Step 3: Test the Configuration
1. **Send Uplink Data**:
   - Ensure your LoRaWAN device is active and sending uplink messages to ChirpStack.
   - Alternatively, use ChirpStack’s device simulator or enqueue a test message.

2. **Check Node-RED Debug Output**:
   - In Node-RED, open the **Debug** tab in the right sidebar.
   - When the device sends an uplink, you should see the payload in the debug window.
   - The payload is typically a JSON object containing device data (e.g., `devEui`, `fPort`, `data`).

   ![Debug Output](https://i.imgur.com/example-debug-output.png)

3. **Troubleshoot if Needed**:
   - If no data appears:
     - Verify the MQTT topic matches ChirpStack’s configuration.
     - Check broker connectivity (IP, port, credentials).
     - Ensure the device is sending uplinks (check ChirpStack’s **Device** > **LoRaWAN frames** tab).
     - Confirm Node-RED is subscribed to the correct topic.

## Step 4: Process Data in Node-RED (Optional)
To process the received data, add nodes to the flow.

1. **Add a Function Node**:
   - Drag a **function** node (under the **function** category) to the canvas.
   - Connect the **device event** (or **mqtt in**) node’s output to the **function** node’s input.
   - Double-click the **function** node and add code to process the payload, e.g.:

     ```javascript
     // Extract temperature from payload (assuming data is base64-encoded)
     let payload = msg.payload;
     if (payload.data) {
         let decoded = Buffer.from(payload.data, 'base64').toString('hex');
         msg.payload = { temperature: parseInt(decoded, 16) };
     }
     return msg;
     ```

   - This example assumes the `data` field is base64-encoded and converts it to a hexadecimal temperature value.

   ![Function Node](https://i.imgur.com/example-function-node.png)

2. **Add a Dashboard Node**:
   - Install `node-red-dashboard` via **Manage palette** if not already installed.
   - Drag a **gauge** node (under the **dashboard** category) to the canvas.
   - Connect the **function** node’s output to the **gauge** node.
   - Configure the **gauge**:
     - **Group**: Create a new group (e.g., “Device Data”).
     - **Label**: Set to “Temperature”.
     - **Value format**: `{{msg.payload.temperature}}`.
     - **Range**: Set min/max (e.g., 0 to 100).
   - Deploy the flow.
   - Access the dashboard at `http://<node-red-ip>:1880/ui`.

   ![Gauge Node](https://i.imgur.com/example-gauge-node.png)

   ![Dashboard Output](https://i.imgur.com/example-dashboard.png)

## Step 5: Secure the Connection (Optional)
For production environments, secure the MQTT connection:
- Enable TLS in the MQTT broker settings (port 8883).
- In ChirpStack, upload TLS certificates under **Integrations** > **MQTT**.
- In Node-RED’s **mqtt in** node, configure TLS with the same certificates.
- Use username/password authentication for the MQTT broker.

## Notes
- Replace `<application-id>` and `<dev-eui>` with actual values from ChirpStack.
- The `node-red-contrib-chirpstack` package simplifies parsing but is optional; raw MQTT payloads can be processed with JSON and function nodes.
- For high-frequency data, add a **limit rate** node to prevent overwhelming Node-RED.
- Always redeploy the flow after changes.

This configuration enables Node-RED to receive and process ChirpStack device data via MQTT, with a visual dashboard for monitoring.
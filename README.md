# 001-logging-sidecar.md
# Use Sidecar Pattern for Legacy Application Log Aggregation

## Context and Problem Statement
We maintain a legacy Python application (`OrderService`) that writes application logs to a local disk file (`/var/log/app.log`). We are migrating this application to a Kubernetes cluster. In Kubernetes, local disk storage is ephemeral; if the container restarts, all log data is lost. Furthermore, our organization requires all logs to be centralized in an Elasticsearch cluster for audit and monitoring purposes. The legacy application code is fragile, and we lack the resources to rewrite its internal logging module to support network streaming.

## Decision Drivers
*   **Data Persistence:** Logs must not be lost if the application container crashes.
*   **Code Stability:** We must minimize changes to the legacy source code to avoid regression bugs.
*   **Centralization:** Logs must be shipped to Elasticsearch in near real-time.
*   **Decoupling:** The logging mechanism should be independent of the application logic.

## Considered Options
*   **Option 1: Modify Application Code:** Rewrite the Python logging module to send logs directly to Elasticsearch.
*   **Option 2: Node-Level Logging Agent (DaemonSet):** Rely on a logging agent running on the host node to scrape Docker log files.
*   **Option 3: Sidecar Pattern:** Deploy a dedicated logging container alongside the application container in the same Pod.

## Decision Outcome
Chosen option: **Option 3: Sidecar Pattern**, because it allows us to achieve log centralization without modifying a single line of the legacy application's source code, thereby eliminating the risk of breaking business logic.

### Positive Consequences
*   **Zero Code Changes:** The legacy application continues to write to disk as it always has.
*   **Isolation:** If the logging agent crashes, the main application is unaffected.
*   **Flexibility:** We can change the logging destination (e.g., switch from Elasticsearch to Splunk) by updating the Sidecar configuration, not the app code.

### Negative Consequences
*   **Resource Usage:** Running a second container per pod increases CPU and Memory consumption.
*   **Complexity:** Requires configuring a "Shared Volume" in Kubernetes so both containers can access the same file.

## Pros and Cons of the Options

### Option 1: Modify Application Code
*   *Pros:* Most efficient resource usage (no extra container).
*   *Cons:* High risk of bugs; requires deep knowledge of legacy code; tightly couples app to logging infrastructure.

### Option 2: Node-Level Logging Agent
*   *Pros:* Simplest infrastructure (one agent per server).
*   *Cons:* The legacy app writes to a custom file path inside the container, which is difficult for a node-level agent to access without complex configuration.

## Sample Code

### Kubernetes Pod Configuration (YAML)

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: legacy-app-with-sidecar
spec:
  volumes:
    - name: shared-logs
      emptyDir: {}  # 1. Create a shared ephemeral disk

  containers:
    # --- Container 1: The Main Legacy App ---
    - name: legacy-python-app
      image: my-company/legacy-app:v1
      volumeMounts:
        - name: shared-logs
          mountPath: /var/log  # App thinks it's writing to local disk
      # App writes to /var/log/app.log normally

    # --- Container 2: The Sidecar (Log Shipper) ---
    - name: log-sidecar
      image: fluent/fluent-bit:latest
      volumeMounts:
        - name: shared-logs
          mountPath: /var/log  # Sidecar sees the EXACT same folder
      # Sidecar command to read the file and print/send it
      command: ["/fluent-bit/bin/fluent-bit", "-i", "tail", "-p", "path=/var/log/app.log", "-o", "stdout"]

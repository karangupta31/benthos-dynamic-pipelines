# Benthos Dynamic Pipeline Orchestrator

This project provides a Benthos configuration that acts as a dynamic orchestrator. It listens for control messages on a GCP Pub/Sub subscription and, based on each message, generates and launches a new, separate Benthos instance to perform a data processing task (e.g., Kafka to Pub/Sub).

This serves as a foundational pattern for scenarios where Benthos pipelines need to be created and managed dynamically using Benthos itself.

**Current Implementation:** Subprocess-based (using `benthos -c` via `rpk connect run`).

**Future Goal:** Transition to Benthos Streams API for more robust management within a single Benthos process.

## How it Works

1.  **Control Message Reception:** The main orchestrator Benthos instance listens to a GCP Pub/Sub subscription (e.g., `benthos-control-sub`).
2.  **Dynamic Configuration Generation:**
    *   Upon receiving a control message (JSON payload), it uses the `dynamic_k2p_config.blobl` Bloblang template to construct a complete Benthos YAML configuration string.
    *   This generated YAML defines a new Benthos pipeline, typically for consuming from Kafka and producing to GCP Pub/Sub, with parameters specified in the control message.
    *   The control message can also include a custom Bloblang mapping string in its `data` field, which will be embedded into the generated pipeline for data transformation.
3.  **Subprocess Launch:**
    *   The generated YAML is written to a temporary file (`/tmp/config_temp.yaml`).
    *   A new Benthos process is launched in the background using `nohup rpk connect run /tmp/config_temp.yaml &` (which executes `benthos -c /tmp/config_temp.yaml`).
4.  **Acknowledgement:** After a delay (to allow the subprocess to start), the control message is acknowledged.

## Key Components

*   `benthos.yaml`: The configuration for the main orchestrator Benthos instance.
*   `dynamic_k2p_config.blobl`: The Bloblang template used to generate configurations for the dynamically launched Benthos instances.

## Prerequisites

1.  **Benthos:** Installed and accessible in your PATH. (Version 4.x recommended). You can use the standalone `benthos` binary or tools that bundle it, like `rpk`.
2.  **GCP Project:**
    *   A Pub/Sub topic for control messages.
    *   A Pub/Sub subscription for the orchestrator to listen to (e.g., `benthos-control-sub`).
    *   Pub/Sub topics for the dynamically launched instances to publish to.
    *   Appropriate GCP credentials configured for Benthos (e.g., via `GOOGLE_APPLICATION_CREDENTIALS` environment variable).
3.  **Kafka Cluster:**
    *   Accessible by the environment where Benthos will run.
    *   Topics that the dynamic instances will consume from.
4.  **Environment Variables:** The `dynamic_k2p_config.blobl` template expects Kafka credentials and broker information to be available as environment variables where the *dynamic Benthos instances are launched*:
    *   `KAFKA_BROKERS`
    *   `KAFKA_MECHANISM`
    *   `KAFKA_USERNAME`
    *   `KAFKA_PASSWORD`
        Ensure these are set in the environment where the main orchestrator `benthos` process runs, as it will spawn child processes that inherit its environment.

## Running the Orchestrator

1.  Ensure all prerequisites are met and environment variables are set.
2.  Place `benthos.yaml` and `dynamic_k2p_config.blobl` in the same directory.
3.  Navigate to that directory and run:
    *   Using `rpk` (Redpanda's tool, which bundles Benthos):
        ```bash
        rpk connect run benthos.yaml
        ```
    *   Or, using a standalone Benthos binary:
        ```bash
        benthos -c benthos.yaml
        ```

## Sending a Control Message

Publish a JSON message to the GCP Pub/Sub topic that the orchestrator is subscribed to (e.g., `projects/<YourProject>/topics/your-control-topic`).

**Example Control Message Payload:**

```json
{
  "kafka_topic": "source-events-main",
  "publish_project": "<YourGCPProjectID>",
  "publish_topic": "processed-events-destination",
  "consumer_name": "dynamic-consumer-group-main",
  "client_id": "my-dynamic-benthos-1",
  "data":"{\"data\":{\"userId\":this.distinct_id.string(),\"key1\":\"val1\",\"key1\":\"val1\",\"metaInfo\":{\"abc\":\"23432\",\"id\":234}}}"
}
```
Output:
```json
{
   "data":{
      "metaInfo":{
         "abc":"23432",
         "id":234
      },
      "key1":"val1",
      "key2":"val2",
      "userId":"841622982"
   }
}
```

## Constructing the `data` Field in Control Messages

The `data` field within your JSON control message is special. Its value must be a **JSON string** that itself represents the *body* of a Bloblang mapping. This mapping will be applied by the dynamically launched Benthos instance to each message it processes from its source (e.g., Kafka).

The orchestrator's `dynamic_k2p_config.blobl` takes this string and effectively does:
`root.pipeline.processors = root.pipeline.processors.append({ "mapping": "root = " + this.data })`
(Where `this.data` is the string you provide in the control message).

**How to Create the `data` String:**

1.  **Define Your Target JSON Structure:** Decide what the output JSON from the dynamic Benthos instance should look like for each processed message.
2.  **Embed Bloblang Expressions for Dynamic Values:** For parts of your target JSON that need to come from the *source message* (e.g., a field from a Kafka message), use Bloblang expressions.
    *   `this.field_name`: Accesses a field from the source message.
    *   `.string()`: Ensures the value is treated as a string.
    *   `.number()`: Ensures the value is treated as a number.
    *   `meta("kafka_key")`: Accesses Kafka message metadata.
3.  **Format as a String Literal:** The entire structure, including Bloblang expressions, must be formatted as a single JSON string value for the `data` key.

**Example:**

Let's say your source Kafka messages look like:
```json
{
  "distinct_id": "841622982",
  "event_name": "page_view",
  "properties": {
    "page_url": "/home"
  }
}
```
And you want to transform it to:
```json
{
  "userId": "841622982",
  "key1": "val1",
  "key2": "val2",
  "metaInfo": {
    "abc": "23432",
    "id": 234
  }
}
```
The `data` field in your control message would look like:
```json
{
  "data": "{\"data\":{\"userId\":this.distinct_id.string(),\"key1\":\"val1\",\"key2\":\"val2\",\"metaInfo\":{\"abc\":\"23432\",\"id\":234}}}"
}
```

## Future Enhancements & Considerations (The Road to a Robust Orchestrator)

While the current subprocess-based approach is functional, for a more robust and production-ready orchestrator, we'll move to using the Benthos Streams API. This transition will provide several advantages, including better lifecycle management, observability, and resource efficiency.:

**Transition to Benthos Streams API:**

**Why this is the most impactful change:** 
   The Benthos HTTP API (specifically the `/streams` endpoint) allows for creating, managing (reading status, updating), and deleting Benthos streams (pipelines) *within a single, long-running Benthos process*.
   **Key Benefits:**
   *   **Eliminates Subprocesses:** No more `nohup benthos ... &`. This significantly simplifies process management and reduces system overhead.
   *   **No Temporary Files:** Configurations are sent directly via the API, removing the need for `/tmp/config_temp.yaml` and associated race condition concerns (even if currently mitigated by `max_outstanding_messages: 1`).
   *   **Centralized Observability:** Logs and metrics from all dynamically created streams can be managed and exposed by the main orchestrator Benthos instance, providing a unified view.
   *   **Lifecycle Management:** The Streams API provides endpoints to `CREATE` streams, check their status (`GET /streams/{id}` which shows uptime and activity, allowing inference of completion for finite inputs like `read_until`), update (`PUT /streams/{id}`), and delete (`DELETE /streams/{id}`) dynamic pipelines gracefully. This enables programmatic polling of stream status and robust lifecycle control.
   *   **Resource Efficiency:** Running multiple stream configurations within one Benthos process is generally more resource-efficient than spawning many individual Benthos processes.
   
**How it would work:**
   *   The orchestrator's main Benthos pipeline would change. Instead of `command` processors to write a file and launch a subprocess, it would use an `http_client` processor (or `http` output).
   *   This `http_client` would make `POST /streams/{id}` requests to its *own* Benthos API endpoint (e.g., `http://localhost:4195/streams/{dynamic_stream_id}`), assuming the main orchestrator's HTTP server is enabled.
   *   The body of the HTTP `POST` request would be the dynamically generated Benthos configuration. **Crucially, the `dynamic_k2p_config.blobl` (or a similar Bloblang mapping used for this purpose) would need to be adjusted to output a Benthos configuration in JSON format**, as the Streams API endpoint expects JSON, not YAML.

# Contributing
Contributions are welcome! Please open an issue to discuss any changes or new features before submitting a pull request. Focus areas include implementing the "Future Enhancements" listed above, especially the transition to the Benthos Streams API.

# License
This project is licensed under the MIT License.
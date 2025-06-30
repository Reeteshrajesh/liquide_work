# 🎯 Jitsu Kubernetes Pipeline: JS Events API → S3 → Amplitude

This repository provides a ready-to-deploy Kubernetes configuration (`jitsu.yaml`) to ingest events **from your app using the Jitsu JavaScript SDK (Events API)**, route them through **Jitsu**, store them in **Amazon S3**, and ingest them into **Amplitude** for analytics.

---

## 🧭 Workflow Overview

1. **Event Generation:**
   - Your application integrates the **Jitsu JavaScript SDK**.
   - The SDK sends events to the **Jitsu Events API** using your js event API key.

2. **Ingestion into Jitsu:**
   - Jitsu receives incoming events from the browser via the **JS Events API**.

3. **Transformation in Jitsu:**
   - Each event is optionally transformed in Jitsu before delivery to S3 in json formate.
   - Example transformation (in Destination settings):

```yaml
const context = $.eventn_ctx || $;
const date = new Date();
const user = context.user || {};
const properties = context.properties || {};
const evnt = $.event_type || context.event_type || $.eventName || context.event || $.event;
const usrId = user._id || $.userId || user.user_id;
return {
     user_id: usrId,
     device_id: $.device_id || user.anonymous_id || context.anonymous_id || usrId+"_default_device",
     session_id: $.session_id || "-1",
     event_type: evnt,
     evt: context.utc_time || $.eventTime || date.toISOString(),
     os_name: context.os_name,
     os_version: context.os_version,
     device_brand: context.device_brand,
     device_manufacturer: context.device_family,
     device_model: context.device_model,
     platform: context.os_name,
     ip: $.source_ip,
     insert_id: $.eventn_ctx_event_id || context.event_id,
     event_epoch: date.getTime(),
     user_agent: context.user_agent,
     event_properties: {
         url: context.url,
         click_id: context.click_id,
         host: context.doc_host,
         path: context.doc_path,
         search: context.doc_search,
         app: $.app,
         referrer: context.referer,
         title: context.page_title,
         src: $.src,
         user_agent: context.user_agent,
         vp_size: $.vp_size,
         local_tz_offset: context.local_tz_offset,
         evt: context.utc_time || context.eventTime,
         ...properties
     },
     user_properties: user
};
```

4. **Delivery to S3:**
   - Transformed events are written into your **S3 bucket**.

5. **Ingestion into Amplitude:**
   - **Amplitude** is configured to use **S3 as a Source**.
   - Amplitude accesses the S3 bucket using an **IAM Role** with read permissions.
   - Events are optionally transformed again in Amplitude:
```yaml
     {
    "converterConfig": {
        "fileType": "json",
        "compressionType": "gzip",
        "convertToAmplitudeFunc": {
            "user_id": "user_id",
            "device_id": "device_id",
            "event_type": "event_type",
            "time": [
                "iso_time_to_ms",
                "evt"
            ],
            "event_properties": "event_properties",
            "user_properties": "user_properties",
            "os_name": "os_name",
            "os_version": "os_version",
            "platform": "platform",
            "user_agent": "user_agent",
            "ip_address": "ip",
            "device_family": "device_manufacturer",
            "device_type": "device_model",
            "session_id": "session_id",
            "display_name": "event_type",
            "ip": "ip"
        }
    },
    "keyValidatorConfig": {
        "filterPattern": ".*.log.gz"
    }
}
```

---

## ⚙️ Components Deployed

The Kubernetes stack includes:

- **Redis:** Lightweight in-memory store for caching.
- **PostgreSQL:** Metadata store for Jitsu configurations.
- **Jitsu:** Event ingestion and routing engine.
- **Services:** Internal Kubernetes services for connectivity.
- **(Optional)** Dedicated node scheduling for Jitsu pods.

---

## 🚀 Quick Start Guide

Follow these steps to set up the pipeline end-to-end.

---

### 1️⃣ Prerequisites

✅ Kubernetes Cluster (EKS, GKE, Minikube, etc.)  
✅ `kubectl` configured and working  
✅ An **S3 bucket** created for event storage  
✅ An **IAM Role** for Amplitude with `GetObject` and `ListBucket` permissions  
✅ An **Amplitude Account**

---

### 2️⃣ Download `jitsu.yaml`

Clone this repository or download the manifest:

```bash
git clone <your-repo-url>
cd <your-repo-directory>
````

Or download directly:

```bash
curl -O https://<your-repo-or-storage>/jitsu.yaml
```

---

### 3️⃣ Deploy to Kubernetes

Apply the resources:

```bash
kubectl apply -f jitsu.yaml
```

Verify the components:

```bash
kubectl get pods -n jitsu-stack
```

---

### 4️⃣ Configure Jitsu

1. **Access the Jitsu UI:**

   Port-forward the service:

   ```bash
   kubectl port-forward -n jitsu-stack svc/jitsu 8000:8000
   ```

   Open in your browser:

   ```
   http://localhost:8000
   ```

2. **Create a JS Events Source:**

   * In the Jitsu UI, create a **Source**.
   * Choose **JavaScript Events API**.
   * Copy the **Public API Key** generated for your project.

3. **Create an S3 Destination:**

   * In the Jitsu UI, create a **Destination**.
   * Select **Amazon S3**.
   * Configure credentials and bucket details.
   * Enable compression if needed.
   * Example transformation script:

     ```javascript
     already provided above
     ```

---

### 5️⃣ Integrate the segment/jitsu SDK (using the js event api key) in Your App

---

### 6️⃣ Configure Amplitude

1. In Amplitude, go to **Data Sources > Amazon S3 > Add Source**.

2. Provide:

   * **Bucket Name:** your S3 bucket
   * **Path Prefix:** (optional)
   * **Region**
   * **IAM Role ARN**

3. Configure **Transformation Rules** (optional):

   ```javascript
   already provided above
   ```

4. Confirm ingestion by verifying events in Amplitude dashboards.

---

## ✨ Scaling & Resource Management

**Jitsu Pods:**

* Default replicas: `4`
* To scale up/down:

  ```bash
  kubectl scale deployment jitsu -n jitsu-stack --replicas=4
  ```

**Resource Recommendations:**

* **Jitsu:** \~4Gi RAM / pod
* **Redis:** \~1Gi RAM
* **PostgreSQL:** \~512Mi RAM

---

## 🧩 Optional: Dedicated Node Scheduling

To isolate Jitsu pods on dedicated nodes:

1. Taint your node:

   ```bash
   kubectl taint nodes <node-name> jitsu-only=true:NoSchedule
   ```

2. Ensure `jitsu.yaml` includes:

   ```yaml
   tolerations:
     - key: "jitsu-only"
       operator: "Equal"
       value: "true"
       effect: "NoSchedule"
   nodeSelector:
     jitsu-dedicated: "true"
   ```

---

## 🛠 Monitoring & Logs

**View Jitsu logs:**

```bash
kubectl logs -n jitsu-stack deployment/jitsu
```

**Monitor pod status:**

```bash
kubectl get pods -n jitsu-stack
```

---

## 🧭 Workflow Recap

✅ **Your App**

* Integrates **Segment SDK with**
* Sends events via Events API

✅ **Jitsu**

* Receives and optionally transforms events
* Stores transformed data in **S3**

✅ **Amplitude**

* Ingests events from **S3**
* Applies additional transformations
* Makes events available for analytics

---

## 🔗 References

* [Jitsu Docs](https://jitsu.com/docs)
* [Jitsu JavaScript SDK](https://jitsu.com/docs/sdk/javascript)
* [AWS S3 Docs](https://docs.aws.amazon.com/s3/)
* [Amplitude S3 Source](https://help.amplitude.com/hc/en-us/articles/360056255391)

---

## 📝 Contact & Support

If you have questions or need assistance, open an issue or contact your platform administrator.

---

```

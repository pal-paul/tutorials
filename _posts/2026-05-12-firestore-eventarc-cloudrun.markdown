---
layout: post
title: "Detecting Firestore Changes and Triggering Cloud Run via Eventarc"
date: 2026-05-12 10:00:00 +0200
categories: Terraform
permalink: /:title/
---

# Detecting Firestore Changes and Triggering Cloud Run via Eventarc

**Date:** May 2026

---

## Overview

This document explains how to detect changes in Firestore data and use them to trigger a Cloud Run service using **Google Cloud Eventarc**.

---

## Architecture

```
Firestore document change
        │
        ▼
  Eventarc Trigger
  (written/created/updated/deleted)
        │
        ▼
  Cloud Run Service
  (receives CloudEvents HTTP POST)
```

Alternatively, if fan-out or buffering is needed (matching the existing pattern in this repo):

```
Firestore document change
        │
        ▼
  Eventarc Trigger
        │
        ▼
  Pub/Sub Topic
        │
        ▼
  Cloud Run (Push Subscription)
```

---

## Option 1: Eventarc → Cloud Run (Recommended)

Eventarc natively listens to Firestore document change events and routes them directly to a Cloud Run service.

### gcloud CLI example

```bash
gcloud eventarc triggers create firestore-trigger \
  --location=europe-west1 \
  --destination-run-service=your-cloud-run-service \
  --destination-run-region=europe-west1 \
  --event-filters="type=google.cloud.firestore.document.v1.written" \
  --event-filters="database=(default)" \
  --event-filters-path-pattern="document=your-collection/{document}" \
  --service-account=your-sa@project.iam.gserviceaccount.com
```

### Terraform example

```hcl
# Grant the service account the Eventarc receiver role
resource "google_project_iam_member" "sa_eventarc_receiver" {
  project = var.project
  role    = "roles/eventarc.eventReceiver"
  member  = "serviceAccount:${google_service_account.service_account.email}"
}

# Grant Firestore permission to publish to Eventarc
resource "google_project_iam_member" "firestore_eventarc_publisher" {
  project = var.project
  role    = "roles/eventarc.publisher"
  member  = "serviceAccount:service-${data.google_project.project.number}@gcp-sa-firestore.iam.gserviceaccount.com"
}

# Eventarc trigger: Firestore document write → Cloud Run
resource "google_eventarc_trigger" "firestore_change_trigger" {
  name     = "${var.component}-firestore-trigger"
  project  = var.project
  location = var.region

  matching_criteria {
    attribute = "type"
    value     = "google.cloud.firestore.document.v1.written"
  }
  matching_criteria {
    attribute = "database"
    value     = "(default)"
  }
  matching_criteria {
    attribute = "document"
    value     = "your-collection/{document}"
    operator  = "match-path-pattern"   # required for wildcard paths
  }

  destination {
    cloud_run_service {
      service = google_cloud_run_v2_service.service_queuer.name
      region  = var.region
      path    = "/firestore-event"     # endpoint on your Cloud Run service
    }
  }

  service_account = google_service_account.service_account.email

  labels = {
    env = var.env
  }
}
```

---

## Option 2: Eventarc → Pub/Sub → Cloud Run

Use this when you need fan-out (multiple consumers) or want to reuse the existing Pub/Sub subscriber pattern already present in this repo.

```hcl
# Eventarc trigger routes Firestore events to a Pub/Sub topic
resource "google_eventarc_trigger" "firestore_pubsub_trigger" {
  name     = "${var.component}-firestore-pubsub-trigger"
  project  = var.project
  location = var.region

  matching_criteria {
    attribute = "type"
    value     = "google.cloud.firestore.document.v1.written"
  }
  matching_criteria {
    attribute = "database"
    value     = "(default)"
  }
  matching_criteria {
    attribute = "document"
    value     = "your-collection/{document}"
    operator  = "match-path-pattern"
  }

  destination {
    cloud_run_service {
      service = google_cloud_run_v2_service.service_queuer.name
      region  = var.region
      path    = "/enqueue"
    }
  }

  service_account = google_service_account.service_account.email
}
```

---

## Firestore Event Types

| Event Type                                   | Fires When                                |
| -------------------------------------------- | ----------------------------------------- |
| `google.cloud.firestore.document.v1.written` | Document created, updated, **or** deleted |
| `google.cloud.firestore.document.v1.created` | New document created only                 |
| `google.cloud.firestore.document.v1.updated` | Existing document changed                 |
| `google.cloud.firestore.document.v1.deleted` | Document deleted                          |

---

## CloudEvents Payload

Your Cloud Run service receives an HTTP POST with a `Content-Type: application/cloudevents+json` header. The body:

```json
{
  "specversion": "1.0",
  "type": "google.cloud.firestore.document.v1.written",
  "source": "//firestore.googleapis.com/projects/my-project/databases/(default)",
  "subject": "documents/your-collection/doc-id",
  "id": "unique-event-id",
  "time": "2026-05-12T10:00:00Z",
  "datacontenttype": "application/json",
  "data": {
    "oldValue": {
      "name": "projects/my-project/databases/(default)/documents/your-collection/doc-id",
      "fields": { "status": { "stringValue": "pending" } }
    },
    "value": {
      "name": "projects/my-project/databases/(default)/documents/your-collection/doc-id",
      "fields": { "status": { "stringValue": "processed" } }
    },
    "updateMask": {
      "fieldPaths": ["status"]
    }
  }
}
```

- `oldValue` — state before the change (`null` on create)
- `value` — state after the change (`null` on delete)
- `updateMask.fieldPaths` — list of fields that changed (only present on update)

---

## Go Handler Example (Cloud Run)

```go
package main

import (
    "encoding/json"
    "io"
    "log"
    "net/http"
)

type FirestoreEvent struct {
    OldValue   map[string]interface{} `json:"oldValue"`
    Value      map[string]interface{} `json:"value"`
    UpdateMask struct {
        FieldPaths []string `json:"fieldPaths"`
    } `json:"updateMask"`
}

func firestoreEventHandler(w http.ResponseWriter, r *http.Request) {
    body, err := io.ReadAll(r.Body)
    if err != nil {
        http.Error(w, "failed to read body", http.StatusBadRequest)
        return
    }

    // CloudEvents wraps the Firestore payload in the "data" field
    var envelope struct {
        Data FirestoreEvent `json:"data"`
    }
    if err := json.Unmarshal(body, &envelope); err != nil {
        http.Error(w, "failed to parse event", http.StatusBadRequest)
        return
    }

    event := envelope.Data
    log.Printf("changed fields: %v", event.UpdateMask.FieldPaths)
    log.Printf("new value: %v", event.Value)

    // Your business logic here

    w.WriteHeader(http.StatusOK)
}
```

---

## Required IAM Roles

| Principal                                                              | Role                           | Purpose                                        |
| ---------------------------------------------------------------------- | ------------------------------ | ---------------------------------------------- |
| Cloud Run service account                                              | `roles/eventarc.eventReceiver` | Receive Eventarc events                        |
| Firestore service account (`gcp-sa-firestore.iam.gserviceaccount.com`) | `roles/eventarc.publisher`     | Publish change events to Eventarc              |
| Cloud Run service account                                              | `roles/run.invoker`            | Allow Eventarc to invoke the Cloud Run service |

---

## Prerequisites

- Google provider `>= 4.50` for `google_eventarc_trigger` with Firestore support
- Firestore must be in **Native mode** (not Datastore mode)
- Eventarc API must be enabled: `gcloud services enable eventarc.googleapis.com`

---

## References

- [Eventarc Terraform resource](https://registry.terraform.io/providers/hashicorp/google/latest/docs/resources/eventarc_trigger)
- [Firestore Eventarc event types](https://cloud.google.com/eventarc/docs/reference/supported-events#cloud-firestore)
- [CloudEvents spec](https://cloudevents.io/)

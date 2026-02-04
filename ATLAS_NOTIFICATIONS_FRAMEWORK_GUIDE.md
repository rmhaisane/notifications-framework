# Atlas Notifications Framework - Integration Guide

## Table of Contents
1. [Overview](#overview)
2. [Architecture](#architecture)
3. [Service Components](#service-components)
4. [Integration Steps](#integration-steps)
5. [Event Structure](#event-structure)
6. [Channel Types](#channel-types)
7. [Configuration Requirements](#configuration-requirements)
8. [Code Examples](#code-examples)
9. [Deployment Requirements](#deployment-requirements)
10. [API Reference](#api-reference)

---

## Overview

The Atlas Notifications Framework is a comprehensive event-driven notification system designed for Infoblox services. It enables services to publish events that are processed, filtered, and delivered to users through multiple channels (Email, In-App, Slack, Webhook, PagerDuty).

### Key Features
- **Multi-channel delivery**: Email, In-App, Slack, Webhook, PagerDuty
- **Event deduplication and summarization**: Prevents notification fatigue
- **Configurable subscriptions**: Users can customize their notification preferences
- **Template-based notifications**: Customizable notification templates per channel
- **Thresholding support**: Prometheus-based alerting rules integration

---

## Architecture

```
┌─────────────────┐     ┌──────────────────────┐     ┌─────────────────────┐
│  Your Service   │────▶│  Events Processor    │────▶│   Dispatcher        │
│  (Event Client) │     │  (Deduplication,     │     │   (Routing,         │
└─────────────────┘     │   Summarization)     │     │    AlertManager)    │
                        └──────────────────────┘     └─────────────────────┘
                                   │                           │
                                   ▼                           ▼
                        ┌──────────────────────┐     ┌─────────────────────┐
                        │   Config Service     │     │      Sender         │
                        │   (Subscriptions,    │     │   (Webhook,         │
                        │    Templates,        │     │    PagerDuty,       │
                        │    Channels)         │     │    InApp)           │
                        └──────────────────────┘     └─────────────────────┘
                                                               │
                                   ┌───────────────────────────┼───────────────┐
                                   ▼                           ▼               ▼
                        ┌──────────────────┐     ┌─────────────────┐  ┌──────────────┐
                        │    Mailbox       │     │   Alert Adapter │  │  PostOffice  │
                        │  (In-App Alerts) │     │   (Legacy API)  │  │   (Email)    │
                        └──────────────────┘     └─────────────────┘  └──────────────┘
```

---

## Service Components

### 1. **atlas.notifications.events.client** (Client Library)
- **Purpose**: Go client library to publish events to the notification pipeline
- **Repository**: `github.com/Infoblox-CTO/atlas.notifications.events.client`
- **Usage**: Import and use in your service to publish events

### 2. **atlas.notifications.events.processor**
- **Purpose**: Receives events, deduplicates, summarizes, and forwards to dispatcher
- **Database**: PostgreSQL for event storage and summarization rules
- **Features**:
  - Event fingerprinting for deduplication
  - Configurable summarization rules
  - Feature flag integration
  - Prometheus metrics

### 3. **atlas.notifications.config**
- **Purpose**: Configuration management for the notification pipeline
- **Database**: PostgreSQL
- **Manages**:
  - Subscriptions
  - Triggers
  - Channels
  - Channel Types
  - Notification Templates
  - Filters
  - Summarization Rules

### 4. **atlas.notifications.dispatcher**
- **Purpose**: Routes processed events to appropriate channels via AlertManager
- **Dependencies**: Prometheus AlertManager, Events Processor, Config Service

### 5. **atlas.notifications.sender**
- **Purpose**: Handles delivery to final destinations (Webhook, PagerDuty, InApp)
- **Uses**: Dapr for message queuing

### 6. **atlas.notifications.mailbox**
- **Purpose**: Stores and serves in-app notifications to the UI
- **Database**: PostgreSQL
- **API**: REST API for UI consumption

### 7. **atlas.notifications.postoffice**
- **Purpose**: Email notification delivery via SES

### 8. **atlas.notifications.alert.adapter**
- **Purpose**: Legacy API adapter for backward compatibility

### 9. **atlas.notifications.thresholding**
- **Purpose**: Interfaces with Prometheus for metric-based alerting rules
- **Dependencies**: Prometheus Operator CRDs

---

## Integration Steps

### Step 1: Add the Events Client Dependency

Add to your `go.mod`:
```go
require (
    github.com/Infoblox-CTO/atlas.notifications.events.client v1.1.0
)
```

### Step 2: Initialize the Client

```go
import (
    notifications_client "github.com/Infoblox-CTO/atlas.notifications.events.client/client"
)

func initNotificationClient(address, topic, pubsubName string) (notifications_client.Client, error) {
    return notifications_client.Connect(address, topic, pubsubName)
}
```

### Step 3: Create and Publish Events

```go
import (
    notifications_client "github.com/Infoblox-CTO/atlas.notifications.events.client/client"
    notifications_pb "github.com/Infoblox-CTO/atlas.notifications.events.client/client/pb"
    "google.golang.org/protobuf/types/known/timestamppb"
)

func publishEvent(ctx context.Context, client notifications_client.Client) error {
    event := &notifications_pb.Event{
        Type:          notifications_pb.Event_SYSTEM,  // or ACCOUNT, PRODUCT, USER
        Subtype:       "your-event-subtype",           // e.g., "license-expiration"
        AccountId:     "account-123",                  // or "*" for all accounts
        ApplicationId: "your-app-id",
        Location:      "k8s-cluster",
        Severity:      "medium",                       // low, medium, high
        Status:        notifications_pb.Event_RAISED,  // or CLEARED
        ProductName:   "your-product",
        Metadata: map[string]string{
            "key1": "value1",
            "key2": "value2",
            // Add template variables here
        },
        OccurredTime: timestamppb.Now(),
    }
    
    return client.Publish(ctx, event)
}
```

### Step 4: Register Event Subtypes (Config Service)

Your event subtypes must be registered in the Config Service database. Contact the notifications team or add migrations to create:
- Event Types
- Event Subtypes
- Categories

### Step 5: Create Default Templates

Templates define how notifications appear in each channel. Create templates via the Config Service API or database migrations.

### Step 6: Configure Helm Values

Add notification client configuration to your Helm chart:
```yaml
notifications:
  enabled: true
  pubsub:
    address: "localhost:50001"  # Dapr sidecar port
    topic: "events"
    name: "notifications-pubsub"
  appId: "your-application-id"
```

---

## Event Structure

### Event Proto Definition

```protobuf
message Event {
  Type type = 1;              // ACCOUNT, SYSTEM, PRODUCT, USER
  string subtype = 2;         // Your custom event subtype
  string account_id = 3;      // Target account or "*" for all
  string application_id = 4;  // Your application identifier
  string location = 5;        // Usually "k8s-cluster"
  string severity = 6;        // "low", "medium", "high"
  Duration TTL = 7;           // Optional time-to-live
  map<string, string> metadata = 8;  // Template variables
  Timestamp occurredTime = 9;
  Timestamp generatedTime = 10;
  string id = 11;             // Auto-generated
  Status status = 12;         // RAISED, CLEARED, REMINDER
  string product_name = 13;   // Product category
  
  enum Type {
    ACCOUNT = 0;
    SYSTEM = 1;
    PRODUCT = 2;
    USER = 3;
  }
  
  enum Status {
    RAISED = 0;
    CLEARED = 1;
    REMINDER = 2;
  }
}
```

### Metadata Keys (Template Variables)

Common metadata keys used in templates:
| Key | Description |
|-----|-------------|
| `subject` | Email subject line |
| `body` | Notification body content |
| `click_link` | Link for "Learn More" button |
| `html_content` | HTML formatted content |
| `account_name` | Account display name |
| `service_name` | Service display name |
| `fingerprint` | Custom deduplication key |

---

## Channel Types

| Channel Type | ID | Description | Targetable |
|--------------|-----|-------------|------------|
| Email | `INFOBLOX_EMAIL` | Email notifications via SES | Yes |
| InApp | `INFOBLOX_INAPP` | In-app notifications (Mailbox) | No |
| Slack | `INFOBLOX_SLACK` | Slack webhook integration | Yes |
| Webhook | Custom | Custom HTTP webhook | Yes |
| PagerDuty | Custom | PagerDuty integration | Yes |

---

## Configuration Requirements

### Environment Variables

```bash
# Dapr/PubSub Configuration
DAPR_GRPC_PORT=50001
NOTIFICATIONS_PUBSUB_TOPIC=events
NOTIFICATIONS_PUBSUB_NAME=notifications-pubsub

# Application Identity
APPLICATION_ID=your-app-id

# Feature Flags (Optional)
FEATURE_FLAG_SERVICE=feature-flag-service:8080
```

### Required Database Tables (Config Service)

The Config Service manages these entities:
1. **channel_types** - Email, InApp, Webhook, PagerDuty, Slack
2. **channels** - Per-account channel configurations
3. **triggers** - Event matching criteria
4. **subscriptions** - User notification preferences
5. **notification_templates** - Message templates per channel
6. **filters** - User email filters
7. **summarization_rules** - Event deduplication rules

---

## Code Examples

### Complete Integration Example

```go
package notifications

import (
    "context"
    "fmt"
    "time"

    notifications_client "github.com/Infoblox-CTO/atlas.notifications.events.client/client"
    notifications_pb "github.com/Infoblox-CTO/atlas.notifications.events.client/client/pb"
    "github.com/sirupsen/logrus"
    "google.golang.org/protobuf/types/known/timestamppb"
)

type NotificationService struct {
    client        notifications_client.Client
    applicationID string
    logger        *logrus.Logger
}

type NotificationConfig struct {
    DaprAddress   string
    Topic         string
    PubSubName    string
    ApplicationID string
    Logger        *logrus.Logger
}

// NewNotificationService creates a new notification service instance
func NewNotificationService(cfg NotificationConfig) (*NotificationService, error) {
    client, err := notifications_client.Connect(cfg.DaprAddress, cfg.Topic, cfg.PubSubName)
    if err != nil {
        return nil, fmt.Errorf("failed to connect to notifications: %w", err)
    }
    
    return &NotificationService{
        client:        client,
        applicationID: cfg.ApplicationID,
        logger:        cfg.Logger,
    }, nil
}

// SendAlert sends an alert notification
func (ns *NotificationService) SendAlert(ctx context.Context, opts AlertOptions) error {
    event := &notifications_pb.Event{
        Type:          opts.EventType,
        Subtype:       opts.Subtype,
        AccountId:     opts.AccountID,
        ApplicationId: ns.applicationID,
        Location:      "k8s-cluster",
        Severity:      opts.Severity,
        Status:        opts.Status,
        ProductName:   opts.ProductName,
        Metadata:      opts.Metadata,
        OccurredTime:  timestamppb.Now(),
    }
    
    if err := ns.client.Publish(ctx, event); err != nil {
        ns.logger.WithError(err).WithFields(logrus.Fields{
            "subtype":    opts.Subtype,
            "account_id": opts.AccountID,
        }).Error("failed to publish notification")
        return err
    }
    
    ns.logger.WithFields(logrus.Fields{
        "subtype":    opts.Subtype,
        "account_id": opts.AccountID,
    }).Info("notification published successfully")
    
    return nil
}

// AlertOptions contains options for sending an alert
type AlertOptions struct {
    EventType   notifications_pb.Event_Type
    Subtype     string
    AccountID   string
    Severity    string
    Status      notifications_pb.Event_Status
    ProductName string
    Metadata    map[string]string
}

// Example usage
func ExampleUsage() {
    cfg := NotificationConfig{
        DaprAddress:   "localhost:50001",
        Topic:         "events",
        PubSubName:    "notifications-pubsub",
        ApplicationID: "my-service",
        Logger:        logrus.New(),
    }
    
    ns, err := NewNotificationService(cfg)
    if err != nil {
        panic(err)
    }
    
    ctx := context.Background()
    
    // Send a raised alert
    err = ns.SendAlert(ctx, AlertOptions{
        EventType:   notifications_pb.Event_SYSTEM,
        Subtype:     "service-degraded",
        AccountID:   "account-123",
        Severity:    "high",
        Status:      notifications_pb.Event_RAISED,
        ProductName: "my-product",
        Metadata: map[string]string{
            "subject":      "Service Degradation Alert",
            "body":         "The service is experiencing degraded performance.",
            "service_name": "MyService",
            "click_link":   "/dashboard/alerts",
        },
    })
    if err != nil {
        panic(err)
    }
    
    // Later, clear the alert
    err = ns.SendAlert(ctx, AlertOptions{
        EventType:   notifications_pb.Event_SYSTEM,
        Subtype:     "service-degraded",
        AccountID:   "account-123",
        Severity:    "high",
        Status:      notifications_pb.Event_CLEARED,
        ProductName: "my-product",
        Metadata: map[string]string{
            "subject":      "Service Recovered",
            "body":         "The service has recovered to normal operation.",
            "service_name": "MyService",
        },
    })
}
```

---

## Deployment Requirements

### Prerequisites

1. **Dapr Sidecar**: Required for pub/sub messaging
2. **PostgreSQL**: Required for Config, Events Processor, and Mailbox services
3. **AWS SES**: Required for email delivery (PostOffice)
4. **Prometheus AlertManager**: Required for Dispatcher
5. **Prometheus Operator**: Required for Thresholding service

### Helm Dependencies

Add these dependencies to your service's Helm chart:

```yaml
# values.yaml
dapr:
  enabled: true
  appId: "your-service"
  
notifications:
  enabled: true
  events:
    processor:
      host: "notification-events-processor"
      port: 9090
  config:
    host: "notification-config"
    port: 9090
```

### Required Services Deployment Order

1. **PostgreSQL** (shared or separate instances)
2. **atlas.notifications.config** (with CRDs)
3. **atlas.notifications.events.processor** (with CRDs)
4. **atlas.notifications.dispatcher**
5. **atlas.notifications.mailbox**
6. **atlas.notifications.postoffice**
7. **atlas.notifications.sender**
8. **atlas.notifications.thresholding** (optional, for metric alerts)
9. **atlas.notifications.alert.adapter** (optional, for legacy support)

### Database Migrations

Each service requires database migrations. Run them in order:
```bash
# Config Service
make migrate-up -C atlas.notifications.config

# Events Processor
make migrate-up -C atlas.notifications.events.processor

# Mailbox
make migrate-up -C atlas.notifications.mailbox
```

---

## API Reference

### Config Service Endpoints

| Endpoint | Method | Description |
|----------|--------|-------------|
| `/atlas-notifications-config/v1/subscribe` | POST | Create subscription |
| `/atlas-notifications-config/v1/subscriptions` | GET | List subscriptions |
| `/atlas-notifications-config/v1/channel` | GET/POST | Manage channels |
| `/atlas-notifications-config/v1/template` | GET/POST | Manage templates |
| `/atlas-notifications-config/v1/trigger` | GET | List triggers |
| `/atlas-notifications-config/v1/filter` | GET/POST | Manage filters |
| `/atlas-notifications-config/v1/channel_types` | GET | List channel types |

### Mailbox Service Endpoints

| Endpoint | Method | Description |
|----------|--------|-------------|
| `/atlas-notifications-mailbox/v1/account_alerts` | POST | Create alert |
| `/atlas-notifications-mailbox/v1/account_alerts` | GET | List alerts |
| `/atlas-notifications-mailbox/v1/account_alerts/{id}` | GET | Get alert |
| `/atlas-notifications-mailbox/v1/account_alerts/{id}` | PATCH | Update alert state |

---

## Checklist for Integration

### Required Changes

- [ ] Add `atlas.notifications.events.client` dependency to `go.mod`
- [ ] Initialize notification client in your service startup
- [ ] Define your event subtypes (coordinate with notifications team)
- [ ] Create notification templates for each channel type
- [ ] Add Dapr sidecar configuration to Helm chart
- [ ] Configure environment variables for pub/sub
- [ ] Implement event publishing in your business logic
- [ ] Add metrics for notification publishing (recommended)
- [ ] Test with different account IDs and event types
- [ ] Document your event subtypes and metadata keys

### Optional Changes

- [ ] Implement CLEARED events for recoverable alerts
- [ ] Add custom fingerprinting for deduplication
- [ ] Configure summarization rules for high-frequency events
- [ ] Create custom webhook channels
- [ ] Set up PagerDuty integration
- [ ] Implement user-facing filter management

---

## Support & Resources

- **Design Document**: [Atlas Notifications Design](https://docs.google.com/document/d/1GmBX212Itei_VLQqwtNUQSbYPRZE1_qV1TJvTGZQvZY/edit)
- **Config Service Schema**: [Data Schema](https://docs.google.com/document/d/15Ke64iZd9G7RIoZ705Of0RUJ4WnDC1ukL0chhsVHQhM/edit)
- **Events Client Repository**: `github.com/Infoblox-CTO/atlas.notifications.events.client`

---

*Last Updated: February 2026*

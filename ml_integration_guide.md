# ML Integration Capabilities Guide

This guide provides comprehensive documentation for the ML integration capabilities added to the Super Mario Bros PPO Dashboard. These capabilities enhance the dashboard with connections to external ML platforms, notification systems, version control, cloud storage, and reporting features.

## Table of Contents

1. [Overview](#overview)
2. [Installation](#installation)
3. [Export Capabilities](#export-capabilities)
4. [Notification Systems](#notification-systems)
5. [Version Control](#version-control)
6. [Cloud Storage](#cloud-storage)
7. [Reporting](#reporting)
8. [API Reference](#api-reference)
9. [Troubleshooting](#troubleshooting)

## Overview

The ML integration capabilities enhance the Super Mario Bros PPO Dashboard with:

- **Export functionality** to popular ML platforms like TensorBoard and Weights & Biases
- **Notification systems** for training milestones and events
- **Version control** for models and configurations
- **Cloud storage** integration with AWS S3, Google Cloud Storage, and Azure Blob Storage
- **Reporting capabilities** for generating comprehensive training reports

These features are designed to integrate seamlessly with the existing dashboard UI, adding new functionality without disrupting the user experience.

## Installation

### Dependencies

The ML integration capabilities require the following Python packages:

```
flask>=2.0.0
numpy>=1.19.0
pandas>=1.3.0
matplotlib>=3.4.0
jinja2>=3.0.0
```

For optional integrations:
```
tensorboardX>=2.4.0  # For TensorBoard export
wandb>=0.12.0        # For Weights & Biases export
boto3>=1.18.0        # For AWS S3 integration
google-cloud-storage>=1.42.0  # For Google Cloud Storage integration
azure-storage-blob>=12.8.0  # For Azure Blob Storage integration
gitpython>=3.1.0     # For Git integration
```

### Setup

1. Install the required dependencies:

```bash
pip install -r requirements.txt
```

2. Integrate with the main application:

```python
from ml_integration_app import integrate_with_main_app

# Your Flask app
app = Flask(__name__)

# Add ML integration capabilities
app = integrate_with_main_app(app)
```

3. Create the necessary directories:

```bash
mkdir -p data/exports
mkdir -p data/reports
mkdir -p data/models
mkdir -p data/configs
```

## Export Capabilities

The export capabilities allow users to export training data, metrics, and models to various ML platforms and file formats.

### Supported Export Formats

- **TensorBoard**: Export metrics for visualization in TensorBoard
- **Weights & Biases**: Export metrics and models to W&B for tracking and visualization
- **CSV**: Export metrics and data to CSV files
- **JSON**: Export metrics and data to JSON files

### Using the Export API

```python
# Export metrics to TensorBoard
response = requests.post('/api/export/tensorboard', json={
    'metrics': {
        'episode_reward': 150.5,
        'episode_length': 250,
        'loss': 0.05
    },
    'step': 10000
})

# Export metrics to W&B
response = requests.post('/api/export/wandb', json={
    'metrics': {
        'episode_reward': 150.5,
        'episode_length': 250,
        'loss': 0.05
    },
    'step': 10000,
    'config': {
        'project': 'mario-ppo',
        'run_name': 'test-run'
    }
})
```

### Export Configuration

Each export format can be configured with specific settings:

- **TensorBoard**: Log directory
- **Weights & Biases**: Project name, API key, run name
- **CSV/JSON**: Export directory, auto-export settings

## Notification Systems

The notification system provides alerts for important training milestones and events.

### Supported Notification Channels

- **Browser**: In-browser notifications
- **Email**: Email notifications via SMTP
- **Slack**: Notifications to Slack channels via webhooks
- **Webhook**: Generic webhook notifications to any HTTP endpoint

### Notification Rules

Notification rules define when notifications are sent based on conditions:

```json
{
    "channel": "email",
    "condition": {
        "metric": "episode_reward",
        "operator": "gt",
        "value": 500
    },
    "subject": "High Reward Achieved",
    "message": "Episode reward exceeded {value}! Current value: {metric_value}",
    "enabled": true
}
```

Supported operators:
- `gt`: Greater than
- `lt`: Less than
- `gte`: Greater than or equal
- `lte`: Less than or equal
- `eq`: Equal to
- `neq`: Not equal to

### Using the Notification API

```python
# Configure email notifications
response = requests.post('/api/notifications/channels/email/configure', json={
    'smtp_server': 'smtp.gmail.com',
    'smtp_port': 587,
    'username': 'your-email@gmail.com',
    'password': 'your-password',
    'sender': 'your-email@gmail.com',
    'recipients': ['recipient@example.com']
})

# Add a notification rule
response = requests.post('/api/notifications/rules', json={
    'channel': 'email',
    'condition': {
        'metric': 'episode_reward',
        'operator': 'gt',
        'value': 500
    },
    'subject': 'High Reward Achieved',
    'message': 'Episode reward exceeded {value}! Current value: {metric_value}',
    'enabled': true
})

# Process a milestone
response = requests.post('/api/notifications/process', json={
    'episode_reward': 550,
    'episode_length': 250,
    'loss': 0.05
})
```

## Version Control

The version control system manages different versions of models and configurations.

### Model Versioning

Models are versioned with unique IDs and metadata:

```json
{
    "version_id": "model_20220530123456",
    "filename": "ppo_mario_v1.zip",
    "original_path": "/tmp/models/ppo_mario_v1.zip",
    "versioned_path": "data/models/model_20220530123456/ppo_mario_v1.zip",
    "hash": "a1b2c3d4e5f6...",
    "timestamp": "2022-05-30T12:34:56.789Z",
    "metadata": {
        "architecture": "PPO",
        "parameters": 1000000,
        "training_episodes": 1000
    }
}
```

### Configuration Versioning

Configurations are versioned similarly to models:

```json
{
    "version_id": "config_20220530123456",
    "filename": "ppo_config_v1.json",
    "original_path": "/tmp/configs/ppo_config_v1.json",
    "versioned_path": "data/configs/config_20220530123456/ppo_config_v1.json",
    "hash": "a1b2c3d4e5f6...",
    "timestamp": "2022-05-30T12:34:56.789Z",
    "metadata": {
        "learning_rate": 0.0003,
        "gamma": 0.99,
        "n_steps": 2048
    }
}
```

### Using the Version Control API

```python
# Register a model
with open('/path/to/model.zip', 'rb') as f:
    files = {'file': f}
    data = {'metadata': json.dumps({
        'architecture': 'PPO',
        'parameters': 1000000,
        'training_episodes': 1000
    })}
    response = requests.post('/api/version-control/register/model', files=files, data=data)

# List model versions
response = requests.get('/api/version-control/models')

# Download a model
response = requests.get('/api/version-control/models/model_123456/download')

# Create an archive of versions
response = requests.post('/api/version-control/archive', json={
    'version_ids': ['model_123456', 'config_789012'],
    'include_models': True,
    'include_configs': True
})
```

## Cloud Storage

The cloud storage integration allows syncing models, data, and reports with cloud storage providers.

### Supported Providers

- **AWS S3**: Amazon Simple Storage Service
- **Google Cloud Storage**: Google Cloud's object storage
- **Azure Blob Storage**: Microsoft Azure's object storage

### Sync Tasks

Sync tasks define how files are synchronized between the local filesystem and cloud storage:

```json
{
    "id": "task_123456",
    "provider": "s3",
    "direction": "upload",
    "local_path": "data/models",
    "bucket": "mario-ppo-models",
    "remote_path": "models"
}
```

### Using the Cloud Storage API

```python
# Configure AWS S3
response = requests.post('/api/cloud-storage/providers/s3/configure', json={
    'aws_access_key': 'your-access-key',
    'aws_secret_key': 'your-secret-key',
    'region_name': 'us-west-2'
})

# Connect to AWS S3
response = requests.post('/api/cloud-storage/providers/s3/connect')

# Add a sync task
response = requests.post('/api/cloud-storage/sync/tasks', json={
    'task_id': 'backup_models',
    'config': {
        'provider': 's3',
        'direction': 'upload',
        'local_path': 'data/models',
        'bucket': 'mario-ppo-models',
        'remote_path': 'models'
    }
})

# Run a sync task
response = requests.post('/api/cloud-storage/sync/tasks/backup_models/run')
```

## Reporting

The reporting capabilities allow generating comprehensive reports from training data.

### Report Templates

Reports are generated from templates that can be customized:

- **HTML**: HTML reports with styling and interactive elements
- **Markdown**: Simple markdown reports
- **JSON**: Structured JSON data
- **CSV**: Tabular data in CSV format

### Using the Reporting API

```python
# Generate a report
response = requests.post('/api/reporting/generate', json={
    'template': 'training_summary_html',
    'data': {
        'metrics': {
            'episode_reward': 324.5,
            'average_reward': 210.3,
            'loss': 0.0324
        },
        'progress': {
            'total_episodes': 100,
            'completed_episodes': 42,
            'completion_percentage': 42
        },
        'performance': {
            'win_rate': 0.65,
            'average_score': 3240,
            'completion_time': 230
        },
        'hyperparameters': {
            'learning_rate': 0.0003,
            'gamma': 0.99,
            'clip_range': 0.2,
            'n_steps': 2048
        }
    },
    'format': 'html'
})

# Get a report
response = requests.get('/api/reporting/reports/report_123456?format=raw')

# Generate figures
response = requests.post('/api/reporting/figures', json={
    'training_history': [
        {'episode': 1, 'episode_reward': 100},
        {'episode': 2, 'episode_reward': 150},
        {'episode': 3, 'episode_reward': 200}
    ]
})
```

## API Reference

### Export API

- `GET /api/export/history`: Get export history
- `POST /api/export/tensorboard`: Export metrics to TensorBoard
- `POST /api/export/wandb`: Export metrics to Weights & Biases
- `POST /api/export/csv`: Export metrics to CSV file
- `POST /api/export/json`: Export metrics to JSON file
- `POST /api/export/bundle`: Create an export bundle
- `GET /api/export/download/<filename>`: Download an exported file

### Notification API

- `GET /api/notifications/channels`: List available notification channels
- `POST /api/notifications/channels/<channel>/configure`: Configure a notification channel
- `GET /api/notifications/rules`: List notification rules
- `POST /api/notifications/rules`: Add a notification rule
- `GET /api/notifications/rules/<rule_id>`: Get a specific notification rule
- `PUT /api/notifications/rules/<rule_id>`: Update a notification rule
- `DELETE /api/notifications/rules/<rule_id>`: Delete a notification rule
- `POST /api/notifications/process`: Process data against notification rules
- `GET /api/notifications/history`: Get notification history
- `POST /api/notifications/test`: Test a notification
- `GET /api/notifications/sse`: Server-Sent Events endpoint for browser notifications

### Version Control API

- `GET /api/version-control/models`: List all model versions
- `GET /api/version-control/models/<version_id>`: Get a specific model version
- `GET /api/version-control/models/<version_id>/download`: Download a specific model version
- `DELETE /api/version-control/models/<version_id>`: Delete a specific model version
- `GET /api/version-control/configs`: List all configuration versions
- `GET /api/version-control/configs/<version_id>`: Get a specific configuration version
- `GET /api/version-control/configs/<version_id>/download`: Download a specific configuration version
- `DELETE /api/version-control/configs/<version_id>`: Delete a specific configuration version
- `POST /api/version-control/register/model`: Register a model file for versioning
- `POST /api/version-control/register/config`: Register a configuration file for versioning
- `POST /api/version-control/archive`: Create an archive of specific versions
- `GET /api/version-control/archive/download/<filename>`: Download a specific archive
- `POST /api/version-control/import`: Import versions from an archive

### Cloud Storage API

- `GET /api/cloud-storage/providers`: List available cloud storage providers
- `POST /api/cloud-storage/providers/<provider>/connect`: Connect to a cloud storage provider
- `POST /api/cloud-storage/providers/<provider>/disconnect`: Disconnect from a cloud storage provider
- `POST /api/cloud-storage/providers/<provider>/configure`: Configure a cloud storage provider
- `GET /api/cloud-storage/providers/<provider>/buckets`: List buckets/containers for a provider
- `GET /api/cloud-storage/providers/<provider>/buckets/<bucket>/objects`: List objects in a bucket/container
- `POST /api/cloud-storage/providers/<provider>/upload`: Upload a file to cloud storage
- `GET /api/cloud-storage/providers/<provider>/download`: Download a file from cloud storage
- `POST /api/cloud-storage/providers/<provider>/delete`: Delete a file from cloud storage
- `GET /api/cloud-storage/providers/<provider>/metadata`: Get metadata for a file in cloud storage
- `GET /api/cloud-storage/providers/<provider>/url`: Get a pre-signed download URL for a file in cloud storage
- `GET /api/cloud-storage/sync/tasks`: List synchronization tasks
- `POST /api/cloud-storage/sync/tasks`: Add a synchronization task
- `GET /api/cloud-storage/sync/tasks/<task_id>`: Get a synchronization task
- `PUT /api/cloud-storage/sync/tasks/<task_id>`: Update a synchronization task
- `DELETE /api/cloud-storage/sync/tasks/<task_id>`: Delete a synchronization task
- `POST /api/cloud-storage/sync/tasks/<task_id>/run`: Run a synchronization task

### Reporting API

- `GET /api/reporting/templates`: List available report templates
- `GET /api/reporting/templates/<template_name>`: Get a specific report template
- `POST /api/reporting/templates`: Save a custom report template
- `DELETE /api/reporting/templates/<template_name>`: Delete a custom report template
- `POST /api/reporting/generate`: Generate a report from a template and data
- `GET /api/reporting/reports/<report_id>`: Get a generated report
- `DELETE /api/reporting/reports/<report_id>`: Delete a generated report
- `GET /api/reporting/reports`: Get report generation history
- `POST /api/reporting/figures`: Generate figures from data
- `POST /api/reporting/bundle`: Create a bundle of reports
- `GET /api/reporting/bundle/<filename>`: Download a report bundle

## Troubleshooting

### Common Issues

**Issue**: TensorBoard export fails with "TensorBoard not available"
**Solution**: Install the TensorBoardX package: `pip install tensorboardX`

**Issue**: W&B export fails with "W&B not available"
**Solution**: Install the W&B package: `pip install wandb`

**Issue**: Cloud storage connection fails
**Solution**: Check that the credentials provided are correct and have the necessary permissions

**Issue**: SSE notifications not working in browser
**Solution**: Make sure the browser supports Server-Sent Events and check that there are no CORS issues

**Issue**: Version control fails with "Git integration not available"
**Solution**: Install the GitPython package: `pip install gitpython`

### Logs

The ML integration components log information to the standard Flask application log. Check the logs for error messages and debugging information.

### Support

For additional support, please open an issue on the GitHub repository or contact the development team.
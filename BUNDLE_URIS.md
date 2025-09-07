# Bundle URIs Support

## Overview

This Docker image now supports Bundle URIs - a GitLab/Gitaly feature that can significantly speed up clones and fetches for users with poor network connections to the GitLab server, and reduce server load by serving bundles from CDNs or other storage providers.

Bundle URIs are built into the Git protocol and allow clients to pre-download bundles from alternative locations before fetching the remaining objects from the main GitLab server.

## Benefits

- Speed up clones and fetches for users with poor network connection to the GitLab server
- Reduce load on servers running CI/CD jobs
- Bundles can be stored on CDNs making them available worldwide
- Reduce network traffic to the main GitLab instance

## Configuration

Bundle URIs support can be enabled through environment variables.

### Basic Configuration

To enable Bundle URIs, set the following environment variable:

```bash
GITALY_BUNDLE_URI_ENABLED=true
GITALY_BUNDLE_URI_GO_CLOUD_URL=<storage_url>
```

### Storage Providers

The feature supports several cloud storage providers:

#### AWS S3

```yaml
version: '3'
services:
  gitlab:
    image: sameersbn/gitlab:latest
    environment:
      - GITALY_BUNDLE_URI_ENABLED=true
      - GITALY_BUNDLE_URI_GO_CLOUD_URL=s3://my-bundle-bucket?region=us-west-1
      - GITALY_BUNDLE_URI_AWS_ACCESS_KEY_ID=your_access_key
      - GITALY_BUNDLE_URI_AWS_SECRET_ACCESS_KEY=your_secret_key
      - GITALY_BUNDLE_URI_AWS_REGION=us-west-1
```

#### S3-Compatible (MinIO, etc.)

```yaml
version: '3'
services:
  gitlab:
    image: sameersbn/gitlab:latest
    environment:
      - GITALY_BUNDLE_URI_ENABLED=true
      - GITALY_BUNDLE_URI_GO_CLOUD_URL=s3://my-bundle-bucket?region=minio&endpoint=my.minio.local:8080&disable_https=true&use_path_style=true
      - GITALY_BUNDLE_URI_AWS_ACCESS_KEY_ID=minio_access_key
      - GITALY_BUNDLE_URI_AWS_SECRET_ACCESS_KEY=minio_secret_key
```

#### Google Cloud Storage

```yaml
version: '3'
services:
  gitlab:
    image: sameersbn/gitlab:latest
    environment:
      - GITALY_BUNDLE_URI_ENABLED=true
      - GITALY_BUNDLE_URI_GO_CLOUD_URL=gs://my-bundle-bucket
      - GITALY_BUNDLE_URI_GOOGLE_APPLICATION_CREDENTIALS=/path/to/service.json
    volumes:
      - /host/path/to/gcs-key.json:/path/to/service.json
```

#### Azure Blob Storage

```yaml
version: '3'
services:
  gitlab:
    image: sameersbn/gitlab:latest
    environment:
      - GITALY_BUNDLE_URI_ENABLED=true
      - GITALY_BUNDLE_URI_GO_CLOUD_URL=azblob://my-bundle-container
      - GITALY_BUNDLE_URI_AZURE_STORAGE_ACCOUNT=mystorageaccount
      - GITALY_BUNDLE_URI_AZURE_STORAGE_KEY=storage_key
      # Or use SAS token instead of storage key:
      # - GITALY_BUNDLE_URI_AZURE_STORAGE_SAS_TOKEN=sas_token
```

### Environment Variables

| Variable | Description | Default | Required |
|----------|-------------|---------|----------|
| `GITALY_BUNDLE_URI_ENABLED` | Enable Bundle URIs support | `false` | Yes |
| `GITALY_BUNDLE_URI_GO_CLOUD_URL` | Storage URL (s3://, gs://, azblob://) | - | Yes if enabled |
| `GITALY_BUNDLE_URI_AWS_ACCESS_KEY_ID` | AWS Access Key ID | Inherits from `AWS_ACCESS_KEY_ID` | For S3 |
| `GITALY_BUNDLE_URI_AWS_SECRET_ACCESS_KEY` | AWS Secret Access Key | Inherits from `AWS_SECRET_ACCESS_KEY` | For S3 |
| `GITALY_BUNDLE_URI_AWS_REGION` | AWS Region | Inherits from `AWS_REGION` | For S3 |
| `GITALY_BUNDLE_URI_GOOGLE_APPLICATION_CREDENTIALS` | Path to GCP service account JSON | - | For GCS |
| `GITALY_BUNDLE_URI_AZURE_STORAGE_ACCOUNT` | Azure Storage Account name | - | For Azure |
| `GITALY_BUNDLE_URI_AZURE_STORAGE_KEY` | Azure Storage Key | - | For Azure (or use SAS token) |
| `GITALY_BUNDLE_URI_AZURE_STORAGE_SAS_TOKEN` | Azure SAS Token | - | For Azure (alternative to storage key) |

## Bundle Generation

Once configured, bundles can be generated in two ways:

### Manual Generation

You can manually generate bundles using the GitLab Rails console:

```bash
docker exec -it gitlab_container_name sudo -HEu git bash -l
cd /home/git/gitlab
bundle exec rails console production

# Generate bundle for a specific project
project = Project.find_by_full_path('group/project')
Gitaly::Repository.new(project.repository).create_bundle_uri
```

### Automatic Generation (if supported by GitLab version)

Gitaly can automatically generate bundles when it detects frequent clone activity for repositories.

## Client Configuration

### For CI/CD Jobs

To use Bundle URIs in CI/CD jobs, ensure your GitLab Runner uses:
- Git version 2.49.0 or later
- GitLab Runner helper version 18.0 or later
- Feature flag `FF_USE_GIT_NATIVE_CLONE: "true"` in your `.gitlab-ci.yml`

### For Local Development

To use Bundle URIs when cloning locally:

```bash
git config --global transfer.bundleuri true
git clone https://gitlab.example.com/group/project.git
```

## Security

Bundle URIs use signed URLs for access control, providing time-limited access to the bundles. The signing is handled automatically by Gitaly.

## Troubleshooting

1. **Check if Bundle URIs are enabled**:
   ```bash
   docker exec gitlab_container grep -A 5 "\[bundle_uri\]" /home/git/gitaly/config.toml
   ```

2. **Check Gitaly logs**:
   ```bash
   docker exec gitlab_container tail -f /home/git/gitlab/log/gitaly/current
   ```

3. **Verify storage connectivity**:
   Ensure your GitLab instance can reach the configured storage provider and has proper authentication.

## Performance Benefits

Bundle URIs can provide significant performance improvements:
- Reduced server load during clone operations
- Faster clones for users with poor connectivity to the main server
- Better scalability for popular repositories

In testing, Bundle URIs have shown reductions from ~5M objects downloaded from the main server to ~1.3M objects, with the majority pre-loaded from the bundle storage.

## Documentation

For more details about Bundle URIs, see the official GitLab documentation:
https://docs.gitlab.com/administration/gitaly/bundle_uris/

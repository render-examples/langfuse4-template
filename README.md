# Langfuse v4 on Render

This repository contains a Render Blueprint for a self-hosted Langfuse v4 deployment.

## What the Blueprint creates

The Blueprint creates a new Render project named `langfuse-v4`. It adds a protected `production` environment with these resources:

- A Langfuse v4 web service
- A Langfuse v4 background worker
- A managed Render Postgres database
- A managed Render Key Value instance
- A private ClickHouse service with a persistent disk
- A MinIO service with a persistent disk

The environment blocks private network traffic from other Render environments. MinIO has a public URL because Langfuse sends presigned media URLs to browsers and SDK clients. MinIO still requires signed requests.

## Deploy

1. Open the [Render Dashboard](https://dashboard.render.com/).
2. Select **New > Blueprint**.
3. Select the `renderinc/langfuse-blueprint` repository.
4. Review the instance types and disk sizes.
5. Enter the requested secret values:
   - `ENCRYPTION_KEY`: Enter 64 hexadecimal characters. Generate a value with `openssl rand -hex 32`.
   - `CLICKHOUSE_PASSWORD`: Enter a long alphanumeric value.
   - `MINIO_ROOT_PASSWORD`: Enter at least 8 characters.
6. Apply the Blueprint.

Render generates `SALT` and `NEXTAUTH_SECRET` during the first Blueprint sync.

## Region

The Blueprint does not set a region. Render uses its default region for each new resource.

To use another region, add the same `region` value to every service and database before the first Blueprint sync. Render cannot move an existing resource to another region.

## Capacity and cost

The Blueprint uses production-oriented instance types and two 100 GB persistent disks. Review the resource cost before you apply it.

ClickHouse and MinIO use one instance each. Render services with a persistent disk cannot scale to more than one instance. This Blueprint does not provide high availability for these two services.

## Validate

Use Render CLI v2.7.0 or later:

```sh
render blueprints validate render.yaml
```

The Langfuse web and worker services use the `langfuse/langfuse:4` and `langfuse/langfuse-worker:4` image tags.

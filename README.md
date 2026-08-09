# Langfuse v4 profile on Render

This repository contains a Render Blueprint for a small self-hosted Langfuse v4 deployment. Do not use this profile for production workloads.

## What the Blueprint creates

The Blueprint creates a new Render project named `langfuse`. It adds an unprotected `production` environment with these resources:

- A `standard` Langfuse v4 web service
- A `standard` Langfuse v4 background worker
- A managed PostgreSQL 18 `basic-1gb` database with 5 GB of storage
- A managed `starter` Render Key Value instance
- A private `pro` ClickHouse service with a 10 GB disk
- A public `starter` MinIO service with a 10 GB disk

The environment blocks private network traffic from other Render environments. MinIO has a public URL because Langfuse sends presigned media URLs to browsers and SDK clients. MinIO still requires signed requests.

## Deploy

1. Open the [Render Dashboard](https://dashboard.render.com/).
2. Select **New > Blueprint**.
3. Select the `renderinc/langfuse-blueprint` repository.
4. Review the instance types and disk sizes.
5. Apply the Blueprint.

Render generates `SALT`, `ENCRYPTION_KEY`, `NEXTAUTH_SECRET`, `CLICKHOUSE_PASSWORD`, and `MINIO_ROOT_PASSWORD` during the first Blueprint sync. The Langfuse startup commands convert `ENCRYPTION_KEY` to the required 64-character hexadecimal format.

## Region

The Blueprint does not set a region. Render uses its default region for each new resource.

To use another region, add the same `region` value to every service and database before the first Blueprint sync. Render cannot move an existing resource to another region.

## Compatibility with Langfuse Docker Compose

The Blueprint follows the [Langfuse v4 Docker Compose configuration](https://github.com/langfuse/langfuse/blob/main/docker-compose.yml). It uses the same Langfuse v4 and ClickHouse 25.12 image versions, service ports, credentials, buckets, and object storage paths. It uses PostgreSQL 18 instead of the upstream default PostgreSQL 17.

The Blueprint has these Render-specific changes:

- Managed Render PostgreSQL 18 and Key Value replace the PostgreSQL and Redis containers.
- The MinIO Docker command combines the upstream Compose entrypoint and command. It creates the `langfuse` bucket directory before it starts MinIO.
- The Langfuse startup commands convert the generated encryption key to the required format, then run the image entrypoints. The web startup command also URL-encodes the generated ClickHouse password during migrations.
- `CLICKHOUSE_DB` is not set. Setting it starts an image initialization path that cannot stop its temporary ClickHouse process on Render.
- Render service references and private networking replace Docker Compose service discovery and port bindings.

## Capacity and cost

This profile uses small instance types and disks for functional tests. The web and worker each have a 1.5 GB Node.js heap limit. Review the resource cost before you apply the Blueprint.

ClickHouse and MinIO use one instance each. Render services with a persistent disk cannot scale to more than one instance. This Blueprint does not provide high availability for these two services.

## Validate

Use Render CLI v2.7.0 or later:

```sh
render blueprints validate render.yaml
```

The Langfuse web and worker services use the `langfuse/langfuse:4` and `langfuse/langfuse-worker:4` image tags.

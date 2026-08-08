# Langfuse v4 test profile on Render

This repository contains a Render Blueprint for a small self-hosted Langfuse v4 test deployment. Do not use this profile for production workloads.

## What the Blueprint creates

The Blueprint creates a new Render project named `langfuse-v4-test`. It adds an unprotected `test` environment with these resources:

- A `standard` Langfuse v4 web service
- A `standard` Langfuse v4 background worker
- A managed Postgres 17 `basic-1gb` database with 5 GB of storage
- A managed `starter` Render Key Value instance
- A private `pro` ClickHouse service with a 10 GB disk
- A public `starter` MinIO service with a 10 GB disk

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

## Compatibility with Langfuse Docker Compose

The Blueprint follows the [Langfuse v4 Docker Compose configuration](https://github.com/langfuse/langfuse/blob/main/docker-compose.yml). It uses the same Langfuse v4 and ClickHouse 25.12 image versions, Postgres 17, service ports, credentials, buckets, and object storage paths.

The Blueprint has these Render-specific changes:

- Managed Render Postgres and Key Value replace the Postgres and Redis containers.
- The MinIO Docker command combines the upstream Compose entrypoint and command. It creates the `langfuse` bucket directory before it starts MinIO.
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

# Langfuse v4 on Render

This repository contains a Render Blueprint for a self-hosted Langfuse v4 deployment. It creates separate Render resources for the Langfuse web service, worker, and data stores. This architecture can support production workloads when you select plans, storage, scaling, backups, and recovery controls that match your workload.

[![Deploy to Render](https://render.com/images/deploy-to-render-button.svg)](https://render.com/deploy-template/api/github/start?template_repo=langfuse4-template)

## What the Blueprint creates

The Blueprint creates a new Render project named `langfuse-4`. It adds an unprotected `production` environment with these resources:

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
3. Select the `render-examples/langfuse4-template` repository.
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
- The ClickHouse startup command disables unused system log tables that Langfuse does not read.
- `CLICKHOUSE_DB` is not set. Setting it starts an image initialization path that cannot stop its temporary ClickHouse process on Render.
- Render service references and private networking replace Docker Compose service discovery and port bindings.

## Production sizing

The Blueprint defaults are a cost-conscious starting point. Increase them before you send production traffic. The following baseline maps the [Langfuse production sizing guidance](https://langfuse.com/self-hosting/configuration/scaling) to [Render instance types](https://render.com/docs/compute-plans):

| Resource | Blueprint default | Suggested production baseline |
| --- | --- | --- |
| Langfuse web | `standard` (1 CPU, 2 GB RAM) | `pro` (2 CPU, 4 GB RAM) |
| Langfuse worker | `standard` (1 CPU, 2 GB RAM) | `pro` (2 CPU, 4 GB RAM) |
| PostgreSQL | `basic-1gb`, 5 GB storage | `pro-8gb` (2 CPU, 8 GB RAM) with at least 20 GB storage |
| Render Key Value | `starter` (256 MB RAM) | `pro` (5 GB RAM) |
| ClickHouse | `pro` (2 CPU, 4 GB RAM), 10 GB disk | `pro_plus` (4 CPU, 8 GB RAM) with at least a 100 GB disk |
| MinIO | `starter` (0.5 CPU, 512 MB RAM), 10 GB disk | `pro` (2 CPU, 4 GB RAM) with at least a 100 GB disk |

These values are an initial production baseline, not a capacity guarantee. Event volume, retention, media size, query patterns, and concurrent users determine the resources that you need. Review current [Render pricing](https://render.com/pricing) before you apply the Blueprint.

When you move the web and worker services to `pro`, increase `NODE_OPTIONS` from `--max-old-space-size=1536` to `--max-old-space-size=3072`. This setting leaves memory for the Node.js runtime outside the JavaScript heap.

Monitor CPU, memory, disk use, request latency, worker queue depth, and database connections after deployment. Add web or worker instances when sustained CPU use exceeds 50 percent. Increase ClickHouse memory first when analytical queries become slow. Increase disks before they reach 80 percent use because Render disks cannot be reduced after expansion.

## Production availability and recovery

The production baseline above provides production-scale capacity, but it does not make every component highly available:

- Render Postgres `pro-8gb` supports high availability and point-in-time recovery. Confirm that both features meet your recovery objectives.
- Render Key Value uses `noeviction` and journal-and-snapshot persistence. Monitor available memory because queue writes fail when the instance is full.
- ClickHouse and MinIO each use one Render service with one persistent disk. Render services with a persistent disk cannot scale to multiple instances, and deploys require downtime.
- For higher availability, use a replicated or managed ClickHouse deployment and managed S3-compatible object storage instead of the single-instance ClickHouse and MinIO services in this Blueprint.

Define retention policies, backup schedules, restore tests, monitoring alerts, and recovery objectives before you send production traffic.

## Validate

Use Render CLI v2.7.0 or later:

```sh
render blueprints validate render.yaml
```

The Langfuse web and worker services use the `langfuse/langfuse:4` and `langfuse/langfuse-worker:4` image tags.

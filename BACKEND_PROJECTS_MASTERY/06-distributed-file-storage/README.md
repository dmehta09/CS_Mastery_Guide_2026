# Project 06: Distributed File Storage System

## Difficulty Level: Advanced
**Estimated Time**: 3-4 weeks per language

## What You're Building

A scalable object storage system like Amazon S3, Google Cloud Storage, or MinIO. Store and retrieve files reliably across multiple servers with features like chunking, replication, and CDN integration.

## Key Concepts

- Object storage vs block storage
- Content-addressable storage
- Chunking and reassembly
- Replication strategies
- Erasure coding
- CDN integration
- Presigned URLs
- Multipart uploads

## Architecture

```
Client → API Gateway → Storage Service → Storage Nodes (Multiple)
                              ↓
                        Metadata DB
                              ↓
                         CDN Cache
```

## Core Features

1. **Upload**: Support files up to 5GB
2. **Download**: Stream files efficiently
3. **Chunking**: Split large files into chunks
4. **Replication**: Store 3 copies across nodes
5. **Versioning**: Keep file versions
6. **CDN**: Cache popular files
7. **Access Control**: Presigned URLs, ACLs
8. **Metadata**: Store file metadata

## Database Schema

```sql
CREATE TABLE objects (
    id UUID PRIMARY KEY,
    bucket VARCHAR(255) NOT NULL,
    key VARCHAR(1024) NOT NULL,
    size_bytes BIGINT NOT NULL,
    content_type VARCHAR(100),
    etag VARCHAR(64) NOT NULL,
    version_id UUID,
    storage_class VARCHAR(20),
    metadata JSONB,
    created_at TIMESTAMP NOT NULL,
    UNIQUE(bucket, key, version_id)
);

CREATE TABLE object_chunks (
    id BIGSERIAL PRIMARY KEY,
    object_id UUID REFERENCES objects(id),
    chunk_index INT NOT NULL,
    storage_node VARCHAR(255) NOT NULL,
    chunk_hash VARCHAR(64) NOT NULL,
    size_bytes INT NOT NULL,
    UNIQUE(object_id, chunk_index)
);
```

## Success Criteria

✅ Handle 10GB+ files
✅ 99.99% durability
✅ Sub-second uploads/downloads
✅ CDN integration
✅ Automatic replication

---

**Next**: Project 07 (GraphQL Federation)

# Day 1 — Why PostgreSQL?

These are rough session notes from the opening module of Track 1. The goal of this session is to answer the first question every participant brings into the room: *"Why should I invest time in PostgreSQL specifically?"* Three answers were discussed — open source, ecosystem variety, and extensibility — each backed with real industry evidence.

---

## Table of Contents

- [1. Open Source](#1-open-source)
- [2. Wide Variety of Options](#2-wide-variety-of-options)
- [3. Highly Extensible](#3-highly-extensible)

---

## 1. Open Source

PostgreSQL is fully open source — the entire source code is publicly available and has been for over 25 years. This matters for two reasons: there are no licensing costs, and you can read exactly what the database is doing internally.

**Reading the source:**

The PostgreSQL source is browsable online via Doxygen — a documentation generator that produces cross-referenced HTML from the C source code. When you want to understand what happens inside `VACUUM` or how the buffer manager works, you go here:

```
https://doxygen.postgresql.org
```

This is not just an academic exercise. When a production issue doesn't match the documentation, the source is the ground truth.

---

## 2. Wide Variety of Options

"PostgreSQL" is not one product — it is an ecosystem. You choose a distribution and a support model based on your budget and requirements.

### Distributions

| Vendor | Product |
|---|---|
| Community | PostgreSQL (Community Version) |
| EDB (EnterpriseDB) | PPAS (EDB Postgres Advanced Server) |
| Amazon | Aurora PostgreSQL |
| Postgres Pro | Postgres Enterprise |
| Microsoft | Azure Database for PostgreSQL |
| Google | Cloud SQL for PostgreSQL |

PPAS adds Oracle compatibility features on top of the community version — useful for Oracle migration projects. Aurora, Cloud SQL, and Azure Database are managed services where the cloud vendor handles patching, backups, and HA.

### Budget Matrix

The distribution you pick is not just a technical decision — it is a budget decision. Real-world options:

| Budget (approx/month) | Stack |
|---|---|
| $100 | EDB support contract + PPAS |
| $50 | EDB support contract + Community Version |
| $30 | Community Version + third-party support (e.g., 2ndQuadrant, Percona) |
| $0 | Community Version + in-house DBA |

The $0 option is viable and extremely common — PostgreSQL's community support, documentation, and mailing lists are strong enough that many organisations run it without a paid support contract. This is only possible because of the open source model.

---

## 3. Highly Extensible

The strongest signal that PostgreSQL is the right bet is not documentation or benchmarks — it is where the industry is moving money. Several high-profile decisions in recent years tell the story:

**Amazon moved off Oracle onto PostgreSQL**
Amazon's internal systems, including large parts of Amazon.com and AWS infrastructure, migrated away from Oracle to Aurora PostgreSQL. This was not a cost-cutting exercise — it was a strategic platform decision by one of the largest database users on the planet.

**PostgreSQL runs on Oracle OCI**
Oracle's own cloud infrastructure (OCI) offers managed PostgreSQL as a first-class service. The vendor most threatened by PostgreSQL's growth chose to host it rather than fight it.

**Crunchy Data acquired by Snowflake**
Crunchy Data, one of the leading PostgreSQL managed service and support companies, was acquired by Snowflake. Snowflake paid for PostgreSQL expertise — a direct signal of where enterprise data infrastructure is heading.

**VMware Postgres**
VMware shipped their own PostgreSQL distribution (Tanzu Postgres) for Kubernetes-native deployments. Enterprise infrastructure vendors don't build distributions around databases they don't believe will dominate.

**YugabyteDB**
YugabyteDB is a distributed SQL database built to be wire-compatible with PostgreSQL. The founders chose PostgreSQL compatibility — not MySQL, not Oracle — as their target, because that is where they expected the market to be.

---

## The One-Line Answer

> PostgreSQL is the default choice for new production database deployments in 2025 — not because it is free, but because the entire industry has converged on it.

The open source model removed the licensing barrier. The ecosystem gave enterprises a migration path from Oracle. The extensibility model made it the base layer for a dozen commercial products. That combination is why we are here.

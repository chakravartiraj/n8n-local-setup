# Enterprise Patterns

## 🎯 Learning Objectives
By the end of this module, you will:
- Design and implement multi-tenant PostgreSQL architectures
- Build scalable data warehousing solutions
- Integrate PostgreSQL with enterprise systems and data pipelines
- Apply advanced integration and scaling strategies
- Ensure compliance, security, and reliability at scale

## 📚 Table of Contents
1. [Multi-Tenant Architectures](#multi-tenant-architectures)
2. [Data Warehousing](#data-warehousing)
3. [Integration Patterns](#integration-patterns)
4. [Scaling Strategies](#scaling-strategies)
5. [Compliance and Security](#compliance-and-security)
6. [Enterprise Automation](#enterprise-automation)
7. [Hands-On Exercises](#hands-on-exercises)
8. [Best Practices](#best-practices)
9. [Next Steps](#next-steps)

---

## Multi-Tenant Architectures

### Approaches to Multi-Tenancy
- **Shared Database, Shared Schema**: All tenants share tables, separated by tenant_id
- **Shared Database, Separate Schemas**: Each tenant has its own schema
- **Separate Databases**: Each tenant has a dedicated database

#### Example: Shared Schema
```sql
CREATE TABLE customers (
    id SERIAL PRIMARY KEY,
    tenant_id INT NOT NULL,
    name TEXT,
    email TEXT
);

CREATE INDEX idx_customers_tenant_id ON customers(tenant_id);
```

#### Example: Separate Schemas
```sql
CREATE SCHEMA tenant_123;
CREATE TABLE tenant_123.customers (
    id SERIAL PRIMARY KEY,
    name TEXT,
    email TEXT
);
```

#### Row-Level Security
```sql
ALTER TABLE customers ENABLE ROW LEVEL SECURITY;
CREATE POLICY tenant_isolation ON customers
    USING (tenant_id = current_setting('my.tenant_id')::int);
```

---

## Data Warehousing

### Data Warehouse Design
- Star and snowflake schemas
- Fact and dimension tables
- Partitioning for performance

#### Example: Partitioned Fact Table
```sql
CREATE TABLE sales (
    id SERIAL PRIMARY KEY,
    sale_date DATE NOT NULL,
    amount NUMERIC,
    region TEXT
) PARTITION BY RANGE (sale_date);

CREATE TABLE sales_2025 PARTITION OF sales
    FOR VALUES FROM ('2025-01-01') TO ('2026-01-01');
```

#### ETL Pipelines
- Use tools like Apache Airflow, dbt, or custom scripts
- Bulk loading with `COPY` and parallel imports

```sql
COPY sales FROM '/data/sales_2025.csv' DELIMITER ',' CSV HEADER;
```

---

## Integration Patterns

### Integrating PostgreSQL with Enterprise Systems
- REST APIs using PostgREST or custom backends
- Change Data Capture (CDC) with Debezium or logical replication
- Event-driven architectures with NOTIFY/LISTEN

#### Example: Logical Replication
```sql
-- On source
CREATE PUBLICATION my_pub FOR TABLE customers, sales;

-- On target
CREATE SUBSCRIPTION my_sub CONNECTION 'host=source_host dbname=source_db user=replicator password=secret' PUBLICATION my_pub;
```

#### Example: Event Notification
```sql
NOTIFY new_order, 'Order 123 created';
LISTEN new_order;
```

---

## Scaling Strategies

### Vertical and Horizontal Scaling
- Vertical: Increase resources (CPU, RAM, SSD)
- Horizontal: Sharding, partitioning, read replicas

#### Example: Sharding with Citus
```sql
-- Install Citus extension
CREATE EXTENSION citus;

-- Distribute table
SELECT create_distributed_table('customers', 'tenant_id');
```

#### Read Scaling
- Use read replicas for reporting and analytics
- Load balancing with PgBouncer, HAProxy

---

## Compliance and Security

### Enterprise Security Practices
- Encryption at rest and in transit
- Auditing and logging
- Role-based access control
- Data masking and anonymization

#### Example: Auditing
```sql
CREATE EXTENSION pgaudit;
ALTER SYSTEM SET pgaudit.log = 'all';
SELECT pg_reload_conf();
```

#### Example: Data Masking
```sql
CREATE FUNCTION mask_email(email TEXT) RETURNS TEXT AS $$
BEGIN
    RETURN regexp_replace(email, '(.{2}).*@', '\1***@');
END;
$$ LANGUAGE plpgsql;
```

---

## Enterprise Automation

### Infrastructure as Code (IaC)
- Use Terraform, Ansible, or Kubernetes operators
- Automated provisioning and scaling

#### Example: Terraform PostgreSQL Provider
```hcl
resource "postgresql_database" "app_db" {
  name = "app_db"
}

resource "postgresql_role" "app_user" {
  name     = "app_user"
  password = "secure_password"
}
```

---

## Hands-On Exercises

### Exercise 1: Multi-Tenant Setup
**Objective**: Implement shared schema and row-level security

**Steps**:
1. Create a shared customers table
2. Enable row-level security
3. Test tenant isolation

### Exercise 2: Data Warehouse Partitioning
**Objective**: Create and query partitioned tables

**Steps**:
1. Create partitioned sales table
2. Load sample data
3. Query partitions for performance

### Exercise 3: Logical Replication
**Objective**: Set up publication and subscription

**Steps**:
1. Create publication on source
2. Create subscription on target
3. Verify data replication

---

## Best Practices

### Enterprise Patterns Best Practices
- Use schemas and row-level security for tenant isolation
- Partition large tables for performance
- Automate infrastructure and deployments
- Monitor and audit all database activity
- Plan for compliance and regulatory requirements
- Test scaling and failover regularly

---

## Next Steps

### Advanced Topics to Explore
1. **Global Database Architectures** - Multi-region, cross-cloud
2. **Machine Learning Integration** - In-database analytics
3. **Streaming Data Pipelines** - Kafka, Flink, etc.
4. **Advanced Security** - Data loss prevention, advanced encryption
5. **Cloud-Native PostgreSQL** - Managed services, Kubernetes operators

### Recommended Reading
- [PostgreSQL Partitioning Documentation](https://www.postgresql.org/docs/current/partitioning.html)
- [Citus Sharding Guide](https://docs.citusdata.com/en/latest/)
- [pgaudit Documentation](https://pgaudit.org/)
- [Terraform PostgreSQL Provider](https://registry.terraform.io/providers/cyrilgdn/postgresql/latest/docs)

### Community Resources
- PostgreSQL Enterprise Mailing List
- Citus Community Slack
- Data Engineering Forums

---

## Summary

In this module, you've learned to:
- ✅ Design multi-tenant and data warehouse architectures
- ✅ Integrate PostgreSQL with enterprise systems
- ✅ Apply advanced scaling and automation strategies
- ✅ Ensure compliance and security at scale

You're now equipped to architect and operate PostgreSQL for demanding enterprise environments.

**Congratulations! You've completed the PostgreSQL comprehensive learning path.**

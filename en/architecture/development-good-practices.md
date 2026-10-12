---
title: Development Good Practices
---
## Introduction

This guide outlines general best practices for building modular, secure, and maintainable systems. Each principle is accompanied by a description of its general importance and an example from a real-world modular ETL pipeline.

### **Modularity**

Modularity promotes the separation of concerns by dividing a system into self-contained, independent components. Each module handles a single responsibility and interacts with others through clear interfaces.

**Advantages:**

- Improves readability and maintainability
- Simplifies debugging and testing by isolating logic

**ETL Example:**

The ETL pipeline is divided into modules such as Extract, Transform, Load, Schemas, Logger, and Scheduler. Each has its own submodules, class definitions, decorators, and configurations. This enables focused development and testing.

### **Encapsulation**

Encapsulation hides internal implementation details and exposes only necessary interfaces. This ensures modularity and protects the internal state of objects from unintended external modifications.

**Advantages:**

- Promotes cleaner, more robust code
- Reduces coupling between components

**ETL Example:**

Each ETL submodule uses abstract base classes from Python's `abc` module to define public behaviors. This enforces encapsulation and allows internal logic to evolve without affecting other modules.

### **Single Source of Truth**

Maintaining a centralized, authoritative location for shared rules, schemas, or configurations helps ensure consistency and reduces the risk of duplication or errors.

**Advantages:**

- Increases data reliability and consistency
- Eases onboarding by making rules and formats discoverable

**ETL Example:**

The Schemas module defines validation models using Pydantic, Pandera, and SQLAlchemy. This approach provides a consistent schema through every phase of the ETL process.

### **Standalone Configurations**

Configuration should be separated from code. Each module should declare and validate its own configuration using strongly typed models.

**Advantages:**

- Enhances flexibility and adaptability across environments
- Reduces hardcoded values and improves traceability

**ETL Example:**

Each module has a local configuration submodule using Pydantic for validation. A global configuration orchestrator aggregates and manages system-wide settings.

### **Unified Logging & Error Reporting**

**General Concept:**

Centralized logging and error reporting ensures visibility into system behavior. A consistent format allows integration with monitoring and alerting tools.

**Advantages:**

- Improves observability and incident response
- Enables data-driven decisions through audit logs

**ETL Example:**

A unified logger module supports structured logging and integrates with external monitoring systems. Errors and warnings trigger email/webhook alerts in real time.

### **Deployability**

A deployable system includes reproducible build and deployment mechanisms. Using containerization and CI/CD pipelines simplifies scaling and rollout.

**Advantages:**

- Reduces deployment errors and speeds up delivery
- Standardizes environments across teams

**ETL Example:**

The ETL system is packaged and deployed using GitLab CI/CD pipelines with Docker. Development and production environments are consistent and easily reproducible.

### **Security**

Security must be embedded throughout the application lifecycle. This includes secure data handling, authentication, and auditability.

**Advantages:**

- Protects sensitive data and maintains trust
- Enables compliance with industry standards

**ETL Example:**

Security is implemented at every stage: OAuth2 in Extraction, hashing and masking in Transformation, and encryption in Load. Secrets are injected dynamically, and the CI/CD process includes static analysis and container hardening.
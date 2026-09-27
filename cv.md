# Tin La

832-475-4456 — tinsila1@gmail.com — linkedin.com/in/tintinator — github.com/tintinator

## Summary

Backend software engineer with 5+ years across storage infrastructure, distributed systems, and microservices, most recently owning high-stakes reliability features (emergency shutdown, zero-downtime updates) for a storage management service at Samsung Semiconductor.

## Experience

### Samsung Semiconductor — Backend Software Engineer, Storage Management Service
San Jose, CA · Aug 2022 – Dec 2025

- Owned the emergency shutdown feature, which stabilizes the storage system during node failures near quorum threshold. Served as the primary on-call contact for emergency shutdown incidents.
  - Converted a customer-flagged data-loss risk into a testable solution within six months by designing a shutdown flow coordinated across three teams.
  - Prevented 8x16 cluster data loss by developing an automatic emergency shutdown feature across internal microservices and expanding real-time node health monitoring.
  - Automated emergency cluster recovery by detecting and re-adding nodes coming back online, performing data rebalancing, and restoring the cluster to a healthy state.
- Increased cluster fault tolerance and eliminated a startup bottleneck by detecting and enabling incomplete drive recoveries to resume post startup. Validated with 93 days of I/O uptime under resilience testing.
- Owned the development and maintenance of the Non-Disruptive Update (NDU) functionality, serving as the primary on-call contact for all NDU-related incidents.
  - Achieved zero-downtime cluster updates by building an NDU workflow cycling cluster nodes offline, updating, and reintegrating them. Resulted in 326 days of continuous uptime for a partner customer.
  - Hardened NDU reliability by implementing version-check utilities, script execution, and bundle validation for update fallback, cleanup, and post-upgrade flows.
- Enforced resource-level access by writing privilege-based authorization checks across resource-sensitive REST APIs.
- Reduced engineer onboarding time by 50% by condensing dev environment setup, package dependencies, and CI/CD tooling instructions into an updated Confluence guide in an effort to improve PR consistency.

### Cox Automotive — Software Engineer, Customer API Platform
Austin, TX · Apr 2021 – Aug 2022

- Contributed to an app modernization effort by converting monorepo backend to a microservice architecture and migrating existing backend schemas to new DynamoDB tables.
- Automated test data provisioning and cleanup and increased unit test coverage by 40%.
- Streamlined the provider-facing upload experience by implementing auth protocols and creating reusable Terraform to provision LaunchDarkly feature flags.
- Expanded service observability by creating New Relic monitors and configuring app StatusPage and PagerDuty.
- Created a secret generation workflow to eliminate manual effort and reduce generation time by 50%.

### EPIC — Software Developer, Home Health and Hospice
Madison, WI · Aug 2019 – Aug 2020

- Modernized the existing home health scheduling native application to a web app using C# and TypeScript.
- Maintained a patient-records desktop application for medical professionals.
- Reduced physician signoff time around 15 minutes per form by redesigning medication order forms according to user feedback onsite.
- Prevented patient bereavement setting loss by writing auto-save functionality.

## Skills

- **Infra:** MongoDB, AWS (DynamoDB, Lambda, S3, SQS), ZeroMQ
- **Languages:** Java, C#, Python, React, JavaScript
- **Distributed Systems:** Consensus, Leader election, rolling updates
- **Testing:** JUnit, Mockito
- **AI Tools:** Claude Code — spec drafting, debugging, test generation
- **Tools:** Git, Linux, Bamboo, CI/CD

## Education

**The University of Texas at Austin** — B.S. Computer Science, GPA: 3.75/4.0
Austin, TX · Jun 2019

+++
title = 'Syncgen'
date = '2025-08-25'
draft = false
tech = 'Go, Datadog, PostgreSQL'
github = 'https://github.com/HasithDeAlwis/ha-syncgen'
description = 'Open-source tool for mid-sized companies to scaffold HA with YAML configs'
hasDetails = true
+++

# What
ha-syncgen is a declarative infrastructure tool designed to simplify the setup of high-availability PostgreSQL database clusters. The project takes a YAML-based configuration approach where users define their primary database server and replica nodes along with synchronization parameters, and the tool automatically generates all the necessary scripts, systemd services, and configuration files needed to maintain the cluster. The tool uses rsync for data synchronization between the primary and replica nodes, with configurable sync intervals and automatic health monitoring.
The generated infrastructure includes individual sync scripts for each replica, health check scripts that monitor the primary server's availability, systemd service and timer units for automated operation, and PostgreSQL configuration patches. 
The backbone of this project is WAL streaming, which is already built-in to PostgreSQL, allowing us to be certain that the data sent over is not corrupted, which could be the case if we use something like `rsync`.

When a primary server failure is detected, the tool can automatically promote a replica to become the new primary, ensuring minimal downtime. The project also includes observability integrations with popular monitoring solutions like Datadog and Prometheus, providing comprehensive status reporting through JSON aggregators.

# Why
The ha-syncgen project demonstrates several strong design principles that make it valuable for database administrators and DevOps teams. Its declarative YAML configuration approach significantly reduces the complexity typically associated with setting up PostgreSQL high-availability clusters, abstracting away the intricate details of rsync synchronization, systemd service management, and failover logic into a simple, human-readable configuration file. The tool's comprehensive script generation covers all aspects of cluster management, from routine synchronization to emergency failover scenarios, while the built-in observability support ensures teams can monitor cluster health through their preferred monitoring stack. Additionally, the cross-platform build support and systemd integration make it well-suited for modern Linux-based production environments, and the project's modular architecture allows for easy customization and extension of the generated infrastructure components.
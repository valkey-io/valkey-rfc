---
RFC: 31
Status: Proposed
---
# Valkey GUI RFC

## Abstract

The proposed feature is a Valkey GUI, a desktop application that brings observability capabilities to Valkey. It addresses critical observability and monitoring gaps identified through community analysis and customer research, particularly focusing on hot key detection, cluster visualization, and near real-time metrics collection. The GUI is a React and Node.js application, packaged with Electron to work as a cross-platform desktop application. The design includes a timeseries metrics service for historic observability trends, built into the desktop app by default, yet capable of running as a standalone service if needed. The application design presented in this RFC is not set in stone, and submitting this RFC serves as a mechanism to gather community feedback about design decisions and implementation details before finalizing the architecture and feature set.

## Motivation

Valkey deployments face four primary observability challenges that impact operational efficiency and incident response times: 

1. _Monitoring visibility gaps_: `INFO` command and `COMMANDLOG` visibility gaps limiting command-level insights for root cause analysis. These gaps extend troubleshooting time and reduce operational effectiveness during performance incidents.

2. _Limited cluster visibility_: lack of real-time slot-level performance metrics makes it very challenging to assess resource allocation or identify when rebalancing actions are needed especially during scaling operations.

3. _Hot key detection_: current hot key detection approach relies on the Least Frequently Used (LFU) eviction policy, which some deployments are unable to configure.

4. _Operational complexity:_ Manual monitoring of clusters demands substantial engineering effort.

The Valkey community has identified these challenges through engagement on [observability GitHub issues](https://github.com/valkey-io/valkey/issues?q=is%3Aissue%20state%3Aopen%20observability). [Issues #852](https://github.com/valkey-io/valkey/issues/852), [#2127](https://github.com/valkey-io/valkey/pull/2127), [#2128](https://github.com/valkey-io/valkey/issues/2128), and [#1880](https://github.com/valkey-io/valkey/issues/1880) in the Valkey repository demonstrate demand for slot level metrics and hot key configurability. Community analysis also shows preference for native observability features for enhanced observability with structured access to internal metrics, command statistics, and performance indicators. (i.e. [Issue #1167](https://github.com/valkey-io/valkey/issues/1167)).

Currently, no comprehensive observability solution exists that addresses these specific Valkey operational challenges in a unified interface. The Valkey community lacks a dedicated tool that combines near real-time monitoring, hot key detection, and cluster visualization tailored to Valkey's architecture and performance characteristics. This initiative fills the observability gap by providing a native solution designed for Valkey deployments, addressing the absence of purpose-built tooling in the current landscape.

## Design Considerations

The application's architecture reflects lessons learned from the community and customer research. It prioritizes solving top customer pain points with minimal overhead on monitored Valkey instances and comprehensive coverage from key-level to cluster-level insights. The community consistently favors built-in observability features and emphasizes minimal performance impact as a critical requirement, with [Issue #1167](https://github.com/valkey-io/valkey/issues/1167) discussions, including specific concerns about monitoring tools that impact production workloads. The GUI addresses this by polling Valkey at configurable intervals to collect time-series metrics.

### Architecture Overview

The application proposes a modular, three-component architecture designed for performance, maintainability, and observability:

**1. Frontend:** Built with React and Redux, the frontend runs as a cross-platform desktop application using Electron. This provides a native desktop experience while leveraging familiar web technologies and avoiding browser limitations.

The frontend operates within the Electron renderer process and connects to the backend via WebSockets to receive near real-time metric updates and request historical data. It receives specific metric types (memory, Central Processing Unit (CPU)) and displays them in tables and charts. When users select a time range, the frontend sends a query over WebSocket and receives filtered data for visualization.

**2. Backend for the Frontend:** A Node.js process that runs locally alongside the frontend and communicates with Valkey instances over WebSockets. It manages connection state, basic authentication, and command execution, returning responses for features like the integrated Command Line Interface (CLI). 

The backend runs in the Electron main process and hosts a WebSocket server that manages subscriptions, sends live updates to the frontend, and handles historical queries. It communicates with the Time Series Metrics service (See section #3 below) over a local or remote HTTP API, using basic authentication. For near real-time updates, the backend reads new entries from NDJSON files on disk, filters by metric type, and emits them to subscribed clients. For historical queries, it fetches rows from the API, filters by timestamp, buckets the results (e.g., per 5s or 1m), and sends them to the frontend.

**3. Timeseries Metrics Service:** This process supports historical trend analysis without introducing external dependencies. This component polls metrics from Valkey clusters or instances at configurable intervals, with default settings to be determined based on further testing and server load analysis. It writes data to efficient local storage, enabling time-series visualizations of CPU load, memory usage, hot keys, and hot slots. Future plans include the ability to store metrics in remote storage (such as S3 or a timeseries DB). While it runs locally by default, the service can also be deployed remotely.

The standalone metrics server connects directly to Valkey, polls metrics (like `MEMORY STATS`, `INFO CPU`, `COMMANDLOG`, `MONITOR`) at configurable intervals, and writes them to daily NDJSON files. It batches data to reduce I/O and includes retry and backoff mechanisms for resilience. Each API endpoint (`/memory`, `/cpu`, etc.) reads and returns recent NDJSON rows, supporting filtering by time in future extensions.

This architecture cleanly separates concerns: the metrics server collects and stores data, the backend handles communication and data shaping, and the frontend focuses on visualization and user interaction

### Data Collection

The primary goal of the metrics collection server is to provide comprehensive observability with minimal impact on server performance. The implementation will evolve iteratively as testing continues across the full application.

The metrics service collects data using the `INFO`, `MEMORY STATS`, and `COMMANDLOG GET` commands, directly addressing community concerns around monitoring blind spots and the need for historical trend visualization.

#### #1 Hot Key Detection With LFU Eviction Policy

For hot key detection in clusters that have the maxmemory policy set to LFU, the application will implement a two-phase approach that prioritizes performance:

1. **Hot Slot Identification:** It first analyzes slot-level metrics already exposed by Valkey, focusing on four key dimensions: key count, CPU usage, network bytes in, and network bytes out. This initial analysis identifies performance hotspots at the slot level without requiring expensive `SCAN` operations across the entire cluster.

2. **Scoped Key Scanning:** It then performs targeted key scans within only the identified hot slots. This reduces the cost of expensive scan operations while preserving accuracy in hot key detection. The scan logic uses undocumented but functional capabilities in Valkey to isolate hot keys without requiring a full cluster scan, helping avoid performance degradation in production environments.

#### #2 Hot Key Detection Without LFU Eviction Policy

For clusters that do not have the maxmemory policy set to LFU, the only practical workaround (without modifying the Valkey core) is to extract access counts from the output of the `MONITOR` command. Because `MONITOR` is resource-intensive, it will be run at configurable intervals to collect a representative sample of commands. This sample will provide sufficient data for parsing key access counts in the Timeseries Metrics Server. Additionally, users can specify the maximum number of commands to collect per interval to prevent out-of-memory (OOM) errors.

## Specification

### Metrics Dashboard and Analysis

The metrics dashboard is designed to close visibility gaps in Valkey’s `INFO`, `COMMANDLOG` outputs by using data collection with built-in retry logic to ensure completeness. For the MVP, we plan to surface 20–30 high-priority metrics based on customer feedback and operational relevance.

Feedback from the community, such as [[Issue #2065](https://github.com/valkey-io/valkey/issues/2065)], which requests I/O thread utilization in `INFO`, shows a clear need for deeper performance insights, especially in multi-threaded and containerized environments. The current Valkey `INFO` output lacks the detail needed for effective troubleshooting. The Valkey GUI addresses this by collecting more comprehensive metrics, displayed in a table format that can be quickly narrowed using search and filter controls. This makes it easier to work with large sets of metrics and find specific data points without digging through multiple views.

The dashboard also visualizes historical trends, made possible by the timeseries data collected by the metrics service. If the connected instance is part of a larger cluster, the metrics for all relevant nodes will be consolidated into a single place. 

### Cluster Topology Visualization

Cluster visualization shows the topology of nodes and slot distribution with intuitive visual representation. It highlights master-replica topology and overlays slot assignments and load distribution with heat maps.

Visual heat maps (post MVP) highlight CPU and memory usage across cluster nodes, leveraging slot-level metrics. The visualization updates in near real-time as new data becomes available, providing immediate feedback on cluster performance characteristics. Historical tracking enables trend analysis of patterns over time, allowing builders to identify recurring performance issues and plan capacity adjustments. An alert mechanism provides configurable notifications for access pattern anomalies, with thresholds that can be adjusted based on application requirements and cluster characteristics.

### Hot Key Detection and Visualization

Hot key detection addresses one of the most critical customer requirements through the two-phase approach that balances accuracy with performance impact concerns. The implementation leverages the slot-level memory metrics proof-of-concept from [PR #2127](https://github.com/valkey-io/valkey/pull/2127), which demonstrates that accurate slot-level memory attribution is technically feasible but requires careful implementation to avoid performance impact. The community discussion in [Issue #852](https://github.com/valkey-io/valkey/issues/852) specifically requests metrics that can identify which slots consume the most memory, directly validating the Valkey GUI's approach of identifying hot slots before scanning individual keys.

When a hot key detection cycle begins, the system first queries slot-level metrics across all cluster nodes to identify slots experiencing high activity across the four monitored dimensions. Slots exceeding configurable thresholds trigger the second phase, where targeted SCAN operations examine keys within those specific slots. This approach reduces the scanning scope from potentially millions of keys across the entire cluster, to hundreds or thousands of keys within identified hot slots.

We store hot key data in a structured format that includes timestamp, instance identifier, key name, access count, memory bytes, and slot identifier for analysis and correlation with cluster performance metrics. This data structure enables both near real-time monitoring and historical analysis, supporting capacity planning and performance optimization workflows.

### Community Engagement and Development Strategy

The Valkey GUI development follows transparent open source practices with public GitHub repositories and community-driven feature prioritization. The RFC process ensures alignment with Valkey community standards and facilitates collaborative development with existing observability developers. Regular community engagement occurs through weekly Monday community meetings, public issue tracking, and early alpha releases for community testing.

The development team maintains active participation in Valkey discussions and coordinates with other observability projects to avoid duplication of effort and enable potential collaboration. Customer validation occurs through direct engagement with customers facing observability issues, providing visibility and visualization capabilities that address operational requirements while managing expectations about server-side functionality changes.

## Implementation Timeline

### Foundation and core features for MVP (Dec 2025)

#### **Foundation Infrastructure:**

* Cross-platform (MacOS and Linux) Electron desktop application framework with React/Redux frontend
* Valkey connection management and cluster discovery
* Timeseries metrics service with configurable polling intervals

#### **Core Features:**

* Near real-time metrics dashboard with filtering capabilities
* Time-series and historical trend visualization
* Hot key detection
* Basic cluster topology visualization
* Integrated CLI for direct Valkey interaction
* Key browser (add, view, edit, delete key-value pairs)

### Future considerations

#### **Enhanced User Interface and Experience:**

* CLI autocomplete and syntax highlighting for improved usability
* Bulk operations support including multi-key deletion capabilities
* Generative artificial intelligence (AI) agent for contextual operational guidance and troubleshooting assistance
* Advanced database analysis with key type distribution and memory allocation insights

#### **Advanced Observability Features:**

* Publish/Subscribe (Pub/Sub) functionality monitoring and visualization
* Plugin architecture for custom integrations (Prometheus, Grafana, Datadog)
* Cluster heat maps with slot-level performance metrics

#### **Enterprise and Scale Features:**

* Remote metrics storage options (Amazon S3, timeseries databases)
* Machine learning-based anomaly detection and predictive capacity planning

#### **Platform Integrations:**

* Cloud platform compatibility (AWS ElastiCache, Azure Cache for Redis, Google Cloud Memorystore)
* Community plugin development framework with extensible architecture
* API integrations for existing monitoring infrastructure

#### **Production Readiness:**

* Performance optimization based on community feedback and telemetry
* Complete documentation, tutorials, and community onboarding materials

## Success Metrics

Community adoption metrics focus on GitHub engagement, issue resolution times under 48 hours for critical issues, and user satisfaction scores above 4.5 out of 5. Monthly active users and feature utilization/adoption rates provide insights into adoption patterns and feature effectiveness, while community contributions and plugin development indicate health and long-term sustainability.

## Conclusion

The Valkey GUI addresses critical observability gaps identified through community and customer analysis while providing a native solution for Valkey. The community validation through high-engagement GitHub issues, demonstrates that the GUI's feature set aligns with real operational needs identified by the Valkey user base.

The desktop application approach enables native performance and comprehensive system integration while avoiding the limitations of browser-based tools. The three-tier architecture with dedicated time-series metrics service provides the foundation for scalable observability without external infrastructure dependencies, addressing the community's preference for self-contained solutions that don't introduce additional operational complexity.

Community-driven development through the RFC process ensures alignment with real-world operational requirements while building a sustainable ecosystem around Valkey observability tooling. The transparent development model facilitates collaboration with existing projects and accelerates feature development through shared expertise, while the focus on performance-conscious design and minimal server impact addresses the community's primary concerns about monitoring tool overhead.

## References

* [Valkey Issue #1827: Feature to find bigkeys](https://github.com/valkey-io/valkey/issues/1827)
* [Valkey Issue #1167: OBSERVE command for enhanced observability](https://github.com/valkey-io/valkey/issues/1167)
* [Valkey Issue #1893: Introducing an independent admin thread](https://github.com/valkey-io/valkey/issues/1893)
* [Valkey Issue #2065: IO threads utilization INFO field](https://github.com/valkey-io/valkey/issues/2065)
* [Valkey Issue #1734: Enhance reliability of INFO command execution](https://github.com/valkey-io/valkey/issues/1734)
* [Valkey Issue #1929: Measure network in/out over cluster bus](https://github.com/valkey-io/valkey/issues/1929)
* [Valkey Issue #852: Introduce slot-level memory metrics](https://github.com/valkey-io/valkey/issues/852)
* [Valkey PR #2127: Slot-level memory metrics POC](https://github.com/valkey-io/valkey/pull/2127)
* [Valkey 9.1 Roadmap](https://github.com/orgs/valkey-io/projects/41)
* [Redis Issue #10472: Hot Key Detection](https://github.com/redis/redis/issues/10472)
* [ValkeyBloom Module RFC](https://github.com/valkey-io/valkey-rfc/blob/main/Bloom.md)


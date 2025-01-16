---
RFC: 17
Status: Proposed
---

# Valkey-Python-Integration-Test-Framework

## Abstract
In this RFC, we are proposing a light weight python testing framework that can be used by Modules and other packages to test against / with the valkey-server. Use cases include Modules in other code packages that need to write integeration tests. Example: ValkeyJSON, Valkey-Bloom

## Motivation

Currently, we can have software developed (e.g. Modules) for Valkey written in different languages - C, C++, Rust. For the purpose of integeration testing against Valkey Servers, without a common framework, each package will need to re-implement the same infrastructure. Having a single Integration Test Framework will solve this problem. A second motivation is simplicity and maintainability. Currently, tests in the valkey core are writtin in tcl files; for new contibutors onboarding, and to allow easy extending the integration test framework (by levegraing existing libraries), using a Language such as Python will help.

## Design Considerations

For minimal requirements, integration will need to support the following:
* ValkeyServer class - Start up / Tear down of a valkey-server
* ValeyClient class - Wrapper for creating a new client & connecting to the server, and supporting common client operations.
* Specifying custom engine versions
* Automatically building the valkey-server binary (pulling from valkey-io project)
* Specifying Custom Startup arguments
* Replication Support
* TLS
* Cluster Mode Enabled / Standalone
* Execution of multiple tests concurrently: Each test running on separate ports

## Interface

TODO

## Reference:
https://github.com/valkey-io/valkey-bloom/tree/unstable/tests/valkeytests
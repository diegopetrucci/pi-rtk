# Infrastructure Specification

## Purpose

Define how `pi-rtk` integrates with Pi as an installable package that
hooks Pi's shell execution surfaces without replacing the built-in
`bash` tool.

## Requirements

### Requirement: /rtk Slash Command Registration

The extension MUST register a `/rtk` slash command via Pi's
`registerCommand` API on load. The command MUST accept the
subcommand arguments `enable`, `disable`, and `status`, and MUST
support a bare invocation with no arguments.

#### Scenario: extension registers /rtk at load

- **WHEN** Pi loads the `pi-rtk` extension
- **THEN** `/rtk` MUST appear in Pi's slash command registry with a
  human-readable description
- **AND** invoking `/rtk` MUST route to the extension's handler

#### Scenario: argument completion lists valid subcommands

- **WHEN** the user types `/rtk ` and triggers argument completion
- **THEN** Pi MUST offer `enable`, `disable`, and `status` as the
  available completions

#### Scenario: unknown subcommand is rejected

- **WHEN** the user invokes `/rtk` with an argument that is not
  `enable`, `disable`, or `status`
- **THEN** the extension MUST surface a user-facing error message
  listing the valid subcommands
- **AND** the session toggle state MUST NOT change

### Requirement: Bash Tool Call Integration

The system MUST integrate with Pi's active `bash` tool through a
`tool_call` lifecycle handler. For agent-initiated `bash` tool calls,
`pi-rtk` MUST attempt rewrites by mutating `event.input.command` only
when session rewriting is enabled and a rewritten command is available.
The extension MUST NOT register or replace a `bash` tool
implementation.

#### Scenario: extension keeps Pi's active bash tool in place

- **GIVEN** the `pi-rtk` extension is loaded by Pi
- **WHEN** the agent invokes the `bash` tool
- **THEN** Pi MUST keep using its active `bash` tool implementation
- **AND** `pi-rtk` MUST participate by observing the `tool_call` event
  for that tool before execution

#### Scenario: rewrite success mutates bash input

- **GIVEN** the session rewrite toggle is `enabled`
- **WHEN** `pi-rtk` receives a `tool_call` event for the `bash` tool
- **AND** `rtkRewriteCommand` returns a rewritten command
- **THEN** `pi-rtk` MUST mutate `event.input.command` to the rewritten
  command
- **AND** Pi's active `bash` tool implementation MUST execute that
  mutated command

#### Scenario: no rewrite leaves original bash input unchanged

- **GIVEN** the session rewrite toggle is `disabled`, or
  `rtkRewriteCommand` does not return a rewritten command
- **WHEN** `pi-rtk` receives a `tool_call` event for the `bash` tool
- **THEN** `pi-rtk` MUST leave `event.input.command` unchanged
- **AND** Pi's active `bash` tool implementation MUST continue normal
  execution of the original command

### Requirement: Tool Call Load-Order Semantics

The system MUST rely on Pi's extension load order for `tool_call`
composition. Extensions that need the original bash command text MUST
run before `pi-rtk` mutates it, while rendering-focused bash tool
replacements can run after `pi-rtk` because `pi-rtk` does not claim the
`bash` tool slot.

#### Scenario: guard extension sees original command

- **GIVEN** a guard, permission, or policy extension also listens to
  `tool_call`
- **WHEN** that extension must inspect the original bash command text
  before `pi-rtk` rewrites it
- **THEN** that extension MUST load before `pi-rtk`
- **AND** it MUST receive the original `event.input.command` value

#### Scenario: rendering-focused bash replacement loads after pi-rtk

- **GIVEN** a rendering-focused bash tool replacement such as
  `quiet-tools`
- **WHEN** that extension loads after `pi-rtk`
- **THEN** `pi-rtk` MUST already have had the opportunity to mutate the
  command input during `tool_call`
- **AND** the later-loaded bash tool replacement MUST remain able to
  execute and render the resulting command

### Requirement: Pi Package Installability

The system MUST be installable and discoverable as a standard Pi package.

#### Scenario: Package metadata

- GIVEN a Pi package installation flow
- THEN the package metadata MUST identify the package as a Pi package
- AND the package metadata MUST declare the extension entry point required to load the package

### Requirement: Pi SDK Compatibility

The package MUST remain compatible with the supported Pi extension runtime and MUST require the documented Pi API surface needed for shell optimization.

#### Scenario: Runtime loading on supported Pi version

- **GIVEN** the package is installed in a Pi v0.60.0 or later environment
- **WHEN** Pi loads the package
- **THEN** the extension MUST load using Pi's exported `createLocalBashOperations()` helper
- **AND** the package MUST NOT require a bundled duplicate of Pi's local bash operations implementation

#### Scenario: Unsupported Pi version

- **GIVEN** a Pi environment earlier than v0.60.0
- **WHEN** a user attempts to use a release of `pi-rtk` that depends on Pi's exported `createLocalBashOperations()` helper
- **THEN** that Pi version MUST be considered unsupported by the package
- **AND** the package documentation and changelog MUST communicate the minimum supported Pi version as a breaking compatibility requirement

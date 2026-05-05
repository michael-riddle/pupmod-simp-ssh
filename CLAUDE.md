# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Module Overview

`pupmod-simp-ssh` is a SIMP Puppet module (version 6.20.1) that manages OpenSSH client and server configuration on RHEL-compatible systems (CentOS, RedHat, OracleLinux, Rocky, AlmaLinux — versions 7–9). It targets Puppet 7–8.

## Common Commands

```bash
# Install dependencies
bundle install

# List all available rake tasks
bundle exec rake -T

# Run unit tests
bundle exec rake spec

# Run a single spec file
bundle exec rspec spec/classes/server_spec.rb

# Run with debug output
PUPPET_DEBUG=true bundle exec rspec spec/classes/server_spec.rb

# Lint Puppet manifests
bundle exec rake lint

# Validate syntax
bundle exec rake syntax

# Lint metadata.json
bundle exec rake metadata_lint

# Run all acceptance tests (requires Vagrant)
bundle exec rake beaker:suites

# Run acceptance tests with FIPS mode
BEAKER_fips=yes bundle exec rake beaker:suites
```

## Architecture

### Class Hierarchy

- `ssh` (`manifests/init.pp`) — Entry point. Conditionally includes `ssh::client` and/or `ssh::server` via `enable_client`/`enable_server` parameters.
- `ssh::client` (`manifests/client.pp`) — Installs `openssh-clients`, manages `/etc/ssh/ssh_config` and `/etc/ssh/ssh_known_hosts`. Creates the default `Host *` entry via `ssh::client::host_config_entry` unless `add_default_entry: false`.
- `ssh::client::host_config_entry` (`manifests/client/host_config_entry.pp`) — Defined type managing per-host entries in `ssh_config`.
- `ssh::server` (`manifests/server.pp`) — Installs `openssh-server`, manages the `sshd` service, SSH host key permissions, and the `sshd` user/group.
- `ssh::server::conf` (`manifests/server/conf.pp`) — Manages all `sshd_config` settings. Declared `assert_private()` — **must be configured via Hiera or ENC (APL), not resource-style class declarations**.
- `ssh::authorized_keys` (`manifests/authorized_keys.pp`) — Loops over a Hiera hash to create `ssh_authorized_key` resources.

### Key Design Patterns

**Augeas providers**: All `sshd_config` and `ssh_config` entries are managed via the `augeasproviders_ssh` module's `sshd_config`/`ssh_config` resource types, not file templates. This means SSH config files are edited surgically, not replaced wholesale.

**`ssh::add_sshd_config` function** (`functions/add_sshd_config.pp`): Every `sshd_config` resource in `ssh::server::conf` is created through this helper, which skips the resource entirely if the key appears in `$remove_entries`. This lets callers declaratively purge settings.

**FIPS-aware cipher selection**: When `$fips = true` or the `fips_enabled` fact is set, `ssh::server::conf` and `ssh::client::host_config_entry` switch to FIPS-compliant cipher/MAC/kex algorithm lists defined in `ssh::server::params` and `ssh::client::params`. The `enable_fallback_ciphers` flag (default: `true`) appends FIPS-compatible fallback ciphers even in non-FIPS mode for interoperability.

**SIMP integration via `simp_options`**: Parameters for firewall, LDAP, PAM, FIPS, haveged, PKI, SSSD, and tcpwrappers all default to `simplib::lookup('simp_options::<feature>', ...)`. This means a SIMP-managed node can enable these features globally without per-module configuration.

**Compliance profiles**: `SIMP/compliance_profiles/` contains DISA STIG and NIST 800-53 rev4 enforcement data consumed by the `compliance_markup` module.

### Custom Facts

- `lib/facter/openssh_version.rb` — Reports the installed OpenSSH version; used in `ssh::server::conf` for version-gated directives (e.g., `UsePrivilegeSeparation` removed in OpenSSH ≥ 7.5).
- `lib/facter/ssh_host_keys.rb` — Returns a list of host key paths; iterated in `ssh::server` to enforce permissions.
- `lib/facter/timezone_file.rb` — Returns the timezone file path; used to populate `/var/empty/sshd/etc/localtime`.

### Adding or Modifying sshd_config Settings

- For **parameterized settings**: add a parameter to `ssh::server::conf` and call `ssh::add_sshd_config(...)`.
- For **arbitrary settings** at runtime: use the `ssh::server::conf::custom_entries` Hiera hash.
- For **`Match` blocks**: use the `sshd_config` Augeas resource type directly (not supported by `custom_entries`).

### Testing Patterns

Unit tests use `simp/rspec-puppet-facts` to iterate across multiple OS/architecture combinations. Hiera test fixtures live in `spec/fixtures/hieradata/`. Use `set_hieradata('filename_without_yaml')` in tests to load a custom hieradata file from that directory.

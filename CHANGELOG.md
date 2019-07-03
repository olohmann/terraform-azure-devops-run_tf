# Changelog

All notable changes to this project will be documented in this file.

The format is based on [Keep a Changelog](https://keepachangelog.com/en/1.0.0/),
and this project adheres to [Semantic Versioning](https://semver.org/spec/v2.0.0.html).

## [Unreleased]

### Added

### Removed

### Changed

----------------

## [1.3.0] - 2019-07-03

### Added
- Support download verfication via Hashicorp's PGP key.

### Removed

### Changed
- Toggle state firewall to default 'allow' during deployment. This resolves issues with flipping IP adresses in deployments behind a SNAT structure with flipping IP addresses. After finishing the deployment, the firewall will be updated to default deny.

## [1.2.1] - 2019-06-28

First release as standalone shell script.

### Added

### Removed

### Changed


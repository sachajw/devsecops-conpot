# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

Conpot is an ICS/SCADA honeypot designed to collect intelligence about adversaries targeting industrial control systems. It emulates multiple industrial protocols to appear as a legitimate ICS/SCADA device.

## Commands

### Running Conpot
```bash
# Production (requires config and template)
conpot -t <template_name> -c <config_file>

# Development/testing (uses testing.cfg)
conpot -t default -f

# List available templates
conpot
```

### Testing
```bash
# Run all tests
pytest

# Run specific test file
pytest conpot/tests/test_modbus_server.py

# Run specific test with verbose output
pytest -v conpot/tests/test_modbus_server.py::TestModbusServer::test_modbus_server

# Run with coverage
pytest --cov=conpot --cov-report=html

# Using tox (isolated test environment)
tox -e test
```

### Code Quality
```bash
# Format code
black .
```

### Docker
```bash
# Build
make build-docker

# Run (exposes multiple ICS protocol ports)
make run-docker
```

## Architecture

### Core Components (`conpot/core/`)
- **Databus** (`databus.py`): Key-value store for inter-component communication and shared state
- **Session Manager** (`session_manager.py`): Tracks attacker sessions across protocols
- **Virtual FS** (`virtual_fs.py`): Isolated filesystem per protocol for file-based attacks
- **Loggers** (`loggers/`): Multiple output formats (JSON, SQLite, TAXII, STIX, HPFriends)

### Protocol Implementations (`conpot/protocols/`)
Each protocol is self-contained with its own server class:
- `modbus/` - Modbus TCP (port 5020)
- `snmp/` - SNMP (port 16100/udp)
- `http/` - HTTP/SCADA web UI (port 8800)
- `s7comm/` - Siemens S7 (port 10201)
- `bacnet/` - BACnet (port 47808/udp)
- `enip/` - EtherNet/IP (port 44818)
- `IEC104/` - IEC 104 protocol
- `ipmi/` - IPMI (port 6230/udp)
- `ftp/` - FTP server (port 2121)
- `tftp/` - TFTP server (port 6969/udp)
- `kamstrup_*` - Kamstrup meter protocols
- `guardian_ast/` - Guardian AST protocol
- `proxy/` - TCP proxy for transparent forwarding

Protocol servers are registered in `conpot/protocols/__init__.py` via `name_mapping`.

### Templates (`conpot/templates/`)
Templates define device profiles using XML files validated against XSD schemas:
- Each template directory contains `template.xml` and protocol-specific XML configs
- Templates specify which protocols to enable, their ports, and device-specific values
- XSD schemas define valid structure for each protocol configuration

### Entry Point
`bin/conpot` is the CLI entry point that:
1. Parses arguments and loads configuration
2. Validates template XML against XSD
3. Initializes the databus with template values
4. Spawns greenlets for each enabled protocol server
5. Starts logging workers

## Key Patterns

### Adding a New Protocol
1. Create directory in `conpot/protocols/<protocol_name>/`
2. Implement server class with `start(host, port)` and `stop()` methods
3. Create `<protocol_name>.xsd` schema for template validation
4. Register in `conpot/protocols/__init__.py` `name_mapping` dict
5. Add test file in `conpot/tests/`

### Template Structure
```
templates/<template_name>/
  template.xml          # Core device metadata, databus values
  <protocol>/<protocol>.xml  # Protocol-specific config
  ssl/                   # SSL certificates (optional)
```

### Using the Databus
```python
from conpot.core import get_databus

databus = get_databus()
databus.set_value('key', value)
value = databus.get_value('key')
```

### Session Tracking
```python
from conpot.core import get_session

session = get_session('protocol_name', host, port)
session.add_event({'data': 'attack_data'})
```

## Concurrency Model

Uses gevent for cooperative multitasking. All protocol servers run as greenlets. The `gevent.monkey.patch_all()` call in `bin/conpot` patches stdlib for async I/O.

## Dependencies

Key dependencies with version constraints (see `requirements.txt`):
- `gevent>=1.0` - Async I/O framework
- `pysnmp==4.4.12` - SNMP implementation
- `scapy==2.4.5` - Packet manipulation
- `bacpypes==0.17.0` - BACnet protocol
- `cryptography==3.4.8` - Cryptographic operations
- `modbus-tk` - Modbus protocol

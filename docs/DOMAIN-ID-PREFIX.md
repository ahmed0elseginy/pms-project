# Domain ID Prefix System

## Overview
Domain IDs use a prefix-based numbering system for event-driven architecture. Each module is assigned a consistent prefix range.

## Prefix Allocation by Module

| Prefix Range | Module | Description |
|--------------|--------|-------------|
| 1xx | Library/Security | Security, JWT, Authentication |
| 10xx | Notification | Email, SMS, Push notifications |
| 20xx | User | User management, roles, auth |
| 30xx | Property | Properties, buildings, units |
| 40xx | Admin | Administration |
| 50xx | File | Document/File management |
| 60xx | Billing | (Reserved) |
| 70xx | (Reserved) | Future use |
| 80xx | Audit | Audit logging |
| 90xx | (Reserved) | Future use |

## Domain ID Format

```
{prefix}{sequence}
```

- **Prefix**: Module identifier (2 digits, except 1xx for security)
- **Sequence**: Sequential number per domain (01, 02, 03...)

### Examples
- User domain: `2001`, `2002`, `2003`, `2004`
- Property domain: `3001`, `3002`, `3003`
- Notification domain: `1001`, `1002`, `1003`

## Event ID Format

```
{domain_id}{sequence}
```

Events are children of domains. Format: `{domainId}{eventSequence}`

### Examples
- User domain (2001) events: `200101`, `200102`, `200103`...
- Property domain (3001) events: `30101`, `30102`...
- Unit domain (3002) events: `30201`, `30202`...

## Current Domain IDs

### Security (Library)
| ID | Domain | Destination |
|----|--------|-------------|
| 101 | SECURITY | - |
| 102 | LOGIN | - |
| 103 | JWT_TOKEN | - |

### Common (Library)
| ID | Domain | Destination |
|----|--------|-------------|
| 2001 | DOMAIN | - |
| 2002 | EVENT | - |

### User
| ID | Domain | Destination |
|----|--------|-------------|
| 2001 | USER | com.pms.user.user |
| 2002 | ACTIVATE_USER | com.pms.user.activate.user |
| 2003 | FORGET_USER_PASSWORD | com.pms.user.forget.user.password |
| 2004 | ROLE | com.pms.user.role |

### Property
| ID | Domain | Destination |
|----|--------|-------------|
| 1 | PROPERTY | com.pms.property |
| 2 | UNIT | com.pms.property.unit |
| 3 | BUILDING | com.pms.property.building |

### Admin
| ID | Domain | Destination |
|----|--------|-------------|
| 3001 | ADMIN | com.pms.admin.admin |

### File
| ID | Domain | Destination |
|----|--------|-------------|
| 5001 | DOCUMENT | com.pms.file.document |

### Notification
| ID | Domain | Destination |
|----|--------|-------------|
| 1001 | EMAIL | com.pms.user.user |
| 1002 | ACTIVATE_USER | com.pms.user.activate.user |
| 1003 | FORGET_USER_PASSWORD | com.pms.user.forget.user.password |

### Audit
| ID | Domain | Destination |
|----|--------|-------------|
| 8001 | AUDIT | com.pms.audit.audit |

## Usage in Event-Driven Architecture

Domains are used for:
1. **Message Queue Topics**: Each domain maps to an Artemis topic
2. **Event Classification**: Events belong to specific domains
3. **Audit Trail**: Domain + event = traceable actions

### Topic Naming Convention
```java
public static final String {DOMAIN}_TOPIC_NAME = "#{T(com.pms.{module}.model.enums.{Module}Domains).{DOMAIN}.destination()}";
```

### Example
```java
public static final String USER_TOPIC_NAME = "#{T(com.pms.user.model.enums.UserDomains).USER.destination()}";
```
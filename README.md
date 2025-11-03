# Splunk Technology Add-on for ufdbguard

## Description
This Technology Add-on (TA) provides field extractions and CIM mappings for ufdbguard URL filtering logs.

## Log Format
ufdbguard logs follow this space-delimited format:
```
MMM DD HH:MM:SS.mmm hostname ufdbguard[PID] YYYY-MM-DD HH:MM:SS [thread] DECISION user src_ip source_category category url method acl_match
```

## Installation

### Install the technical Add-on TA-ufdbguard

- Install the TA
- Restart Splunk
- Configure inputs.conf  
Configure the input to monitor your ufdbguard log directory.

## Extracted Fields

### Timestamp Fields
- `syslog_timestamp` - Syslog timestamp (MMM DD HH:MM:SS.mmm)
- `ufdb_timestamp` - ufdbguard internal timestamp (YYYY-MM-DD HH:MM:SS) - IMPORTANT this timestamp is use to set `_time` 

### Network and Host Fields
- `proxy_host` - Proxy server hostname
- `src_ip` - Source IP address (client)
- `src` - alias of `src_ip` (CIM compliance)
- `process_id` - ufdbguard process ID
- `thread_id` - Thread ID handling the request

### Decision Fields
- `decision` - Access decision (PASS, BLOCK, REDIRECT)
- `action` - Normalized action (allowed, blocked, unknown)
- `action_category` - Category from lookup table

### User and Authentication
- `user` - User (or `-` if not authenticated) (CIM compliance)

### Categorization Fields
- `source_category` - Source category (e.g., ipv4_src_all)
- `category` - URL category (searchengine, news, checked, etc.)
- `category_type` - Human-readable category name
- `category_description` - Description from lookup
- `risk_level` - Risk level from lookup (low, medium, high, critical)
- `acl_match` - ACL rule that matched (e.g., ipv4-subnet)

### Request Fields
- `url` - Requested URL (CIM compliance)
- `http_method` - HTTP method (GET, CONNECT, POST, etc.) (CIM compliance)

### URL Parsing Fields
- `http_scheme` - Protocol (http, https) if present
- `site` - Hostname or domain (e.g., www.google.com)
- `dest` - Alias for site (CIM compliance)
- `dest_port` - Port number if specified (e.g., 443, 80)
- `uri_path` - Path component (e.g., /search/results)
- `uri_query` - Query string after ? (e.g., q=test&page=1)
- `uri_fragment` - Fragment after # (e.g., section1)

### Calculated Fields
- `vendor_product` - Always set to "ufdbguard"

## Tags and Event Types

### Event Type
- `ufdbguard_web` - All ufdbguard events

### Tags
- `web` - Web traffic events
- `proxy` - Proxy events

Search with tags:
```spl
tag=web tag=proxy
| stats count by src_ip, url
```

## Usage Examples

### Search all blocked requests
```spl
index=proxy sourcetype=ufdbguard action=blocked
| stats count by src_ip, url, category
| sort -count
```

### Top users by request count
```spl
index=proxy sourcetype=ufdbguard user!="-"
| stats count by user, action
| sort -count
```

### Requests by category
```spl
index=proxy sourcetype=ufdbguard
| stats count by category, decision
| sort -count
```

### High-risk category access attempts
```spl
index=proxy sourcetype=ufdbguard
| lookup ufdbguard_categories.csv category OUTPUT risk_level
| where risk_level IN ("high", "critical")
| stats count by src_ip, category, url, decision
| sort -count
```

### Blocked requests timeline
```spl
index=proxy sourcetype=ufdbguard decision=BLOCK
| timechart count by category
```

### User activity summary
```spl
index=proxy sourcetype=ufdbguard user!="healthcheck" user!="-"
| stats count as total_requests, 
        dc(url) as unique_urls, 
        dc(category) as unique_categories,
        values(category) as categories
        by user, src_ip
| sort -total_requests
```

### Search engines accessed
```spl
index=proxy sourcetype=ufdbguard category=searchengine
| rex field=url "(?<domain>[^/:]+)"
| stats count by domain, user
| sort -count
```

### Exclude healthcheck events
```spl
index=proxy sourcetype=ufdbguard user!="healthcheck"
| stats count by category
```

### Access denied by source IP
```spl
index=proxy sourcetype=ufdbguard decision=BLOCK
| stats count as blocked_count, 
        values(category) as blocked_categories,
        values(url) as blocked_urls
        by src_ip
| sort -blocked_count
```

### CONNECT vs GET methods distribution
```spl
index=proxy sourcetype=ufdbguard
| stats count by method, decision
| chart count over method by decision
```

## Field Mapping Reference

| Field | CIM Field | Description |
|----------------|-----------|-------------|
| src_ip | src | Source IP address |
| user | user | Username |
| url | url | Requested URL |
| site | dest | Hostname/domain |
| method | http_method | HTTP method |
| decision | action | Access decision |

## Filtering Healthcheck Events

To exclude healthcheck events from indexing, uncomment this line in `props.conf`:
```ini
TRANSFORMS-exclude_healthcheck = ufdbguard_exclude_healthcheck
```

Or filter them at search time:
```spl
index=proxy sourcetype=ufdbguard user!="healthcheck"
```

## Notes

- The regex handles variable spacing between fields
- Usernames shown as `-` indicate unauthenticated requests
- CONNECT method typically indicates HTTPS traffic
- Categories can be customized in the `ufdbguard_categories.csv` lookup

## Troubleshooting

### No fields extracted
1. Verify sourcetype: `index=proxy | stats count by sourcetype`
2. Test regex: 
```spl
index=proxy sourcetype=ufdbguard 
| rex field=_raw "^(?<test_syslog>\w+\s+\d+\s+[\d:\.]+)\s+(?<test_host>\S+)\s+ufdbguard"
| table test_syslog, test_host
```

### Missing lookup fields
1. Verify lookup files exist:
```bash
ls -l $SPLUNK_HOME/etc/apps/TA-ufdbguard/default/lookups/
```

2. Test lookup:
```spl
| inputlookup ufdbguard_decisions.csv
```

### Time parsing issues
Verify TIME_FORMAT matches your logs. The default uses the syslog timestamp (MMM DD HH:MM:SS).

### Duplicate events
Check if logs are being ingested from multiple sources (file + syslog).

## Version
- Version: 1.0.0
- Compatible with: Splunk 8.x, 9.x
- Last Updated: November 2025
- Supports: ufdbguard URL filter logs

## Support
For issues or enhancements, please check:
- Splunk Answers: https://community.splunk.com
- ufdbguard documentation
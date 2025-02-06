---
title: Understanding Grafana's Data Source Proxy Whitelist Behavior
date: 2025-02-03
draft: false
tags:
  - finding
  - security
categories:
  - security
keywords:
  - finding
  - security
---

During my recent assessment of Grafana's data source functionality, I discovered an interesting behavior regarding how different endpoints handle the data source proxy whitelist. This highlights important security considerations for organizations using Grafana's data source proxy features.

## Overview

Grafana provides a data source proxy whitelist feature through the `data_source_proxy_whitelist` configuration setting. This setting is intended to restrict which targets data sources can connect to. However, the implementation reveals an interesting behavior that security teams should be aware of.

## Technical Deep Dive

The key finding involves two different endpoints that handle data source connections:

1. `/api/datasources/proxy/<uid>` - This endpoint goes through Grafana's data source proxy and respects the whitelist configuration.
2. `/api/datasources/uid/<uid>/resources/*` - This endpoint uses the `CallResource` API and intentionally bypasses the proxy.

This distinction is by design, as confirmed by the Grafana team. The `CallResource` API endpoint is documented and intended to operate independently of the data source proxy mechanism.

## Demonstration Environment

To demonstrate this behavior clearly, I created a test environment consisting of two services:

1. A Grafana instance with whitelist configuration
2. A Python test server that logs incoming requests

### Environment Setup

First, let's look at the Docker Compose configuration:

```yaml
version: '3.8'

networks:
  ssrf-test:
    ipam:
      driver: default
      config:
        - subnet: 172.20.0.0/16

services:
  grafana:
    image: grafana/grafana-enterprise:latest
    ports:
      - "3000:3000"
    volumes:
      - grafana-storage:/var/lib/grafana
      - ./grafana.ini:/etc/grafana/grafana.ini
    environment:
      - GF_SECURITY_ADMIN_PASSWORD=admin
      - GF_SECURITY_ADMIN_USER=admin
    networks:
      ssrf-test:
        ipv4_address: 172.20.0.2
    restart: unless-stopped

  test-server:
    image: python:3.9-slim
    networks:
      ssrf-test:
        ipv4_address: 172.20.0.3
    volumes:
      - ./server.py:/app/server.py
    working_dir: /app
    command: sh -c "pip install flask && python server.py"
    ports:
      - "8000:8000"

volumes:
  grafana-storage:
```

The Grafana configuration file (`grafana.ini`) includes the whitelist setting:

```ini
[paths]
data = /var/lib/grafana
logs = /var/lib/grafana/logs
plugins = /var/lib/grafana/plugins

[server]
protocol = http
http_port = 3000

[security]
data_source_proxy_whitelist = 127.0.0.1:3000
allow_embedding = true

[users]
default_theme = dark
```

Our test server (`server.py`) logs all incoming requests:

```python
from flask import Flask, jsonify, request
import logging
import json

# Set up logging
logging.basicConfig(level=logging.INFO)
logger = logging.getLogger(__name__)

app = Flask(__name__)

@app.route('/', defaults={'path': ''})
@app.route('/<path:path>')
def catch_all(path):
    # Convert headers to dict and log them
    headers_dict = dict(request.headers)
    logger.info(json.dumps({
        "headers": headers_dict,
        "method": request.method,
        "path": f"/{path}" if path else "/"
    }, indent=2))

    return jsonify({
        "SSRF POC": "success",
        "path": f"/{path}" if path else "/"
    })

if __name__ == "__main__":
    app.run(host="0.0.0.0", port=8000)
```

### Testing the Behavior

With this environment set up, we can demonstrate the different endpoint behaviors:

1. Start the environment:
   ```bash
   docker-compose up -d
   ```

2. Log into Grafana at http://localhost:3000 (default credentials: admin/admin)

3. Create a data source pointing to our test server (http://172.20.0.3:8000)

![](Pasted%20image%2020250204204228.png)

4. Test both endpoints:
   - `/api/datasources/proxy/<uid>` - This will be blocked due to whitelist
   
![](Pasted%20image%2020250204204325.png)
   
   - `/api/datasources/uid/<uid>/resources/` - This will successfully connect
   
![](Pasted%20image%2020250204204359.png)

To automate this testing, I created a [Python script](https://github.com/jsnv-dev/grafana_data_source_poc/blob/main/poc_ssrf.py) that demonstrates the behavior:

```bash
python poc_ssrf.py --url http://127.0.0.1:3000 --username admin --target http://172.20.0.3:8000/test_poc
```

![](Pasted%20image%2020250204204603.png)

The script will:
1. Login to Grafana
2. Create a data source
3. Attempt connections through both endpoints
4. Clean up by removing the test data source

### Observed Results

When running these tests, we can observe that:
- The proxy endpoint correctly enforces the whitelist.
- The resources endpoint successfully connects regardless of whitelist settings.
- All requests are logged by our test server for verification.

This setup clearly demonstrates the different handling of requests between the two endpoints, helping understand Grafana's proxy implementation.

## Security Considerations

While this behavior is working as designed, it raises important security considerations:

1. Organizations relying on the whitelist for security controls should be aware of this architectural design.
2. The different behavior between endpoints needs to be considered in security planning.
3. Access to data source creation should be carefully controlled.
4. Network-level access controls (like firewall rules or network segmentation) should be implemented as additional security measures, rather than relying solely on application-level restrictions.

## Conclusion

Understanding these architectural decisions in Grafana's implementation is crucial for security teams. While the behavior might initially appear as a security bypass, it's an intentional design choice that needs to be considered when planning security controls.

The Grafana team plans to update their documentation to provide more clarity about these different endpoints and their behavior. For more details about the `CallResource` API, refer to [Grafana's official documentation](https://grafana.com/developers/plugin-tools/key-concepts/backend-plugins/#resources).

## Acknowledgments

Special thanks to the Grafana security team for their transparency and collaboration in understanding this behavior, and for allowing the public disclosure of this finding.

## References
- [Github repository of the codes involved in this post](https://github.com/jsnv-dev/grafana_data_source_poc)
- [Grafana's official documentation](https://grafana.com/developers/plugin-tools/key-concepts/backend-plugins/#resources)

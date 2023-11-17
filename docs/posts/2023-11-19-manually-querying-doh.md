---
comments: true
date: 2023-11-19
categories:
  - DNS
authors:
  - tupcakes
---


# Manually Querying DNS Over HTTPS

Not much too this really. I might expand this later.

```
curl -H 'accept: application/dns-json' \
'https://dns.nextdns.io/abc123?name=google.com&type=A' | jq .
```

<!-- more -->

!!! info

    The Status value below will return 0 for success. In my testing it returned 3 when it failed, but there might be other status values.

```json title="Response"
{
  "Status": 0,
  "TC": false,
  "RD": true,
  "RA": true,
  "AD": false,
  "CD": false,
  "Question": [
    {
      "name": "google.com.",
      "type": 1
    }
  ],
  "Answer": [
    {
      "name": "google.com.",
      "type": 1,
      "TTL": 300,
      "data": "142.250.190.142"
    }
  ],
  "Additional": [
    {
      "name": ".",
      "type": 41,
      "TTL": 0,
      "data": "\n;; OPT PSEUDOSECTION:\n; EDNS: version 0; flags:; udp: 1232"
    }
  ]
}
```
# Logrotate-to-S3-option1
Logrotate access logs to S3

---
Objective: Move access logs from your cluster to S3 for long-term storage and possible anaysis

Seems like a simple problem...  Here are the existing options:

1. syslog-ng or rsyslogd logging to a central location.
Pros: centralized logs, easy to analyse, near to real time
Cons: Requires extra dedicated server (costs). Have to extend storage or upload to S3 periodically. PITA setup from a security point of view unless you know the IP addresses in advance (you can set up your own CA and generate a number of SSL keys and distribute these to the servers on startup and use TLS encrypted communications, but I really CBA with all that hassle – not to mention that last time I checked Amazon Linux didn’t support these serversOOTB so I’d have to install them from a CentOS RPM or similar)

2. Message queuing
(I don’t think this is a perfect match)

3. Hadoop cluster
Pros: awesome data crunching ability
Cons: we don’t have 10million users yet, I think this is a bit heavy handed (not to mention expensive)

4. Scribe from Facebook
Cons: too much setup, requires specific server heirarchy.
logg.ly and splunk
Cons: too expensive/untested/unknown for now

5. Something else…
In this case I will be using logrotate util, so config “/etc/logrotate.d/httpd”:

```

/var/log/http/*access-* {
    missingok
    notifempty
    sharedscripts
    delaycompress
    postrotate
        /bin/systemctl reload httpd.service > /dev/null 2>/dev/null || true
<strong>        /usr/local/utils/apache_log_uploader.sh > /dev/null 2>/dev/null || true</strong>
    endscript
}

```



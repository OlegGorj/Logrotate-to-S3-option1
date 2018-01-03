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


Let’s go through the options of logrotate config:

details of logrotate utility are here: https://linux.die.net/man/8/logrotate

> weekly: Rotate logs once per week. Available options are daily, weekly, monthly, and yearly.
> missingok: If no *access.* files are found, don’t freak out :)
> rotate 52: Keep 52 files before deleting old log files (Thats a default of 52 weeks, or one years worth of logs!)
> compress: Compress (gzip) log files
delaycompress: Delays compression until 2nd time around rotating. As a result, you’ll have one current log, one older log which remains uncompressed, and then a series of compressed logs.
compresscmd: Set which command to used to compress. Defaults to gzip.
uncompresscmd: Set the command to use to uncompress. Defaults to gunzip.
> notifempty: Don’t rotate empty files
> create 640 root adm: Create new log files with set permissions/owner/group, This example creates file with user rootand group adm. In many systems, it will be root for owner and group.
> sharedscripts: Run postrotate script after all logs are rotated. If this is not set, it will run postrotate after each matching file is rotated.
> postrotate: Scripts to run after rotating is done. I.e, Apache is reloaded so it writes to the newly created log files. Reloading Apache (gracefully) lets any current connection finish before reloading and setting the new log file to be written to.
> prerotate: Run before log rotating begins


so, file `/usr/local/utils/apache_log_uploader.sh` would look like:

```
APACHE_LOG_PATH='/var/log/httpd'
LOG_GLOB='*.access.log-*'
BUCKET_NAME='apache-access-logs'
AWS_CLI='/usr/local/bin/aws'
 
# Iterate over all log files matching regex in path
for filepath in `ls $APACHE_LOG_PATH/$LOG_GLOB`
do
 # Check if is file
 [[ -e $filepath ]] || continue
 
 log_filename="$(hostname)_$(basename $filepath)"
 
 # Check if file exists is S3 bucket
 if [ `$AWS_CLI s3 --output text ls s3://$BUCKET_NAME/$log_filename | wc -l` -gt 0 ]
 then
 echo "File $log_filename exists"
 continue
 fi
 
 $AWS_CLI s3 cp $filepath s3://$BUCKET_NAME/$log_filename
done

```





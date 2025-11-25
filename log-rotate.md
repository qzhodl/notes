logrotate can automatically:
 - Rotate logs by size or by date
 - Compress old logs
 - Delete outdated logs
 - Re-create an empty log file after rotation

 1. Create a logrotate configuration file
 ```bash
 sudo vim /etc/logrotate.d/op-node
 ```

```bash
/path/to/your.log {
    size 100M
    rotate 7
    compress
    delaycompress
    missingok
    copytruncate
    notifempty
    su es es
}
```

Explanation:
 - Rotate the log when it exceeds 100 MB
 - Keep 7 old log files
 - Gzip old logs automatically
 - Compress the previous rotation one day later
 - Do not report an error if the file is missing
 - Copy and truncate the current log (required when using tee)
 - Do not rotate empty files
 - Run rotation as user/group es:es

2. Test whether the logrotate config works
```bash
sudo logrotate -d /etc/logrotate.d/op-node
```
3. Execute rotation for real
```bash
sudo logrotate -f /etc/logrotate.d/op-node
```
4. logrotate runs automatically every day
```bash
systemctl status logrotate.timer
```

delete old logs
backup
disk utilisation and trigger email


1. which directory
2. is the directory exist?
3. find the files *.log older than 14 days
4. remove them

log the script execution

find . -name "*.log" -mtime +14
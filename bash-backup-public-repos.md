# Backup Public Repositories

You never know when disaster will strike.  Backup important vendor dependencies with rclone.

> https://rclone.org/

```shell script
#!/usr/bin/env bash
# User Script:
# bash /volume1/GithubBackups/backup-github.sh

# Exit on Errors and switch to parent directory of script.
set -e; cd "$(dirname "${BASH_SOURCE[0]}")"

# Local Destination Directory
destination=./data/github

# Setup Log File
logFile="./logs/github-$(date +"%Y-%m-%d_%H-%M-%S").log"

# Log Writer
function logger(){
  echo "$1" >> "$logFile"
}

# Begin Backup Operations
logger "Running RClone in $PWD as $(whoami)"
logger "Logfile: $logFile"
logger "Synchronizing Github Repos"

declare -a repoCollection=(
"repo-a" \
"repo-b" \
"repo-c"
)

# Iterate the string array using for loop
for repo in "${repoCollection[@]}"; do
  logger "Synchronizing $repo..."
  ./rclone-linux \
  copyurl \
  --verbose \
  "https://github.com/bayareawebpro/$repo/archive/master.zip" \
  "$destination/$repo-master.zip" \
  >> "$logFile" 2>&1
done

logger "Backup Completed"
```

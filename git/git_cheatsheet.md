# Git

Useful commands for using git

##### Clone Repo

```bash
git clone https://github.com/my-username/my-repo.git
```

##### Create and checkout a new branch

```bash
git checkout -b "Mybranch"
```

##### Lazy add/commit/push

This will add all changed files, commit with a message and then push to the specified branch

```bash
git add -A && git commit -m "change all the 1s to 0s" && git push origin Mybranch
```

##### Update branch from latest in master

This will retain the history tracking

```bash
git checkout master    
git pull --rebase
git checkout Mybranch    
git rebase master
git push -f origin Mybranch
```

##### Config Commands (including credentials)

```bash
# List config
git config --list

# Use username/password store:
git config --global credential.helper store
# NOTE: The store is in ~/.git-credentials by default with a format of https://username:password@your.github.hostname

# Change to SSH keys auth (for a repo that uses https)
# Check current settings:
git remote -v

# Change to SSH
git remote set-url origin git@your.github.hostname:ORG_OR_USERNAME/REPO_NAME.git
```


##### Housekeeping
Prune local branches if they've been deleted from remote

```bash
git remove prune origin
```

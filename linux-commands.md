# 1. list all the open ports
sudo ss -tulnp

# 2. delete all the running docker contrainers
docker rm -f $(docker ps -q)

# 3. use direnv
```bash
eval "$(direnv hook bash)"
direnv allow
```

# 4. modify password
passwd <user name>

# 5. install mise with specified version
mise self-update 2025.4.5

# 6. change user and group
sudo chown user:group /path/to/file

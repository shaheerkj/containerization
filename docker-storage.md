Containers are **ephemeral** (temporary).  
If you stop/remove a container, everything inside its filesystem is gone.

Docker provides two main solutions:

# Volumes

A **Docker volume** is a special directory **managed by Docker**, stored on your host (your machine), but _outside_ the container.

```
/var/lib/docker/volumes 
```

A volume is NOT deleted when a container is deleted. So volumes → permanent storage.

### Advantages of volumes:


- Docker manages them → safer & more portable 
- Great for databases 
- Can be shared between containers 
- Backed up easily 
- Survive container restarts/rebuilds 
- Isolated from the container filesystem
- More secure than bind mounts (Docker controls access)

## Creating volumes & operations:

```bash
#creating volume
docker volume create mydata

# list volumes
docker volume ls

# Inspect volume
docker volume inspect mydata

# Using a volume with a container
# mydata is the vol name, and the path after : is where MySQL stores DB files  inside the container
docker run -d \ -v mydata:/var/lib/mysql \ mysql
```

### Anonymous Volumes:

If you use 
- `-v /var/lib/mysql`

Docker creates a random-name volume automatically
# Bind Mounts

A **bind mount** is a way to _directly link_ a folder or file from your **host machine** into a **running container**.

This means:
- Anything in your host folder appears **instantly** inside the container
- If you edit on your host, it changes inside the container    
- If the container writes to that path, your host also sees it

Bind mounts are **host-controlled**, unlike Docker volumes (which are Docker-managed).
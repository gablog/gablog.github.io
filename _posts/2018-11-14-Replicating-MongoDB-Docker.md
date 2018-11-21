---
layout: post
title: Replicating MongoDB databases to and from running containers
---

This post builds off  Kynan Rilee’s technique in [this blog post](https://medium.com/kokster/mount-volumes-into-a-running-container-65a967bee3b5) from 2017 to mount volumes.
We assume in this guide you are familiar with docker layers. If not, checkout our blog post on what Docker layers are and how they work [here](/what-are-docker-layers)

## Disclaimer
Following this guide is assumed to be at your own risk. Before running this on any production database please make sure to run it on a test database. We also advise before doing anything with your production database, make a snapshot of your current data in case something goes awry.

## Attaching a volume to your running container
(Skip this if your MongoDB container already has a volume attached to it).

To allow us to send data outside our container we first for our MongoDB database, we first need to attach a volume our container. To prevent downtime and SLA issues, we will be doing this while the container is running. If you are able to stop your container and restart it, adding a `volume` setting to your docker-compose.yml file or Dockerfile, or by running `docker run -v /path_of_folder_on_host:/path_of_folder_in_container` should do the trick.



**Note**: If you are on Mac OSX, you will need to run the below command first before starting.
```
screen ~/Library/Containers/com.docker.docker/Data/com.docker.driver.amd64-linux/tty
```
## Setup Tools

1. Install nsenter
```
docker run --rm -v /usr/local/bin:/target jpetazzo/nsenter
```
2. Install docker-enter and add it to your path
``` 
git clone -b master https://github.com/jpetazzo/nsenter /usr/local/lib/docker-enter
chmod +x /usr/local/lib/docker-enter/docker-enter
ln -s /usr/local/lib/docker-enter/docker-enter /usr/local/bin/docker-enter
```


## Setup Host Machine Filesystem


### 1. Create folder on your host machine that will link to your volume
```
mkdir ~/testvolume
REALPATH=~/testvolume†
```

### 2. We then want to figure out which filesystem (on the host) contains your folder
```
FILESYS=$(df -P $REALPATH | tail -n 1 | awk '{print $6}')
```

### 3. Retrieve device of filesystem
#### For Linux:
```
while read DEV MOUNT JUNK
do [ $MOUNT = $FILESYS ] && break
done </proc/mounts
```

#### For MacOS:
```
DEV=$(df -P $REALPATH | tail -n 1 | awk '{print $1}' )
```

### 4. Get sub root 
#### For Linux:
```
while read A B C SUBROOT MOUNT JUNK
do [ $MOUNT = $FILESYS ] && break
done < /proc/self/mountinfo 
```

#### For MacOS:
```
SUBROOT=$(df -P $REALPATH | tail -n 1 | awk '{print $6}’ )
```

### 5. Compute Subpath
```
SUBPATH=$(echo $REALPATH | sed s,^$FILESYS,,)
```

### 6. Get device type numbers
#### For Linux:
```
DEVDEC=$(printf "%d %d" $(stat --format "0x%t 0x%T" $DEV))
```

#### For MacOS:
```
DEVDEC=$(printf "%d %d" $(stat -f "0x%Hp 0x%Lp” $DEV))
```


### 7. Create device (inside container)
```
docker-enter bawkbox_db_1 bash -c "[ -b $DEV ] || mknod --mode 0600 $DEV b $DEVDEC"
```

### 8. Create temporary mount point and mount filesystem
```
docker-enter bawkbox_db_1 bash -c " mkdir /tmpmnt  "
docker-enter bawkbox_db_1 bash -c " mount $DEV /tmpmnt  "
```

### 9. Bind-mount volume to mount point
```
docker-enter bawkbox_db_1 bash -c " mkdir -p /src "
docker-enter bawkbox_db_1 bash -c " mount -o bind /tmpmnt/$SUBROOT/$SUBPATH /src "
```

### 10. Cleanup unused folders
```
docker-enter bawkbox_db_1 bash -c " umount /tmpmnt "
docker-enter bawkbox_db_1 bash -c " rmdir /tmpmnt "
```

## Retrieving data from your MongoDB Container

Now that you have successfully attached a volume to your running MongoDB container, now we need to dump the database into a file on our volume

```
docker-enter bawkbox_db_1 bash -c ‘mongodump --out /src --host mongo:27017’`
```

Your db dump will then be available at the directory specified in $REALPATH

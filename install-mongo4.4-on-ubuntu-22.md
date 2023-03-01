## How to install MongoDB 4.4 on Ubuntu 22.04

### 1. Import the public key
We need to import the public key for the package management system.
Run the following from the terminal:
```
wget -qO - https://www.mongodb.org/static/pgp/server-4.4.asc | sudo apt-key add -
```
This operation should respond with an `OK`.
You may receive a warning that this process is deprecated and you should use keyring but for now, this will work.

However, if you receive error that gnupg is not installed, please try the following:
- 1.1. Install `gnupg`:
    ```
    sudo apt install gnupg
    ```
- 1.2. Retry importing the key:
    ```
    wget -qO - https://www.mongodb.org/static/pgp/server-4.4.asc | sudo apt-key add -
    ```

### 2. Create a list file for MongoDB
To let APT know where to look for the online sources of packages to download and install MongoDB on your system, we need `sources.list` and `sources.list.d`.
`sources.list` is a file that defines the sources to get the packages. It maintains one source per line with the most preferred source at the top.
`sources.list.d` is a directory where APT looks for `sources.list` files.
Run the following to create a file `mongodb-org-4.4.list` under `sources.list.d`:
```
echo "deb [ arch=amd64,arm64 ] https://repo.mongodb.org/apt/ubuntu focal/mongodb-org/4.4 multiverse" | sudo tee /etc/apt/sources.list.d/mongodb-org-4.4.list
```

### 3. Update the local package index on system to let it know where to find `mongodb-org`
```
sudo apt update
```

### 4. Install libssl1.1
Ubuntu 22.04 moved to security library libssl3.0, however, MongoDB 4.4 require libssl1.1 to install.<br/>
**It's NOT recommended** to use any workaround in Ubuntu 22.04 to install MongoDB, because it can lead to problems if you're gonna use it in production. Below is the workaround that worked for me:
- 4.1. Download `libssl1.1_1.1.1f-1ubuntu2_amd64.deb` from official repository:
    ```
    wget http://archive.ubuntu.com/ubuntu/pool/main/o/openssl/libssl1.1_1.1.1f-1ubuntu2_amd64.deb
    ```
- 4.2. Install it:
    ```
    sudo dpkg -i libssl1.1_1.1.1f-1ubuntu2_amd64.deb
    ```

### 5. Install MongoDB packages.
```
sudo apt-get install -y mongodb-org
```

### 6. Setup MongoDB
- 6.1. Create and modify `/data/db` directory
    MongoDB 4.4 uses `/data/db` by default. Ignores the one mentioned in `/etc/mongod.conf`
    - 6.1.1.
        ```
        sudo mkdir -p /data/db
        ```
    - 6.1.2.
        ```
        sudo chown -R mongodb:mongodb /data
        ```
    - 6.1.3.
        ```
        sudo chmod -R a+w /data
        ```
- 6.2. Create and modify `/var/lib/mongodb` directory
    - 6.2.1.
        ```
        sudo mkdir -p /var/lib/mongodb
        ```
    - 6.2.2.
        ```
        sudo chown -R mongodb:mongodb /var/lib/mongodb
        ```
    - 6.2.3.
        ```
        sudo chmod -R a+w /var/lib/mongodb
        ```
- 6.3. Create and modify `/tmp/mongodb-27017.sock` file
    - 6.3.1.
        ```
        touch /tmp/mongodb-27017.sock
        ```
    - 6.3.2.
        ```
        sudo chown mongodb:mongodb /tmp/mongodb-27017.sock
        ```
    - 6.3.3.
        ```
        sudo chmod -R a+w /tmp/mongodb-27017.sock
        ```
- 6.4. Update `/etc/mongod.conf`
    - 6.4.1. Open `/etc/mongod.conf`
    - 6.4.2. Change `dbPath` to `/data/db`
    - 6.4.3. Save the conf file.
- 6.5. Setup `ulimit`
    - Starting from 4.4, MongoDB startup throws error if `ulimit` value for number of open files is under 64000.
    - Fix this by running:
        ```
        ulimit -n 64000
        ```

### 7. Run MongoDB service
- 7.1 Try running `mongod.service`
    - 7.1.1. Run the following without root privileges:
        ```
        /usr/bin/mongod --config /etc/mongod.conf
        ```
    If `mongod.service` is running without failure, you are good to go. Mongo is working properly. Try running `mongo` in a new terminal, you should be able to login to MongoDB shell.
    However, if running `mongo` doesn't work, you can check the last few lines of `/var/lib/mongodb/mongod.log`. It will be mentioned with a proper message about exactly where/when/how it failed. Most probably, the reason will be related to file permissions. To fix that, run the following two commands on the mentioned file/directory:
    ```
    sudo chown -R mongodb:mongodb <file/directory>
    sudo chmod -R a+w <file/directory>
    ```
    #### NOTE: DO NOT RUN MONGOD.SERVICE WITH ROOT PRIVILEGES, IT WILL CHANGE FILE PERMISSIONS, CREATING MORE PROBLEMS

### 8. Enable `mongod.service` on system start
- 8.1. Check if service is running or not:
    ```
    sudo systemctl status mongod.service
    ```
- 8.2. If running (green):
    ```
    sudo systemctl enable mongod.service
    ```
- 8.3. If not running:
    ```
    sudo systemctl start mongod.service
    ```
    - 8.3.1. Try step 8.1 again
    - 8.3.2. If not resolved yet, try solving with `7.1`

### 9. Optional Settings
- To prevent unintended upgrades by `apt`, you can pin the package at the currently installed version:
    ```
    echo "mongodb-org hold" | sudo dpkg --set-selections
    echo "mongodb-org-server hold" | sudo dpkg --set-selections
    echo "mongodb-org-shell hold" | sudo dpkg --set-selections
    echo "mongodb-org-mongos hold" | sudo dpkg --set-selections
    echo "mongodb-org-tools hold" | sudo dpkg --set-selections
    ```
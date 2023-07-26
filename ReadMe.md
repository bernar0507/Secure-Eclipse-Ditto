# Securing Eclipse Ditto through Authetication and Encryption at Rest in MongoDB

This repo will contain the steps to setup this Authetication and Encryption at Rest in MongoDB.

## 1 - Prepare the Directories
You need to create the directory on your host machine that will be used for the Docker volume. 
Remember, you have to replace /path/on/host/to/mongodb-decrypted in your YAML file with this actual path. Once you've decided on a directory, create it using the mkdir command. 
For example:
```
mkdir -p /path/on/host/to/mongodb-decrypted
```
Replace /path/on/host/to/mongodb-decrypted with the actual directory path.

## 2 - Install and Configure eCryptfs
Install eCryptfs on your host system. 
On a Debian-based system, this would look like:
```
sudo apt-get update
sudo apt-get install ecryptfs-utils
```

## 3 - Set up an Encrypted Mount point
Once eCryptfs is installed, you will set up an encrypted mount point. 
This involves creating an additional directory for the encrypted data, which should not be the same as the one you are using for the Docker volume.
```
sudo mkdir -p /path/to/encrypted/data
```

## 4 - Create the Encrypted Mount point
```
sudo mount -t ecryptfs /path/to/encrypted/data /path/on/host/to/mongodb-decrypted
```



* Select key type to use for newly created files: 
```
 1) tspi
 2) passphrase
```

* Select option 2:
```Selection: 2```

* Write the passphrase:
```Passphrase: ```


* Select cipher:
``` 
 1) aes: blocksize = 16; min keysize = 16; max keysize = 32
 2) blowfish: blocksize = 8; min keysize = 16; max keysize = 56
 3) des3_ede: blocksize = 8; min keysize = 24; max keysize = 24
 4) twofish: blocksize = 16; min keysize = 16; max keysize = 32
 5) cast6: blocksize = 16; min keysize = 16; max keysize = 32
 6) cast5: blocksize = 8; min keysize = 5; max keysize = 16
```

* Select option 1:
```Selection [aes]: 1```

* Select key bytes:
```
 1) 16
 2) 32
 3) 24
```

* Select option 1:
```Selection [16]: 1```

* Enable plaintext passthrough (y/n) [n]: ```n```

* Enable filename encryption (y/n) [n]: ```n```

* Attempting to mount with the following options:
  ecryptfs_unlink_sigs
  ecryptfs_key_bytes=16
  ecryptfs_cipher=aes
  ecryptfs_sig=6d619b4401f1f943
WARNING: Based on the contents of [/root/.ecryptfs/sig-cache.txt],
it looks like you have never mounted with this key 
before. This could mean that you have typed your 
passphrase wrong.

* Would you like to proceed with the mount (yes/no)? :
```yes```

* Would you like to append sig [6d619b4401f1f943] to [/root/.ecryptfs/sig-cache.txt]  in order to avoid this warning in the future (yes/no)? :
```yes```

* You should receive this message:
```
Successfully appended new sig to user sig cache file

Mounted eCryptfs
```

## 5 - Change permissions so Mongo can access

* change owner
```
sudo chown -R $(whoami):$(whoami) /home/bagao/project2/mongodb-decrypted
```

* change permissions
```
sudo chmod -R 777 /home/bagao/project2/mongodb-decrypted
```



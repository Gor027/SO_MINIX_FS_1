# SO_MINIX_FS_1
Enhancing the *mfs* server with prototype encryption of file contents

# File Encryption

## Encrypting contents

This is an in-course project from the subject Operating Systems in the faculty of MIM, University of Warsaw.
The purpose is to redevelop the *mfs* server, which supports the MINIX file system, to enable a prototype encryption of file contents.

*Note: The word file is used in a send of a regular file, not a directory.*

Only the content of files is encrypted: their content should be encrypted during the write operation and decrypted during the read operation. Directories (e.g. directory structure on the disk) and file metadata (e.g. name, size, permissions) are not encrypted.

The user provides the encryption key by saving it to a file called `KEY`, which is located in the root directory of the partition. However, this value is not actually saved in the file content, but only properly interpreted by the modified mfs server . File metadata `KEY` (including size and modification time) should also not be changed . The key provided is used to encrypt the contents of the files in the given partition until the partition is unmounted or until the user provides another key. Writing to the file `KEY` more data than 1 byte should end in error `EINVAL`. However, attempting to read the file content `KEY` should result in an error `EPERM`.

The mfs server does not validate the key: each 1-byte value specified is treated as a valid key. By changing (manually) the keys, the user can encrypt different files with different keys. When the key value is 0, the data before and after encryption looks the same. Example:

```
# cd <główny katalog partycji>
# touch KEY
# touch plik
# ls
KEY  plik
# printf '\x2A' > ./KEY
# echo 'ALA MA KOTA' > ./plik
# cat ./plik
ALA MA KOTA
# cat ./KEY
cat: ./KEY: Operation not permitted
# printf '\x0' > ./KEY
# cat ./plik
kvkJwkJuy~k4#
```
## Blocking partitions

To prevent accidental reading or writing of the file before the user provides the encryption key, the partition is blocked immediately after mounting : attempting to read or write the content of the file should result in an error `EPERM` (except pseudo-write to the file `KEY`). The partition is automatically unlocked when the user provides the key.

All other file operations (including creation, deletion, renaming) and directories are not blocked. Example:

```
# cd <główny katalog właśnie zamontowanej partycji>
# ls
KEY  plik
# echo 'ala ma kota' | tee ./plik
tee: ./plik: Operation not permitted
ala ma kota
# cat ./plik
cat: ./plik: Operation not permitted
# printf '\x2A' > ./KEY
# echo 'ala ma kota' | tee ./plik
ala ma kota
# cat ./plik
ala ma kota
#
```

## Deactivate encryption

If the user does not want to encrypt the partition, he can put a file or directory named in its root directory `NOT_ENCRYPTED`. When such a file or directory is present, the content of the files is not encrypted during writes and decrypted during reads, and the partition is not blocked. However, it is not possible then to change the key: an attempt to write to a file `KEY` that is located in the root directory of the partition should result in an error `EPERM`.

The file or directory `NOT_ENCRYPTED` can be created (deactivated encryption) and deleted (enabled encrypted) by the user at any time. After deleting it, the partition is further encrypted with the key that was previously set or locked if the user has not previously set any key. Example:

```
# cd <główny katalog partycji>
# ls
KEY  plik
# touch ./NOT_ENCRYPTED
# echo 'ala ma kota' > ./plik
# cat ./plik
ala ma kota
# printf '\x2A' | tee ./KEY
tee: ./KEY: Operation not permitted
*# rm ./NOT_ENCRYPTED
# printf '\x2A' | tee ./KEY
*# cat ./plik
7B7�C7�AEJ7�#
```

## Tips

If the MINIX is running under QEMU you can attach additional hard drives to test modifications.

1. On your host computer create a new file which corresponds to the new disk: `qemu-img create -f raw extra.img 1M`

2. Connect the disk to the virutal machine, adding to the parameters with which QEMU is run, parameters `-drive file=extra.img,format=raw,index=1,media=disk` where the parameter index specifies the disk serial number (0 is the main disk - the image of your machine).

3. The first time you create a new disk file system MFS: `/sbin/mkfs.mfs /dev/c0d1`

4. Create an empty directory (mkdir /root/new) and mount to the connected drive: mount /dev/c0d1 /root/new

5. Opeartions will be performed on the disk mounted in this location.

6. To unmount the disk: unmount /root/new

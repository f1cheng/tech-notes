
- mk fs of directories based on xfilesystem format(superblock,inodes blocks, blockdata), then write into into image
```
main.py  mount folder to--> imagename
==write contents in folders into image name file.

  fs = FileSystem(sys.argv[2])
    fs.mount(sys.argv[1])



    def mount(self, path: str) -> None:
        if not os.path.isdir(path):
            raise ValueError("File system can only mount directories, not files.")
        
        self._root = self._seek(self._root, 0, path)
        self._disk.write(self._root.binary, self._get_inode_offset(0))
        self._flush()

```

- pseudo filesystem
```
devfs is a special filesystem containing a representation of physical devices.

The implementation of devfs doesn't maintain file system statistics,
like space available (because you can't store anything on devfs),------------------
but because devfs is visible as a part of the filesystem hierarchy,
it must report those values to the regular tools asking for them.
'In effect the tools will show the file system space and inode usage as 100%.
```
-- pseudo fs read 
  ```
   Can be none real file content but to reuse file's read/write/open. even open with dummy func which is just printing "info".
   newfs_file_read in this new newfs_file_op, which is read buffer data from e.g. keyboard or others depends on the dev types
  ```

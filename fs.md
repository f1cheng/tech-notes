
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


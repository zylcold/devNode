那些不使用 Git 的人们创建一个最新的快照归档.

```shell

$ git archive master --prefix='project/' | gzip > `git describe master`.tar.gz 
$ ls *.tar.gz 
v1.6.2-rc1-20-g8c5b85c.tar.gz

```
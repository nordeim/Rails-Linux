```bash
split -b 20M /Home1/rails.tgz rails_part_  

split -n 6 /Home1/rails.tgz rails_part_  

cat rails_part_* > rails_restored.tgz  

split -b 20M -d --verbose /Home1/rails.tgz rails_part_  
-d: Uses numeric suffixes (00, 01, 02) instead of letters (aa, ab, ac).  
--verbose: Shows progress as each new chunk is created.  

=================

$ md5sum rails.tgz mise.tgz 
b17e34b69da4c42e70af2c39e789dc0a  rails.tgz
fdda11c99934df20003f75ab46ca493f  mise.tgz

$ ls -l rails.tgz mise.tgz 
-rw-r--r-- 1 root root  24571943 Dec 16 07:03 mise.tgz
-rw-r--r-- 1 root root 121155295 Dec 16 07:06 rails.tgz

=================

$ which rails
$HOME/.local/share/mise/installs/ruby/3.4.7/bin/rails

$ which gem
$HOME/.local/share/mise/installs/ruby/3.4.7/bin/gem

=================

$ (cd ~/. ; ls -l bin/mise* ; cd /Home1 ; ls -l mise/* ; cd ~/.local/share/mise/installs ; ls -l ruby/*)

-rwxrwxr-x 1 pete pete 2274 Dec 15 07:01 bin/mise_activate.sh

=================

-rw-r--r-- 1 user user 1068 Dec 15 01:33 mise/LICENSE
-rw-r--r-- 1 user user 5833 Dec 15 01:33 mise/README.md

mise/bin:
total 64252
-rwxr-xr-x 1 user user 65778296 Dec 15 01:33 mise
-rw-r--r-- 1 user user    10989 Dec 15 01:33 mise.d

mise/man:
total 4
drwxr-xr-x 2 user user 4096 Dec 15 01:33 man1

mise/share:
total 4
drwxr-xr-x 3 user user 4096 Dec 15 01:33 fish

=================

lrwxrwxrwx 1 pete pete    7 Dec 15 07:07 ruby/3 -> ./3.4.7
lrwxrwxrwx 1 pete pete    7 Dec 15 07:07 ruby/3.4 -> ./3.4.7
lrwxrwxrwx 1 pete pete    7 Dec 15 07:07 ruby/latest -> ./3.4.7

ruby/3.4.7:
total 16
drwxr-xr-x 2 pete pete 4096 Dec 15 14:40 bin
drwxr-xr-x 3 pete pete 4096 Dec 15 07:07 include
drwxr-xr-x 4 pete pete 4096 Dec 15 07:07 lib
drwxr-xr-x 5 pete pete 4096 Dec 15 07:07 share


```

https://share.google/aimode/YrEy3o65K6EjOaOWg 

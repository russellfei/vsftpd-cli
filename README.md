# vsftpd-cli
Bug fixed autocmd for vsftpd configuration on CentOS7



## Installation Guide

See *`VSFTPD` configuration notes*



## Improvements

1. improve removal of virtual user that previous versions failed to remove user group at `etc/group`
2. add `allow_writeable_chroot` in virtual user template to guarantee success running under `TLSv1` encryption
3. update service reload cmd

## Discussions

1. Whether we use `allow_writeable_chroot` is a good idea?

   A: According to my test, virtual writeable chroot is separated from my root files.



## Feedback

mailto: yangwanggauss [at] gmail.com
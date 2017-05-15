# VSFTPD Configuration Notes

Author: feifeng @ supernengine Inc

Date: May 11th, 2017



## 1. Introduction

`vsftpd` is short for Very Secure File Transfer Protocol Daemon. Chinese introduction is [here](http://os.51cto.com/art/201008/222036.htm)

## 2. Motivation

It is built for the 3rd release of super engine image service platform. There are dozens of users who want to download and upload their data or analysis results, and in that circumstances we have to provide fast and secure ftp service to support the data flow.

## 3. System Info & Utilities

|  Name  | Version  |                   Info                   |
| :----: | :------: | :--------------------------------------: |
| CentOS | 7.3.1611 |             operation system             |
| vsftp  |  3.0.2   |         `yum install -y vsftpd`          |
| libdb  |  5.3.21  | Berkeley DB tools `yum install -y libdb` |



## 4. Configuration

### 4.1 Common FTP directories

- `/etc/vsftpd/`: all configurations for `vsftpd`
- `/etc/pam.d/vsftpd`: configuration for PAM authentication
- `/var/log/secure`: LOG file for vsftpd
- `/var/ftp/pub`: anonymous user directory
- `/var/vsftpd/`: SELF-DEFINED virtual user root directory **NEED `mkdir`**

### 4.2 Open server side ports

```shell
iptables -A IN_public_allow -m state --state NEW -m tcp -p tcp --dport 21 -j ACCEPT
iptables -A IN_public_allow -m state --state NEW -m tcp -p tcp --dport 64000:65535 -j ACCEPT
```



### 4.3 Setup local accessible FTP service

```shell
./vsftpd_virtual_config_withTLS.sh
./vsftpd_virtualuser_add.sh
./vsftpd_virtualuser_info.sh
```





## Appendix

- Auto Configuration `vsftpd` without encryption	


```shell
  #!/bin/sh
  # LSN VSFTPD chroot install
  # Version 1.0
  # August 1, 2005
  # Fire Eater <LinuxRockz@gmail.com>
  # Released under the GPL License- http://www.fsf.org/licensing/licenses/gpl.txt
  # May 12, 2017
  # Updated by Feng Fei <feifeng@superengine.com.cn>
  # ALL RIGHTS RESERVED
  ##############################################################################
  #
  IP_Address="`( /sbin/ifconfig | head -2 | tail -1 | awk '{ print $2; }' | tr --delete [a-z]:)`"
  My_FTP_User="my_ftp_virtual_user"
  My_FTP_Password="my_secret_password"

  echo ""
  echo "Setting up Vsftpd with non-system user logins"
  echo ""
  #
  #
  mv  /etc/vsftpd/vsftpd.conf /etc/vsftpd/vsftpd.conf.orig
  cat <<EOFVSFTPD> /etc/vsftpd/vsftpd.conf
  anon_world_readable_only=NO
  anonymous_enable=NO
  chroot_local_user=YES
  guest_enable=NO
  guest_username=ftp
  hide_ids=YES
  listen=YES
  listen_address=$IP_Address
  local_enable=YES
  max_clients=100
  max_per_ip=2
  nopriv_user=ftp
  pam_service_name=ftp
  pasv_max_port=65535
  pasv_min_port=64000
  session_support=NO
  use_localtime=YES
  user_config_dir=/etc/vsftpd/users
  userlist_enable=YES
  userlist_file=/etc/vsftpd/denied_users
  xferlog_enable=YES
  anon_umask=0027
  local_umask=022
  async_abor_enable=YES
  connect_from_port_20=YES
  dirlist_enable=NO
  download_enable=NO
  EOFVSFTPD

  cat /etc/passwd | cut -d ":" -f 1 | sort > /etc/vsftpd/denied_users; mkdir /etc/vsftpd/users
  sed -e '/'$My_FTP_User'/d' < /etc/vsftpd/denied_users > /etc/vsftpd/denied_users.tmp
  mv /etc/vsftpd/denied_users.tmp /etc/vsftpd/denied_users
  chmod 644 /etc/vsftpd/denied_users

  cat <<EOFPAMFTP> /etc/pam.d/ftp
  auth    required pam_userdb.so db=/etc/vsftpd/accounts
  account required pam_userdb.so db=/etc/vsftpd/accounts
  EOFPAMFTP

  cat <<EOFVSFTPU> /etc/vsftpd/users/$My_FTP_User
  dirlist_enable=YES
  download_enable=YES
  local_root=/var/ftp/virtual_users/$My_FTP_User/
  write_enable=YES
  EOFVSFTPU

  echo $My_FTP_User > /etc/vsftpd/accounts.tmp
  echo $My_FTP_Password >> /etc/vsftpd/accounts.tmp
  /usr/bin/db_load -T -t hash -f  /etc/vsftpd/accounts.tmp /etc/vsftpd/accounts.db

  #
  # Set Permissions
  #
  chmod 600 /etc/vsftpd/accounts.db
  if [ /usr/sbin/selinuxenabled ];then
      printf ' Setting up SELinux Boolean (allow_ftpd_anon_write 1) ... '
      /usr/sbin/setsebool -P allow_ftpd_anon_write 1
      printf "Done.\n"
  fi
```

  ​



- Auto Configuration `vsftpd` with `TLSv1`  encryption

  ```shell
  #!/bin/sh
  # Chroot FTP with Virtual Users - Installation (with TLS support)
  #
  # Version 1.0
  #       August 1, 2005
  #       Fire Eater <LinuxRockz@gmail.com>
  #
  # Version 1.1
  #       December 14, 2008
  #       Alain Reguera Delgado <alain.reguera@gmail.com>
  #
  # Released under the GPL License- http://www.fsf.org/licensing/licenses/gpl.txt
  # May 12, 2017
  # Updated by Feng Fei <feifeng@superengine.com.cn>
  # ALL RIGHTS RESERVED
  #
  # Initialization
  #
  IP_Address="`( /sbin/ifconfig | head -2 | tail -1 | awk '{ print $2; }' | tr --delete [a-z]:)`"
  #LOCALPATH=`pwd`
  LOCALPATH=/home/vsftp/
  #
  # Add some presentation :)
  #
  clear;
  echo " vsftpd 2.0.5 -> Virtual Users -> Configuration"
  echo '-------------------------------------------------------------------'

  #
  # Check dependencies
  #
  PACKISMISSING=""
  PACKDEPENDENCIES="vsftpd libdb-utils"
  for i in `echo $PACKDEPENDENCIES`; do
      /bin/rpm -q $i > /dev/null
      if [ "$?" != "0" ];then
          PACKISMISSING="$PACKISMISSING $i"
      fi
  done
  if [ "$PACKISMISSING" != "" ];then
      echo " ATTENTION: The following package(s) are needed by this script:"
      for i in `echo $PACKISMISSING`; do
          echo "             - $i"
      done
      echo '-------------------------------------------------------------------'
      exit;
  fi

  #
  # Move into pki and create vsftpd certificate.
  #
  echo ''
  echo ' Creating Vsftpd RSA certificate ...'
  echo ''

  cd /etc/pki/tls/certs/
  if [ -f vsftpd.pem ];then
  	rm vsftpd.pem
  fi
  make vsftpd.pem

  #
  # Set up vsftpd configuration
  #
  echo '' 
  printf ' Setting up Vsftpd with non-system user logins and TLS support ... '

  mv  /etc/vsftpd/vsftpd.conf /etc/vsftpd/vsftpd.conf.orig
  cat <<EOFVSFTPD> /etc/vsftpd/vsftpd.conf
  anon_world_readable_only=NO
  anonymous_enable=NO
  chroot_local_user=YES
  guest_enable=NO
  guest_username=ftp
  hide_ids=YES
  listen=YES
  listen_address=$IP_Address
  local_enable=YES
  max_clients=100
  max_per_ip=2
  nopriv_user=ftp
  pam_service_name=ftp
  pasv_max_port=65535
  pasv_min_port=64000
  session_support=NO
  use_localtime=YES
  user_config_dir=/etc/vsftpd/users
  userlist_enable=YES
  userlist_file=/etc/vsftpd/denied_users
  xferlog_enable=YES
  anon_umask=027
  local_umask=027
  async_abor_enable=YES
  connect_from_port_20=YES
  dirlist_enable=NO
  download_enable=NO
  #
  # TLS Configuration
  #
  ssl_enable=YES
  allow_anon_ssl=NO
  force_local_data_ssl=NO
  force_local_logins_ssl=YES
  ssl_tlsv1=YES
  ssl_sslv2=NO
  ssl_sslv3=NO
  rsa_cert_file=/etc/pki/tls/certs/vsftpd.pem
  EOFVSFTPD

  #
  # Users
  #
  if [ ! -d /etc/vsftpd/users ]; then
  mkdir /etc/vsftpd/users
  fi
  cat /etc/passwd | cut -d ":" -f 1 | sort > /etc/vsftpd/denied_users; 
  chmod 644 /etc/vsftpd/denied_users
  printf "Done.\n"

  #
  # PAM
  #
  printf ' Setting up PAM ... '
  cat <<EOFPAMFTP> /etc/pam.d/ftp
  auth    required pam_userdb.so db=/etc/vsftpd/accounts
  account required pam_userdb.so db=/etc/vsftpd/accounts
  EOFPAMFTP
  printf "Done.\n"

  #
  # SELinux
  #
  printf ' Setting up SELinux Boolean (allow_ftpd_anon_write 1) ... '
  /usr/sbin/setsebool -P allow_ftpd_anon_write 1
  printf "Done.\n"

  #
  # Add first ftp virtual user
  #
  ${LOCALPATH}/vsftpd_virtualuser_add.sh
  ```



- Auto add virtual user


```shell
  #!/bin/sh
  # Chroot FTP with Virtual Users - Add ftp virtual user
  #
  # Version 1.0
  #       August 1, 2005
  #       Fire Eater <LinuxRockz@gmail.com>
  #
  # Version 1.1
  #       December 14, 2008
  #       Alain Reguera Delgado <alain.reguera@gmail.com>
  #
  # Released under the GPL License- http://www.fsf.org/licensing/licenses/gpl.txt
  # May 12, 2017
  # Updated by Feng Fei <feifeng@superengine.com.cn>
  # ALL RIGHTS RESERVED
  #
  # Initialize some variables
  #
  LOCALPATH=/home/vsftp
  SHELL=/sbin/nologin
  FTPCONF=/etc/vsftpd
  HOMEDIR=/var/ftp/virtual_users

  if [ -f $FTPCONF/accounts.tmp ];then
      ACCOUNTDB_TOTALLINES=`grep '.' -c $FTPCONF/accounts.tmp`
  else
      ACCOUNTDB_TOTALLINES=0
  fi

  function checkNewUser_Existence () {
      C=1;

      if [ "$ACCOUNTDB_TOTALLINES" != "0" ];then
          while [ $C -lt $ACCOUNTDB_TOTALLINES ]; do
              VALIDUSER=`sed -n -e "$C p" $FTPCONF/accounts.tmp`
              if [ "$USERNAME" == "$VALIDUSER" ];then
                  USERNAMEOK=NO
                  break;
              else
                  USERNAMEOK=YES
             fi
             let C=$C+2;
          done 
      fi
  }

  function checkNewUser_Availability () {

      if [ -f $FTPCONF/denied_users ];then
          if [ ! `grep -w $USERNAME $FTPCONF/denied_users` ];then
              USERNAMEOK=YES
  	else
  	    USERNAMEOK=NO
          fi
      
      else
          USERNAMEOK=NO
      fi
  }

  function checkNewUser_Homedir () {

      # Verify User's Home Directory.
      if [ -d $HOMEDIR ];then
          for i in `ls $HOMEDIR/`; do
             VALIDUSER=$i
             if [ "$USERNAME" == "$VALIDUSER" ];then
                 USERNAMEOK=NO
  	       break;
  	   else
  	       USENAMEOK=YES
             fi
          done
      fi
  }

  function getUsername () {

      printf " Enter Username (lowercase)      : "
      read USERNAME

      checkNewUser_Existence;
      checkNewUser_Availability;
      checkNewUser_Homedir;

      if [ "$USERNAMEOK" == "NO" ];then
          echo "  --> Invalid ftp virtual user. Try another username."
          getUsername;
      fi

  }

  #
  # Add some presentation :)
  #
  clear;
  echo " vsftpd 2.0.5 -> Virtual Users -> New User"
  echo '-------------------------------------------------------------------'

  #
  # Check dependencies
  #
  PACKISMISSING=""
  PACKDEPENDENCIES="vsftpd libdb-utils"
  for i in `echo $PACKDEPENDENCIES`; do
      /bin/rpm -q $i > /dev/null
      if [ "$?" != "0" ];then
          PACKISMISSING="$PACKISMISSING $i"
      fi
  done
  if [ "$PACKISMISSING" != "" ];then
      echo " ATTENTION: The following package(s) are needed by this script:"
      for i in `echo $PACKISMISSING`; do
      echo "             - $i"
      done
      echo '-------------------------------------------------------------------'
      exit;
  fi

  #
  # Get user information
  #
  getUsername;
  printf " Enter Password (case sensitive) : "
  read PASSWORD
  printf " Enter Comment(user's full name) : "
  read FULLNAME
  printf " Account disabled ? (y/N)        : "
  read USERSTATUS
  echo " Home directory location         : ${HOMEDIR}/$USERNAME " 
  echo " Home directory permissions      : $USERNAME.$USERNAME | 750 | public_content_rw_t"
  echo " Login Shell                     : $SHELL "

  #
  # Create specific user configuration, based on 
  # vsftpd_virtualuser_config.tpl file.
  #
  cp $LOCALPATH/vsftpd_virtualuser_config.tpl $LOCALPATH/vsftpd_virtualuser_config.tpl.1
  sed -i -e "s/USERNAME/$USERNAME/g;" $LOCALPATH/vsftpd_virtualuser_config.tpl.1
  cat $LOCALPATH/vsftpd_virtualuser_config.tpl.1 > $FTPCONF/users/$USERNAME
  rm -f $LOCALPATH/vsftpd_virtualuser_config.tpl.1

  #
  # Update denied_users file
  #
  if [ "$USERSTATUS" == "y" ];then
  	echo $USERNAME >> $FTPCONF/denied_users	
  else
  	sed -i -r -e "/^$USERNAME$/ d" $FTPCONF/denied_users
  fi

  #
  # Update accounts.db file.
  #
  echo $USERNAME >> $FTPCONF/accounts.tmp; 
  echo $PASSWORD >> $FTPCONF/accounts.tmp;
  rm -f $FTPCONF/accounts.db
  db_load -T -t hash -f  $FTPCONF/accounts.tmp $FTPCONF/accounts.db

  #
  # Create ftp virtual user $HOMEDIR
  #
  if [ ! -d $HOMEDIR  ];then
      mkdir $HOMEDIR
  fi

  #
  # Set user information
  #
  /usr/sbin/useradd -d "${HOMEDIR}/$USERNAME" -s "/sbin/nologin" -c "$FULLNAME" $USERNAME

  #
  # Set Permissions
  #
  /bin/chmod 600 $FTPCONF/accounts.db
  /bin/chmod 750 $HOMEDIR/$USERNAME
  /usr/bin/chcon -t public_content_rw_t $HOMEDIR/$USERNAME

  # Restart vsftpd after user addition.
  echo '-------------------------------------------------------------------'
  /sbin/service vsftpd restart
  echo '-------------------------------------------------------------------'
```

  ​

- Auto update virtual user

  ```shell
  #!/bin/sh
  # Chroot FTP with Virtual Users - Update ftp virtual user information.
  #
  # Version 1.0
  #       August 1, 2005
  #       Fire Eater <LinuxRockz@gmail.com>
  #
  # Version 1.1
  #       December 14, 2008
  #       Alain Reguera Delgado <alain.reguera@gmail.com>
  #
  # Released under the GPL License- http://www.fsf.org/licensing/licenses/gpl.txt
  # May 12, 2017
  # Updated by Feng Fei <feifeng@superengine.com.cn>
  # ALL RIGHTS RESERVED
  #
  # Initialize some variables
  #
  HOMEDIR=/var/ftp/virtual_users
  FTPCONF=/etc/vsftpd
  SHELL=/sbin/nologin
  CHMOD=750
  SELCONTEXT=public_content_rw_t
  ACCOUNTSDB_TMP=$FTPCONF/accounts.tmp
  ACCOUNTSDB_DB=$FTPCONF/accounts.db

  if [ -f $FTPCONF/accounts.tmp ];then
      ACCOUNTDB_TOTALLINES=`grep '.' -c $FTPCONF/accounts.tmp`
  else
      ACCOUNTDB_TOTALLINES=0
  fi

  function checkUser_Existence () {
      C=1;

      if [ "$ACCOUNTDB_TOTALLINES" != "0" ];then
          while [ $C -lt $ACCOUNTDB_TOTALLINES ]; do
              VALIDUSER=`sed -n -e "$C p" $FTPCONF/accounts.tmp`
              if [ "$USERNAME" == "$VALIDUSER" ];then
                  USERNAMEOK=YES
                  break;
              else
                  USERNAMEOK=NO
             fi
             let C=$C+2;
          done 
      fi
  }

  function getUsername () {

      printf " Enter Username (lowercase)      : "
      read USERNAME

      checkUser_Existence;

      if [ "$USERNAMEOK" == "NO" ];then
          echo "  --> Invalid ftp virtual user. Try another username."
          getUsername;
      fi

  }

  #
  # Add some presentation :)
  #
  clear;
  echo ' vsftpd 2.0.5 -> Virtual Users -> Update User';
  echo '-------------------------------------------------------------------'

  #
  # Check dependencies
  #
  PACKISMISSING=""
  PACKDEPENDENCIES="vsftpd libdb-utils"
  for i in `echo $PACKDEPENDENCIES`; do
      /bin/rpm -q $i > /dev/null
      if [ "$?" != "0" ];then
          PACKISMISSING="$PACKISMISSING $i"
      fi
  done
  if [ "$PACKISMISSING" != "" ];then
      echo " ATTENTION: The following package(s) are needed by this script:"
      for i in `echo $PACKISMISSING`; do
          echo "             - $i"
      done
      echo '-------------------------------------------------------------------'
      exit;
  fi

  #
  # Get user information
  #
  getUsername;
  printf " Enter Password (case sensitive) : "
  read PASSWORD
  printf " Enter Comment(user's full name) : "
  read FULLNAME
  printf " Account disabled ? (y/N)        : "
  read USERSTATUS
  echo " Home directory location         : ${HOMEDIR}/$USERNAME " 
  echo " Home directory permissions      : $USERNAME.$USERNAME | 750 | public_content_rw_t"
  echo " Login Shell                     : $SHELL "

  #
  # Create specific user configuration, based on 
  # vsftpd_virtualuser_config.tpl file.
  #
  # ... Do not change it in this script.

  #
  # Update denied_users file
  #
  if [ "$USERSTATUS" == "y" ];then
  	echo $USERNAME >> $FTPCONF/denied_users	
  else
  	sed -i -r -e "/^$USERNAME$/ d" $FTPCONF/denied_users
  fi

  #
  # Update accounts.db file.
  #
  sed -i -e "/$USERNAME/,+1 d" $ACCOUNTSDB_TMP
  echo $USERNAME >> $ACCOUNTSDB_TMP; 
  echo $PASSWORD >> $ACCOUNTSDB_TMP;
  rm -f $ACCOUNTSDB_DB
  db_load -T -t hash -f $ACCOUNTSDB_TMP $ACCOUNTSDB_DB

  #
  # Set Permissions
  #
  /bin/chmod 600 $ACCOUNTSDB_DB
  /bin/chmod -R $CHMOD $HOMEDIR/$USERNAME
  /usr/bin/chcon -R -t public_content_rw_t $HOMEDIR/$USERNAME

  #
  # Update user information
  #
  /usr/bin/chfn -f "$FULLNAME" $USERNAME 1>/dev/null

  # Restart vsftpd after user addition.
  echo '-------------------------------------------------------------------'
  /sbin/service vsftpd restart
  echo '-------------------------------------------------------------------'
  ```

  ​

- Auto remove virtual user

  ```shell
  #!/bin/sh
  # Chroot FTP with Virtual Users - Remove ftp virtual user
  #
  # Version 1.0
  #       August 1, 2005
  #       Fire Eater <LinuxRockz@gmail.com>
  #
  # Version 1.1
  #       December 14, 2008
  #       Alain Reguera Delgado <alain.reguera@gmail.com>
  #
  # Released under the GPL License- http://www.fsf.org/licensing/licenses/gpl.txt
  #
  # May 12, 2017
  # Updated by Feng Fei <feifeng@superengine.com.cn>
  # ALL RIGHTS RESERVED
  # Initialization
  #
  HOMEDIR=/var/ftp/virtual_users
  FTPCONF=/etc/vsftpd

  if [ -f $FTPCONF/accounts.tmp ];then
      ACCOUNTDB_TOTALLINES=`grep '.' -c $FTPCONF/accounts.tmp`
  else
      ACCOUNTDB_TOTALLINES=0
  fi

  function checkUser_Existence () {
      C=1;

      if [ "$ACCOUNTDB_TOTALLINES" != "0" ];then
          while [ $C -lt $ACCOUNTDB_TOTALLINES ]; do
              VALIDUSER=`sed -n -e "$C p" $FTPCONF/accounts.tmp`
              if [ "$USERNAME" == "$VALIDUSER" ];then
                  USERNAMEOK=YES
                  break;
              else
                  USERNAMEOK=NO
             fi
             let C=$C+2;
          done 
      fi
  }

  function checkUser_Homedir () {

      # Verify User's Home Directory.
      if [ -d $HOMEDIR ];then
          for i in `ls $HOMEDIR/`; do
             VALIDUSER=$i
             if [ "$USERNAME" == "$VALIDUSER" ];then
                 USERNAMEOK=YES
  	       break;
  	   else
  	       USENAMEOK=NO
             fi
          done
      fi

  }

  function removeUser () {

      # Remove user from accounts.tmp
      printf " Updating $FTPCONF/accounts.tmp file ... ";
          sed -i -e "/$USERNAME/,+1 d" $FTPCONF/accounts.tmp
      printf "done. \n"
    
      # Remove user from account.db
      printf " Updating $FTPCONF/accounts.db file ... ";
          db_load -T -t hash -f  $FTPCONF/accounts.tmp $FTPCONF/accounts.db
      printf "done. \n"
    
      # Remove user from denied_users 
      printf " Updating $FTPCONF/denied_users file ... "
          sed -i -e "/$USERNAME/ d" $FTPCONF/denied_users
      printf " done.\n"
      
      # Remove user from /etc/passwd and /etc/group. Also 
      # remove related user information.
      printf " Removing user information from the system ... ";
          /usr/sbin/userdel -r $USERNAME
      printf "done. \n"
    
      # Remove user config from /etc/vsftpd/users
      printf " Removing user config file from user config dirs ... ";
          rm -f $FTPCONF/users/$USERNAME
      printf "done. \n"
    
      # Remove user ftp dir from $HOMEDIR
      printf " Removing user ftp root dir from HOMEDIR ... ";
          rm -rf $HOMEDIR/$USERNAME
      printf "done. \n"
  }

  clear;
  echo " vsftpd 2.0.5 -> Virtual Users -> Remove User"
  echo '-------------------------------------------------------------------'

  #
  # Check dependencies
  #
  PACKISMISSING=""
  PACKDEPENDENCIES="vsftpd libdb-utils"
  for i in `echo $PACKDEPENDENCIES`; do
      /bin/rpm -q $i > /dev/null
      if [ "$?" != "0" ];then
          PACKISMISSING="$PACKISMISSING $i"
      fi
  done
  if [ "$PACKISMISSING" != "" ];then
      echo " ATTENTION: The following package(s) are needed by this script:"
      for i in `echo $PACKISMISSING`; do
          echo "             - $i"
      done
      echo '-------------------------------------------------------------------'
      exit;
  fi

  #
  # Non-interactive
  #
  if [ "$1" ];then

      for i in $1; do
      USERNAME=$i
      echo "Removing user $USERNAME: "
      checkUser_Existence;
      checkUser_Homedir;
    
      if [ "$USERNAMEOK" == "YES" ];then
      removeUser;
      echo '-------------------------------------------------------------------'
      /sbin/service vsftpd reload
      echo '-------------------------------------------------------------------'
      else
         echo "   ATTENTION : This user can't be removed. It is an invalid user."
         echo '-------------------------------------------------------------------'
      fi
      done
    
      exit;
  fi

  #
  # Interactive
  #
  printf " Enter username (lowercase): "
  read USERNAME

  checkUser_Existence;
  checkUser_Homedir;

  if [ "$USERNAMEOK" == "YES" ];then

      echo ' ****************************************************************** '
      echo " * ATTENTION: All data related to the user $USERNAME will be removed."
      echo ' ****************************************************************** '
      printf ' Are you sure ? (N/y): '
      read CONFIRMATION
    
      if [ "$CONFIRMATION" != "y" ];then
          exit;
      fi
      removeUser;
      echo '-------------------------------------------------------------------'
      /sbin/service vsftpd restart
      echo '-------------------------------------------------------------------'

  else
         echo "   ATTENTION : This user can't be removed. It is an invalid user."
         echo '-------------------------------------------------------------------'
  fi
  ```



- List all virtual users

  ```shell
  #!/bin/sh
  # Chroot FTP with Virtual Users - Information about ftp virtual users
  #
  # Version 1.0
  #       August 1, 2005
  #       Fire Eater <LinuxRockz@gmail.com>
  #
  # Version 1.1
  #       December 14, 2008
  #       Alain Reguera Delgado <alain.reguera@gmail.com>
  #
  # Released under the GPL License- http://www.fsf.org/licensing/licenses/gpl.txt
  # May 12, 2017
  # Updated by Feng Fei <feifeng@superengine.com.cn>
  # ALL RIGHTS RESERVED
  #
  # Initializations
  #
  FTPCONF=/etc/vsftpd
  HOMEDIR=/var/ftp/virtual_users
  USERCOUNT=0
  TOTALSIZE=0
  COUNTER=1

  if [ -f $FTPCONF/accounts.tmp ];then
      ACCOUNTDB_TOTALLINES=`grep '.' -c $FTPCONF/accounts.tmp`
  else
      ACCOUNTDB_TOTALLINES=0
  fi

  function checkUser_Existence () {
      C=1;

      if [ "$ACCOUNTDB_TOTALLINES" != "0" ];then
          while [ $C -lt $ACCOUNTDB_TOTALLINES ]; do
      	    VALIDUSER=`sed -n -e "$C p" $FTPCONF/accounts.tmp`
     	    if [ "$USERNAME" == "$VALIDUSER" ];then
                  USERNAMEOK=YES
                  break;
              else
                  USERNAMEOK=NO
              fi
              let C=$C+2;
           done 
      fi
  }

  function checkUser_Homedir () {

      # Verify User's Home Directory.
      if [ -d $HOMEDIR ];then
          for i in `ls $HOMEDIR/`; do
             VALIDUSER=$i
             if [ "$USERNAME" == "$VALIDUSER" ];then
                 USERNAMEOK=YES
  	       break;
  	   else
  	       USENAMEOK=NO
             fi
          done
      fi

  }

  # getUserInfo. This function retrives information related to ftp
  # virtual user. If you want to see more information about an ftp
  # virtual user, add it in this function.
  #
  function getUserInfo {

      echo "           User : $USERNAME"

      checkUser_Existence;
      checkUser_Homedir;

      if [ "$USERNAMEOK" == "YES" ];then
          SIZE=`du -sc $HOMEDIR/$USERNAME | head -n 1 | sed -r 's/\s.*$//' | cut -d' ' -f1`

          # Set if the username is DENIED or ACTIVE
          if [ `grep -w $USERNAME $FTPCONF/denied_users | head -n 1` ];then
  	    	USERSTATUS=DISABLED
  	    else
  	        USERSTATUS=AVAILABLE
          fi

          echo "           Size : $SIZE"
          echo "     Commentary : `grep $USERNAME /etc/passwd | cut -d: -f5`"
          echo " Home directory : `grep $USERNAME /etc/passwd | cut -d: -f6`"
          echo "    Login Shell : `grep $USERNAME /etc/passwd | cut -d: -f7`"
          echo "  Accout Status : $USERSTATUS"

          let USERCOUNT=$USERCOUNT+1
          let TOTALSIZE=$TOTALSIZE+$SIZE

      else

          echo "      ATTENTION : Invalid ftp virtual user."

      fi

      echo "---------------------------------------------------------------"

  }

  # showTotals.
  function showTotals {
      echo "    Total Users : $USERCOUNT"
      echo "Total Size Used : $TOTALSIZE"
  }

  #
  # Some presentation :)
  #
  clear;
  echo " vsftpd 2.0.5 -> Virtual Users -> Information "
  echo "---------------------------------------------------------------"

  #
  # Interactive
  #
  if [ "$1" ];then

      for i in $1;do

          USERNAME=$i
          getUserInfo;

      done

  showTotals;

  exit;

  fi

  #
  # Non-Interactive
  #
  while [ $COUNTER -lt $ACCOUNTDB_TOTALLINES ]; do

      USERNAME=`sed -n -e "$COUNTER p" $FTPCONF/accounts.tmp`

      getUserInfo;

      let COUNTER=$COUNTER+2;

  done 

  showTotals;
  ```

  ​

- Virtual user configuration template

  ```	shell
  # May 12, 2017
  # Updated by Feng Fei <feifeng@superengine.com.cn>
  # ALL RIGHTS RESERVED
  dirlist_enable=YES
  download_enable=YES
  local_root=/var/ftp/virtual_users/USERNAME
  write_enable=YES
  allow_writeable_chroot=YES
  ```

  ​

## Reference Links

1. Official Site:[https://security.appspot.com/vsftpd.html](https://security.appspot.com/vsftpd.html)
2. Redhat reference:[https://access.redhat.com/documentation/en-US/Red_Hat_Enterprise_Linux/6/html/Deployment_Guide/s2-ftp-servers-vsftpd.html](https://access.redhat.com/documentation/en-US/Red_Hat_Enterprise_Linux/6/html/Deployment_Guide/s2-ftp-servers-vsftpd.html)
3. CentOS setup:[https://www.liquidweb.com/kb/how-to-install-and-configure-vsftpd-on-centos-7/](https://www.liquidweb.com/kb/how-to-install-and-configure-vsftpd-on-centos-7/)


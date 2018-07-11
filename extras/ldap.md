# Configuring LDAP for use with RStudio Pro Products

*These step-by-step instructions explain how to configure Shiny Server Pro with LDAP on Ubuntu. For more information about configuring authentication with Shiny Server Pro, see the [Shiny Server admin guide](http://docs.rstudio.com/shiny-server/#ldap-and-active-directory).*

1. Point Ubuntu to the right CRAN location

```
sudo sh -c 'echo "deb http://cran.rstudio.com/bin/linux/ubuntu trusty/" >> /etc/apt/sources.list'
gpg --keyserver keyserver.ubuntu.com --recv-key E084DAB9
gpg -a --export E084DAB9 | sudo apt-key add -
```

2. Update and install R, and gdebi

```
sudo apt-get update
sudo apt-get install gdebi-core
sudo apt-get install r-base
```

3. Install the shiny package

```
sudo su - -c "R -e \"install.packages('shiny', repos='https://cran.rstudio.com/')\""
```

4. Download and install SSP

```
wget https://s3.amazonaws.com/rstudio-shiny-server-pro-build/ubuntu-14.04/x86_64/shiny-server-commercial-1.5.7.954-amd64.deb
sudo gdebi shiny-server-commercial-1.5.7.954-amd64.deb
```

5. If prompted, activate SSP

```
sudo /opt/shiny-server/bin/license-manager activate [key]
sudo restart shiny-server
```

6. Install the LDAP server. When prompted, use “admin” for the admin password

```
sudo apt install slapd ldap-utils
```

7. Setup a domain for the LDAP server. There will be several prompts, here are the selections or inputs to use for each:

```
1. Select No
2. Use example.com as the domain name
3. Use example as the company name
4. Use rstudio as the password
5. Confirm password
6. Select HDB
7. Yes
8. Yes
9. Yes
```

```
sudo dpkg-reconfigure slapd
```

8. Create the file that contains the initial user and group

```
sudo vi add_content.ldif
```

9. Use the following content to the add_content.ldif by copying and pasting

```
dn: ou=People,dc=example,dc=com
objectClass: organizationalUnit
ou: People

dn: ou=Groups,dc=example,dc=com
objectClass: organizationalUnit
ou: Groups

dn: cn=miners,ou=Groups,dc=example,dc=com
objectClass: posixGroup
cn: miners
gidNumber: 5000

dn: uid=john,ou=People,dc=example,dc=com
objectClass: inetOrgPerson
objectClass: posixAccount
objectClass: shadowAccount
uid: john
sn: Doe
givenName: John
cn: John Doe
displayName: John Doe
uidNumber: 10000
gidNumber: 5000
userPassword: johnldap
gecos: John Doe
loginShell: /bin/bash
homeDirectory: /home/john
```

10. Exit the text editor by using the Escape key, and then type :wq

11. Send the content to the LDAP server

```
ldapadd -x -D cn=admin,dc=example,dc=com -W -f add_content.ldif
```

12. Install the LDAP client

```
sudo apt install libnss-ldap
```

13. Remove the existing shiny-server.conf file

```
sudo rm /etc/shiny-server/shiny-server.conf
```

14. Add the new configuration to a new configuration file

```
sudo vi /etc/shiny-server/shiny-server.conf
```

15. Copy and paste the following content

```
# Instruct Shiny Server to run applications as the user "shiny"
run_as shiny;

# Specify the authentication method to be used.
# Initially, a flat-file database stored at the path below.

auth_ldap ldap://127.0.0.1/dc=example,dc=com {
  group_filter "memberUid={uid}";
  group_search_base ou=People;
};

# Define a server that listens on port 3838
server {
  listen 3838;

  # Define a location at the base URL
  location / {

    # Only up tp 20 connections per Shiny process and at most 3 Shiny processes
    # per application. Proactively spawn a new process when our processes reach 
    # 90% capacity.
    utilization_scheduler 20 .9 3;

    # Host the directory of Shiny Apps stored in this directory
    site_dir /srv/shiny-server;

    # Log all Shiny output to files in this directory
    log_dir /var/log/shiny-server;

    # When a user visits the base URL rather than a particular application,
    # an index of the applications available in this directory will be shown.
    directory_index on;

  location /sample-apps/hello {
     required_user john;
   }
 }
}

# Provide the admin interface on port 4151
admin 4151 {

  # Restrict the admin interface to the usernames listed here. Currently 
  # just one user named "admin"
  required_user admin;
}
```

16. Exit the text editor by using `escape` followed by `:wq`
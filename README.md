# Quick Start

## Installation

Build the gitlab image.

```bash
git clone https://github.com/jasonbrooks/docker-gitlab.git
cd docker-gitlab
docker build --tag="$USER/gitlab" .
```

Build a postgresql image.

```bash
git clone https://github.com/CentOS/CentOS-Dockerfiles.git
docker build --tag="$USER/postgres" CentOS-Dockerfiles/postgres/centos7/.
```

Build a redis image.

```bash
git clone https://github.com/CentOS/CentOS-Dockerfiles.git
docker build --tag="$USER/redis" CentOS-Dockerfiles/redis/centos7/.
```

## Startup

Start the redis container

```bash
docker run --name=redis -d $USER/redis
```

Start the postgres container

```bash
docker run --name postgresql -d \
-e 'DB_USER=gitlab' \
-e 'DB_PASS=password' \
-e 'DB_NAME=gitlab_production' \
$USER/postgres
```

Start the gitlab container

```bash
docker run --name=gitlab -d \
--link redis:redis \
--link postgresql:postgresql \
--publish 8090:80 \
$USER/gitlab
```

Monitor the setup process with `docker logs -f gitlab`. When supervisor reports that `unicorn`, `nginx`, `cron`, `sshd` and `sidekiq` are `RUNNING`, find the IP address of your gitlab container with `docker inspect gitlab | grep IP`, point your browser at this IP, and log in using the default username and password:

* username: root
* password: 5iveL!fe

You should now have the GitLab application up and ready for testing. If you want to use this image in production, then please read on. 

# Hardware Requirements

## CPU
- 1 core works for under 100 users but the responsiveness might suffer
- 2 cores is the recommended number of cores and supports up to 100 users
- 4 cores supports up to 1,000 users
- 8 cores supports up to 10,000 users

## Memory

- 512MB is too little memory, GitLab will be very slow and you will need 250MB of swap
- 768MB is the minimal memory size but we advise against this
- 1GB supports up to 100 users (with individual repositories under 250MB, otherwise git memory usage necessitates using swap space)
- **2GB** is the **recommended** memory size and supports up to 1,000 users
- 4GB supports up to 10,000 users

## Storage

The necessary hard drive space largely depends on the size of the repos you want
to store in GitLab. But as a *rule of thumb* you should have at least twice as much
free space as your all repos combined take up. You need twice the storage because [GitLab satellites](https://github.com/gitlabhq/gitlabhq/blob/master/doc/install/structure.md) contain an extra copy of each repo.

If you want to be flexible about growing your hard drive space in the future consider mounting it using LVM so you can add more hard drives when you need them.

Apart from a local hard drive you can also mount a volume that supports the network file system (NFS) protocol. This volume might be located on a file server, a network attached storage (NAS) device, a storage area network (SAN) or on an Amazon Web Services (AWS) Elastic Block Store (EBS) volume.

If you have enough RAM memory and a recent CPU the speed of GitLab is mainly limited by hard drive seek times. Having a fast drive (7200 RPM and up) or a solid state drive (SSD) will improve the responsiveness of GitLab.

# Supported Web Browsers

- Chrome (Latest stable version)
- Firefox (Latest released version)
- Safari 7+ (Know problem: required fields in html5 do not work)
- Opera (Latest released version)
- IE 10+

### Mail
The mail configuration should be specified using environment variables while starting the GitLab image. The configuration defaults to using gmail to send emails and requires the specification of a valid username and password to login to the gmail servers.

The following environment variables need to be specified to get mail support to work.

* SMTP_DOMAIN (defaults to www.gmail.com)
* SMTP_HOST (defaults to smtp.gmail.com)
* SMTP_PORT (defaults to 587)
* SMTP_USER
* SMTP_PASS
* SMTP_STARTTLS (defaults to true)
* SMTP_AUTHENTICATION (defaults to 'login' if SMTP_USER is set)

```bash
docker run --name=gitlab -d \
  -e 'SMTP_USER=USER@gmail.com' -e 'SMTP_PASS=PASSWORD' \
  -v /opt/gitlab/data:/home/git/data \
  $USER/gitlab
```

### SSL
Access to the gitlab application can be secured using SSL so as to prevent unauthorized access to the data in your repositories. While a CA certified SSL certificate allows for verification of trust via the CA, a self signed certificates can also provide an equal level of trust verification as long as each client takes some additional steps to verify the identity of your website. I will provide instructions on achieving this towards the end of this section.

To secure your application via SSL you basically need two things:
- Private key (.key)
- SSL certificate (.crt)

When using CA certified certificates, these files are provided to you by the CA. When using self-signed certificates you need to generate these files yourself. Skip the following section if you are armed with CA certified SSL certificates.

Jump to the [Strengthening the server security](#strengthening-the-server-security) section if you are using a load balancer such as hipache, haproxy or nginx.

#### Generation of Self Signed Certificates
Generation of self-signed SSL certificates involves a simple 3 step procedure.

**STEP 1**: Create the server private key

```bash
openssl genrsa -out gitlab.key 2048
```

**STEP 2**: Create the certificate signing request (CSR)

```bash
openssl req -new -key gitlab.key -out gitlab.csr
```

**STEP 3**: Sign the certificate using the private key and CSR

```bash
openssl x509 -req -days 365 -in gitlab.csr -signkey gitlab.key -out gitlab.crt
```

Congratulations! you have now generated an SSL certificate thats valid for 365 days.

#### Strengthening the server security
This section provides you with instructions to [strengthen your server security](https://raymii.org/s/tutorials/Strong_SSL_Security_On_nginx.html). To achieve this we need to generate stronger DHE parameters.

```bash
openssl dhparam -out dhparam.pem 2048
```

#### Installation of the SSL Certificates
Out of the four files generated above, we need to install the gitlab.key, gitlab.crt and dhparam.pem files at the gitlab server. The CSR file is not needed, but do make sure you safely backup the file (in case you ever need it again).

The default path that the gitlab application is configured to look for the SSL certificates is at /home/git/data/certs, this can however be changed using the SSL_KEY_PATH, SSL_CERTIFICATE_PATH and SSL_DHPARAM_PATH configuration options.

If you remember from above, the /home/git/data path is the path of the [data store](#data-store), which means that we have to create a folder named certs inside /opt/gitlab/data/ and copy the files into it and as a measure of security we will update the permission on the gitlab.key file to only be readable by the owner.

```bash
mkdir -p /opt/gitlab/data/certs
cp gitlab.key /opt/gitlab/data/certs/
cp gitlab.crt /opt/gitlab/data/certs/
cp dhparam.pem /opt/gitlab/data/certs/
chmod 400 /opt/gitlab/data/certs/gitlab.key
```

Great! we are now just a step away from having our application secured.

#### Enabling HTTPS support
HTTPS support can be enabled by setting the GITLAB_HTTPS option to true. Additionally, when using self-signed SSL certificates you need to the set SSL_SELF_SIGNED option to true as well. Assuming we are using self-signed certificates

```bash
docker run --name=gitlab -d \
  -e 'GITLAB_HTTPS=true' -e 'SSL_SELF_SIGNED=true' \
  -v /opt/gitlab/data:/home/git/data \
  $USER/gitlab
```

In this configuration, any requests made over the plain http protocol will automatically be redirected to use the https protocol. However, this is not optimal when using a load balancer.

#### Using HTTPS with a load balancer
Load balancers like haproxy/hipache talk to backend applications over plain http and as such, installation of ssl keys and certificates in the container are not required when using a load balancer.

When using a load balancer, you should set the GITLAB_HTTPS_ONLY option to false with the GITLAB_HTTPS options set to true and the SSL_SELF_SIGNED option to the appropriate value. With this in place, you should also configure the load balancer to support handling of https requests. But that is out of the scope of this document. Please refer to [Using SSL/HTTPS with HAProxy](http://seanmcgary.com/posts/using-sslhttps-with-haproxy) for information on the subject.

Note that when the GITLAB_HTTPS_ONLY is disabled, the application does not perform the automatic http to https redirection and this functionality has to be configured at the load balancer which is also described in the link above. Unfortunately hipache does not come with an option to perform http to https redirection, so the only choice you really have is to switch to using haproxy or nginx for load balancing.

In summation, the docker command would look something like this:

```bash
docker run --name=gitlab -d \
  -e 'GITLAB_HTTPS=true' -e 'SSL_SELF_SIGNED=true' \
  -e 'GITLAB_HTTPS_ONLY=false' \
  -v /opt/gitlab/data:/home/git/data \
  $USER/gitlab
```

Again, drop the `-e 'SSL_SELF_SIGNED=true'` option if you are using CA certified SSL certificates.

#### Establishing trust with your server
This section deals will self-signed ssl certificates. If you are using CA certified certificates, your done.

This section is more of a client side configuration so as to add a level of confidence at the client to be 100 percent sure they are communicating with whom they think they.

This is simply done by adding the servers certificate into their list of trusted ceritficates. On ubuntu, this is done by copying the `gitlab.crt` file to `/usr/local/share/ca-certificates/` and executing `update-ca-certificates`.

Again, this is a client side configuration which means that everyone who is going to communicate with the server should perform this configuration on their machine. In short, distribute the gitlab.crt file among your developers and ask them to add it to their list of trusted ssl certificates. Failure to do so will result in errors that look like this:

```bash
git clone https://git.local.host/gitlab-ce.git
fatal: unable to access 'https://git.local.host/gitlab-ce.git': server certificate verification failed. CAfile: /etc/ssl/certs/ca-certificates.crt CRLfile: none
```

You can do the same at the web browser. Instructions for installing the root certificate for firefox can be found [here](http://portal.threatpulse.com/docs/sol/Content/03Solutions/ManagePolicy/SSL/ssl_firefox_cert_ta.htm). You will find similar options chrome, just make sure you install the certificate under the authorities tab of the certificate manager dialog.

There you have it, thats all there is to it.

#### Installing Trusted SSL Server Certificates
If your GitLab CI server is using self-signed SSL certificates then you should make sure the GitLab CI server certificate is trusted on the GitLab server for them to be able to talk to each other.

The default path image is configured to look for the trusted SSL certificates is at /home/git/data/certs/ca.crt, this can however be changed using the CA_CERTIFICATES_PATH configuration option.

Copy the ca.crt file into the certs directory on the [datastore](#data-store). The ca.crt file should contain the root certificates of all the servers you want to trust. With respect to GitLab CI, this will be the contents of the gitlab_ci.crt file as described in the [README](https://github.com/sameersbn/docker-gitlab-ci/blob/master/README.md#ssl) of the [docker-gitlab-ci](https://github.com/sameersbn/docker-gitlab-ci) container.

By default, our own server certificate [gitlab.crt](#generation-of-self-signed-certificates) is added to the trusted certificates list.

### Run under sub URI
If you like to serve the GitLab under sub URI like http://localhost/gitlab, set GITLAB_RELATIVE_URL_ROOT=/gitlab or anything you like.
The path should start with slash, and should not have any trailing slashes.

```bash
docker run --name=gitlab -d \
  -v /opt/gitlab/data:/home/git/data \
  -e 'GITLAB_RELATIVE_URL_ROOT=/gitlab' \
  $USER/gitlab
```

When you change the sub URI path, you need to recompile all precompiled assets. This can be done with either deleting tmp/cache/VERSION file under data store, or just `rm -Rf /PATH/TO/DATA_STORE/tmp`. After cleaning up cache files, restart the container.

### OmniAuth Integration

GitLab leverages OmniAuth to allow users to sign in using Twitter, GitHub, and other popular services. Configuring OmniAuth does not prevent standard GitLab authentication or LDAP (if configured) from continuing to work. Users can choose to sign in using any of the configured mechanisms.

Refer to the GitLab [documentation](http://doc.gitlab.com/ce/integration/omniauth.html) for additional information.

#### Google

To enable the Google OAuth2 OmniAuth provider you must register your application with Google. Google will generate a client ID and secret key for you to use. Please refer to the GitLab [documentation](http://doc.gitlab.com/ce/integration/google.html) for the procedure to generate the client ID and secret key with google.

Once you have the client ID and secret keys generated, configure them using the `OAUTH_GOOGLE_API_KEY` and `OAUTH_GOOGLE_APP_SECRET` environment variables respectively.

For example, if your client ID is `xxx.apps.googleusercontent.com` and client secret key is `yyy`, then adding `-e 'OAUTH_GOOGLE_API_KEY=xxx.apps.googleusercontent.com' -e 'OAUTH_GOOGLE_APP_SECRET=yyy'` to the docker run command enables support for Google OAuth.

#### Twitter

To enable the Twitter OAuth2 OmniAuth provider you must register your application with Twitter. Twitter will generate a API key and secret for you to use. Please refer to the GitLab [documentation](http://doc.gitlab.com/ce/integration/twitter.html) for the procedure to generate the API key and secret with twitter.

Once you have the API key and secret generated, configure them using the `OAUTH_TWITTER_API_KEY` and `OAUTH_TWITTER_APP_SECRET` environment variables respectively.

For example, if your API key is `xxx` and the API secret key is `yyy`, then adding `-e 'OAUTH_TWITTER_API_KEY=xxx' -e 'OAUTH_TWITTER_APP_SECRET=yyy'` to the docker run command enables support for Twitter OAuth.

#### GitHub

To enable the GitHub OAuth2 OmniAuth provider you must register your application with GitHub. GitHub will generate a Client ID and secret for you to use. Please refer to the GitLab [documentation](http://doc.gitlab.com/ce/integration/github.html) for the procedure to generate the Client ID and secret with github.

Once you have the Client ID and secret generated, configure them using the `OAUTH_GITHUB_API_KEY` and `OAUTH_GITHUB_APP_SECRET` environment variables respectively.

For example, if your Client ID is `xxx` and the Client secret is `yyy`, then adding `-e 'OAUTH_GITHUB_API_KEY=xxx' -e 'OAUTH_GITHUB_APP_SECRET=yyy'` to the docker run command enables support for GitHub OAuth.

### External Issue Trackers

GitLab can be configured to use third party issue trackers such as Redmine and Atlassian Jira. Use of third party issue trackers have to be configured on a per project basis from the project settings page. This means that the GitLab's issue tracker is always the default tracker unless specified otherwise.

#### Redmine

Support for issue tracking using Redmine can be added by specifying the complete URL of the Redmine web server in the `REDMINE_URL` configuration option.

For example, if your Redmine server is accessible at `https://redmine.example.com`, then adding `-e 'REDMINE_URL=https://redmine.example.com'` to the docker run command enables Redmine support in GitLab

#### Jira

Support for issue tracking using Jira can be added by specifying the complete URL of the Jira web server in the `JIRA_URL` configuration option.

For example, if your Jira server is accessible at `https://jira.example.com`, then adding `-e 'JIRA_URL=https://jira.example.com'` to the docker run command enables Jira support in GitLab

### Available Configuration Parameters

*Please refer the docker run command options for the `--env-file` flag where you can specify all required environment variables in a single file. This will save you from writing a potentially long docker run command.*

Below is the complete list of available options that can be used to customize your gitlab installation.

- **GITLAB_HOST**: The hostname of the GitLab server. Defaults to localhost
- **GITLAB_PORT**: The port of the GitLab server. Defaults to 80 for plain http and 443 when https is enabled.
- **GITLAB_EMAIL**: The email address for the GitLab server.  Defaults to example@example.com.
- **GITLAB_SIGNUP**: Enable or disable user signups. Default is false.
- **GITLAB_SIGNIN**: If set to false, standard login form won't be shown on the sign-in page. Default is true.
- **GITLAB_PROJECTS_LIMIT**: Set default projects limit. Defaults to 100.
- **GITLAB_PROJECTS_VISIBILITY**: Set default projects visibility level. Possible values 'public', 'private' and 'internal'. Defaults to 'private'.
- **GITLAB_RESTRICTED_VISIBILITY**: Comma seperated list of visibility levels to restrict non-admin users to set. Possible visibility options are public, private and internal.
- **GITLAB_BACKUPS**: Setup cron job to automatic backups. Possible values disable, daily or monthly. Disabled by default
- **GITLAB_BACKUP_EXPIRY**: Configure how long to keep backups before they are deleted. By default when automated backups are disabled backups are kept forever (0 seconds), else the backups expire in 7 days (604800 seconds).
- **GITLAB_SSH_PORT**: The ssh port number. Defaults to 22.
- **GITLAB_RELATIVE_URL_ROOT**: The sub URI of the GitLab server, e.g. /gitlab. No default.
- **GITLAB_HTTPS**: Set to true to enable https support, disabled by default.
- **GITLAB_HTTPS_ONLY**: Configure access over plain http when GITLAB_HTTPS is enabled. Should be set to false when using a load balancer. Defaults to true.
- **SSL_SELF_SIGNED**: Set to true when using self signed ssl certificates. false by default.
- **SSL_CERTIFICATE_PATH**: Location of the ssl certificate. Defaults to /home/git/data/certs/gitlab.crt
- **SSL_KEY_PATH**: Location of the ssl private key. Defaults to /home/git/data/certs/gitlab.key
- **SSL_DHPARAM_PATH**: Location of the dhparam file. Defaults to /home/git/data/certs/dhparam.pem
- **CA_CERTIFICATES_PATH**: List of SSL certificates to trust. Defaults to /home/git/data/certs/ca.crt.
- **NGINX_MAX_UPLOAD_SIZE**: Maximum acceptable upload size. Defaults to 20m.
- **REDIS_HOST**: The hostname of the redis server. Defaults to localhost
- **REDIS_PORT**: The connection port of the redis server. Defaults to 6379.
- **UNICORN_WORKERS**: The number of unicorn workers to start. Defaults to 2.
- **UNICORN_TIMEOUT**: Sets the timeout of unicorn worker processes. Defaults to 60 seconds.
- **SIDEKIQ_CONCURRENCY**: The number of concurrent sidekiq jobs to run. Defaults to 5
- **DB_TYPE**: The database type. Possible values: mysql, postgres. Defaults to mysql.
- **DB_HOST**: The database server hostname. Defaults to localhost.
- **DB_PORT**: The database server port. Defaults to 3306 for mysql and 5432 for postgresql.
- **DB_NAME**: The database database name. Defaults to gitlabhq_production
- **DB_USER**: The database database user. Defaults to root
- **DB_PASS**: The database database password. Defaults to no password
- **DB_POOL**: The database database connection pool count. Defaults to 10.
- **SMTP_DOMAIN**: SMTP domain. Defaults to www.gmail.com
- **SMTP_HOST**: SMTP server host. Defaults to smtp.gmail.com.
- **SMTP_PORT**: SMTP server port. Defaults to 587.
- **SMTP_USER**: SMTP username.
- **SMTP_PASS**: SMTP password.
- **SMTP_STARTTLS**: Enable STARTTLS. Defaults to true.
- **SMTP_AUTHENTICATION**: Specify the SMTP authentication method. Defaults to 'login' if SMTP_USER is set.
- **LDAP_ENABLED**: Enable LDAP. Defaults to false
- **LDAP_HOST**: LDAP Host
- **LDAP_PORT**: LDAP Port. Defaults to 636
- **LDAP_UID**: LDAP UID. Defaults to sAMAccountName
- **LDAP_METHOD**: LDAP method, Possible values are ssl, tls and plain. Defaults to ssl
- **LDAP_BIND_DN**: No default.
- **LDAP_PASS**: LDAP password
- **LDAP_ALLOW_USERNAME_OR_EMAIL_LOGIN**: If enabled, GitLab will ignore everything after the first '@' in the LDAP username submitted by the user on login. Defaults to false if LDAP_UID is userPrincipalName, else true.
- **LDAP_BASE**: Base where we can search for users. No default.
- **LDAP_USER_FILTER**: Filter LDAP users. No default.
- **OAUTH_ALLOW_SSO**: This allows users to login without having a user account first. User accounts will be created automatically when authentication was successful. Defaults to false.
- **OAUTH_BLOCK_AUTO_CREATED_USERS**: Locks down those users until they have been cleared by the admin. Defaults to true.
- **OAUTH_GOOGLE_API_KEY**: Google App Client ID. No defaults.
- **OAUTH_GOOGLE_APP_SECRET**: Google App Client Secret. No defaults.
- **OAUTH_TWITTER_API_KEY**: Twitter App API key. No defaults.
- **OAUTH_TWITTER_APP_SECRET**: Twitter App API secret. No defaults.
- **OAUTH_GITHUB_API_KEY**: GitHub App Client ID. No defaults.
- **OAUTH_GITHUB_APP_SECRET**: GitHub App Client secret. No defaults.
- **REDMINE_URL**: Location of the redmine server, e.g. `-e 'REDMINE_URL=https://redmine.example.com'`. No defaults.
- **JIRA_URL**: Location of the jira server, e.g. `-e 'JIRA_URL=https://jira.example.com'`. No defaults.

# Maintenance

## Creating backups

Gitlab defines a rake task to easily take a backup of your gitlab installation. The backup consists of all git repositories, uploaded files and as you might expect, the sql database.

Before taking a backup, please make sure that the gitlab image is not running for obvious reasons

```bash
docker stop gitlab
```

To take a backup all you need to do is run the gitlab rake task to create a backup.

```bash
docker run --name=gitlab -it --rm [OPTIONS] \
  $USER/gitlab app:rake gitlab:backup:create
```

A backup will be created in the backups folder of the [Data Store](#data-store)

## Restoring Backups

Gitlab defines a rake task to easily restore a backup of your gitlab installation. Before performing the restore operation please make sure that the gitlab image is not running.

```bash
docker stop gitlab
```

To restore a backup, run the image in interactive (-it) mode and pass the "app:restore" command to the container image.

```bash
docker run --name=gitlab -it --rm [OPTIONS] \
  $USER/gitlab app:rake gitlab:backup:restore
```

The restore operation will list all available backups in reverse chronological order. Select the backup you want to restore and gitlab will do its job.

## Automated Backups

The image can be configured to automatically take backups on a daily or monthly basis. Adding `-e 'GITLAB_BACKUPS=daily'` to the docker run command will enable daily backups, while `-e 'GITLAB_BACKUPS=monthly'` will enable monthly backups.

Daily backups are created at 4 am (UTC) everyday, while monthly backups are created on the 1st of every month at the same time as the daily backups.

By default, when automated backups are enabled, backups are held for a period of 7 days. While when automated backups are disabled, the backups are held for an infinite period of time. This can behaviour can be configured via the GITLAB_BACKUP_EXPIRY option.

## Shell Access

For debugging and maintenance purposes you may want access the container shell. Since the container does not allow interactive login over the SSH protocol, you can use the [nsenter](http://man7.org/linux/man-pages/man1/nsenter.1.html) linux tool (part of the util-linux package) to access the container shell.

Some linux distros (e.g. ubuntu) use older versions of the util-linux which do not include the `nsenter` tool. To get around this @jpetazzo has created a nice docker image that allows you to install the `nsenter` utility and a helper script named `docker-enter` on these distros.

To install the nsenter tool on your host execute the following command.

```bash
docker run --rm -v /usr/local/bin:/target jpetazzo/nsenter
```

Now you can access the container shell using the command

```bash
sudo docker-enter gitlab
```

For more information refer https://github.com/jpetazzo/nsenter

Another tool named `nsinit` can also be used for the same purpose. Please refer https://jpetazzo.github.io/2014/03/23/lxc-attach-nsinit-nsenter-docker-0-9/ for more information.

# Upgrading

GitLabHQ releases new versions on the 22nd of every month, bugfix releases immediately follow. 

To upgrade to newer gitlab releases, follow this 5 step upgrade procedure.

- **Step 1**: Rev the "GITLAB_VERSION" value in the file `assets/setup/install`, or fetch updates from this repo to pick up this edit.

- **Step 2**: Rebuild your docker-gitlab image.

```bash
cd docker-gitlab
docker build --tag="$USER/gitlab" .
```

- **Step 3**: Stop and remove the currently running image

```bash
docker stop gitlab
docker rm gitlab
```

- **Step 4**: Backup the application data.

```bash
docker run --name=gitlab -it --rm [OPTIONS] \
  $USER/gitlab app:rake gitlab:backup:create
```

- **Step 5**: Start the image

```bash
docker run --name=gitlab -d [OPTIONS] $USER/gitlab
```

## Rake Tasks

The app:rake command allows you to run gitlab rake tasks. To run a rake task simply specify the task to be executed to the app:rake command. For example, if you want to gather information about GitLab and the system it runs on.

```bash
docker run --name=gitlab -d [OPTIONS] \
  $USER/gitlab app:rake gitlab:env:info
```

Similarly, to import bare repositories into GitLab project instance

```bash
docker run --name=gitlab -d [OPTIONS] \
  $USER/gitlab app:rake gitlab:import:repos
```

For a complete list of available rake tasks please refer https://github.com/gitlabhq/gitlabhq/tree/master/doc/raketasks or the help section of your gitlab installation.

# References
  * https://github.com/sameersbn/docker-gitlab
  * https://github.com/gitlabhq/gitlabhq
  * https://github.com/gitlabhq/gitlabhq/blob/master/doc/install/installation.md
  * https://github.com/gitlabhq/gitlabhq/blob/master/doc/install/requirements.md
  * http://wiki.nginx.org/HttpSslModule
  * https://raymii.org/s/tutorials/Strong_SSL_Security_On_nginx.html
  * https://github.com/gitlabhq/gitlab-recipes/blob/master/web-server/nginx/gitlab-ssl
  * https://github.com/jpetazzo/nsenter
  * https://jpetazzo.github.io/2014/03/23/lxc-attach-nsinit-nsenter-docker-0-9/

# Setting UP Embedded Discovery Service (EDS) on the same Jagger installation (Federation Registory) node.

1. Install dependancies to overcome issues with libcurl libraries. ( Credits: [Josh L.'s Blog](https://depts.washington.edu/bitblog/2018/06/libcurl3-libcurl4-shibboleth-php-curl-ubuntu-18-04/) )
   * ```bash
     apt install liblog4shib1v5 libxerces-c3.2 libxml-security-c17v5 libcurl3
     cp /usr/lib/x86_64-linux-gnu/libcurl.so.4.5.0 /usr/lib/x86_64-linux-gnu/libcurl3.so.4.5.0
     apt-get install libcurl4
     mkdir ~/temp
     cd ~/temp
     apt-get download libxmltooling7 # Ignore the warnings
     ar x libxmltooling7_1.6.4-1ubuntu2.1_amd64.deb
     tar xf control.tar.xz
     sed -i -e 's/libcurl3 (>= 7.16.2)/libcurl4/g' control
     tar -cJvf control.tar.xz control md5sums shlibs triggers
     ar rcs libxmltooling-local.deb debian-binary control.tar.xz data.tar.xz
     dpkg -i libxmltooling-local.deb
     mkdir /etc/systemd/system/shibd.service.d
     ```
   * Create the following script to override defaults,
   * `vim /etc/systemd/system/shibd.service.d/override.conf`
   
     ```bash
     [Service]
     Environment="LD_PRELOAD=libcurl3.so.4.5.0"
     ```
   
2. Install Shibboleth SP:
   * ```bash
     apt install libapache2-mod-shib2 libapache2-mod-php
     ```

   From this point the location of the SP directory is: ```/etc/shibboleth```
   
3. Enable shibboleth SP for  Embeded Discovery Server over SSL (Letsencypt)

   Create a seperate virtual host  `/etc/apache2/sites-available/eds.conf` with

```bash
<VirtualHost *:80>
  
        ServerName fds.YOUR-DOMAIN
        ServerAdmin admin[AT]YOUR-DOMAIN
        DocumentRoot /var/www/html

        ErrorLog ${APACHE_LOG_DIR}/error.log
        CustomLog ${APACHE_LOG_DIR}/access.log combined

        RewriteEngine on
        RewriteCond %{SERVER_NAME} =fds.YOUR-DOMAIN
        RewriteRule ^ https://%{SERVER_NAME}%{REQUEST_URI} [END,NE,R=permanent]
</VirtualHost>
```

   Run,

```bash
a2ensite eds
certbot --apache -d fds.YOUR-DOMAIN
```

   answer all the questions as you did for previous installations
   
   Edit `/etc/apache2/sites-available/eds-le-ssl.conf` with
   
```bash
<IfModule mod_ssl.c>
<VirtualHost *:443>

        ServerName fds.YOUR-DOMAIN
        ServerAdmin admin[AT]YOUR-DOMAIN
        #DocumentRoot /var/www/html
        DocumentRoot /etc/shibboleth-ds

        ErrorLog ${APACHE_LOG_DIR}/error.log
        CustomLog ${APACHE_LOG_DIR}/access.log combined

        #RewriteEngine on
# Some rewrite rules in this file were disabled on your HTTPS site,
# because they have the potential to create redirection loops.

#       RewriteCond %{SERVER_NAME} =fds.YOUR-DOMAIN
#       RewriteRule ^ https://%{SERVER_NAME}%{REQUEST_URI} [END,NE,R=permanent]

<IfModule mod_alias.c>
  <Location />
    Require all granted
    <IfModule mod_shib.c>
      AuthType shibboleth
      ShibRequestSetting requireSession false
      require shibboleth
    </IfModule>
  </Location>
</IfModule>


SSLCertificateFile /etc/letsencrypt/live/fds.YOUR-DOMAIN/fullchain.pem
SSLCertificateKeyFile /etc/letsencrypt/live/fds.YOUR-DOMAIN/privkey.pem
Include /etc/letsencrypt/options-ssl-apache.conf
</VirtualHost>
</IfModule>
```

4. Edit `/etc/shibboleth/shibboleth2.xml` opportunely:

```bash
...
<ApplicationDefaults entityID="https://fds.YOUR-DOMAIN/shibboleth"
     REMOTE_USER="eppn persistent-id targeted-id">
...
<Sessions lifetime="28800" timeout="3600" checkAddress="false" handlerSSL="true" cookieProps="https">
...
<SSO discoveryProtocol="SAMLDS" discoveryURL="https://fds.YOUR-DOMAIN/index.html" isDefault="true">
   SAML2
</SSO>
<!-- SAML and local-only logout. -->
<Logout>SAML2 Local</Logout>
...
<!-- JSON feed of discovery information. -->
<Handler type="DiscoveryFeed" Location="/DiscoFeed"/
...
   	 <MetadataProvider type="XML" validate="true"
              uri="https://fr.YOUR-DOMAIN/metadata/federation/Your-Federation/metadata.xml"
              backingFilePath="federation-metadata.xml" legacyOrgNames="true" reloadInterval="7200">
            <MetadataFilter type="RequireValidUntil" maxValidityInterval="2419200"/>
            

        </MetadataProvider>
```

5. Enable Shibboleth Apache2 configuration:

```bash
a2enmod shib2
systemctl reload apache2.service 
systemctl restart shibd
```
6. Install Shobboleth EDS

```bash
cd /usr/local/src

wget https://shibboleth.net/downloads/embedded-discovery-service/1.2.1/shibboleth-embedded-ds-1.2.1.tar.gz -O shibboleth-eds.tar.gz

tar xzf shibboleth-eds.tar.gz

cd shibboleth-embedded-ds-1.2.1

sudo apt install make ; make install

systemctl reload apache2.service 
systemctl restart shibd
```


7. Now you are able to reach your Shibboleth SP Metadata on:
   
   `https://fds.YOUR-DOMAIN/Shibboleth.sso/Metadata` (change fds.YOUR-DOMAIN to you SP full qualified domain name)


### Registering EDS on NREN Federation

Login to Federation web portal to register EDS as service provider. Wait until Federation admin accept your submission


#### Temporary Enabling EDS Test GUI

Edit `etc/shibboleth-ds/idpselect_config.js` to change

 ```this.testGUI = true;```

You may disable above after a test

### Customization
#### Add a Login Button
Edit `/var/www/html/index.php`

```html
<html>
   <head>
     <title>Test Provider - LIAF</title>
     <meta name="GENERATOR" content="Quanta Plus">
     <meta http-equiv="Content-Type" content="text/html; charset=utf-8">
     <style>
.button {
   background-color: #4CAF50; /* Green */
   border: none;
   color: white;
   padding: 15px 32px;
   text-align: center;
   text-decoration: none;
   display: inline-block;
   font-size: 16px;
   margin: 4px 2px;
   cursor: pointer;
   -webkit-transition-duration: 0.4s; /* Safari */
   transition-duration: 0.4s;
}
.buttonsignout:hover {
   box-shadow: 0 12px 16px 0 rgba(0,0,0,0.24),0 17px 50px 0
rgba(0,0,0,0.19);
}

</style>
   </head>
<body>

<h1> Test Service Provider</h1>
Log-in in to check your Attributes.
<p>
<a href="https://sp-1.learn.ac.lk/secure"><button class="button
buttonsignout">Sign In</button></a>
</p>
</body>
</html>
```

### Errors
If you got a *Return Parameter is not whitelisted*, comment out the line in `/etc/shibboleth-ds/idpselect_config.js` in your EDS

```
//    this.returnWhiteList = [ "^https:\/\/example\.org\/Shibboleth\.sso\/Login.*$" , "^https:\/\/example\.com\/Shibboleth\.sso\/Login.*$" ];
```

Restart shibd and apache2

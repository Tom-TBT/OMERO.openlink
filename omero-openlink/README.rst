OMERO.openlink
==============

An OMERO.web plugin that creates openly accessible links (URLS for raw files) to your data in OMERO and a batch file to download the data with 'curl'.

Main application:

* Bundle data of different groups/projects/datasets
* Fast web download
* Data sharing via link

Requirements
============
- OMERO.web 5.6.0 or newer
- nginx server configurations
- OMERO.web and OMERO.server are running on the same server

Installation
============

1. Install the app in omero-web python venv:

::

    $pip install -U omero-openlink

2. Create a directory where openlinks will be created. Must be read-write for both \
system users omero-server and omero-web while nginx user must have read-only access.

::

    # Create what is refered to later on as the **OPENLINK_DIR**
    $sudo mkdir /some/path/openlink
    # Set the ownership to the group 'omero'
    $sudo chown -R root:omero /some/path/openlink
    # Set the permission to the group so that users in omero can read and write
    $sudo chmod 774 /some/path/openlink
    # Add omero-server and omero-web to the group omero (if they aren't there already)
    $sudo usermod -aG omero omero-web
    $sudo usermod -aG omero omero-server

3. Add openlink app to your installed web apps:
::

    $omero config append omero.web.apps '"omero_openlink"'

4. Display the OpenLink pane in the right pane

::

    $omero config append omero.web.ui.right_plugins '["OpenLink", "omero_openlink/webclient_plugins/right_plugin.openlink.js.html", "openlink_tab"]'

5. Set other openlink configurations

::

    # path of prepared OPENLINK_DIR, here as eaxmple */storage/openlink*
    $omero config set omero.web.openlink.dir '/storage/openlink'

    # set the url alias of your OMERO.web server without leading http:// but with the openlink nginx location (set in the nginx config bellow)
    $omero config set omero.web.openlink.servername 'demo.openmicroscopy.org/openlink'

    # http or https
    $omero config set omero.web.openlink.type_http 'https'


6. Edit and upload the script to create Open Links from OMERO.web

::

    $pip show omero-openlink  # shows the installation path of omero-openlink
    $cd .../venv3/lib/python3.6/site-packages/.../omero_openlink/scripts  # cd to the scripts directory
    # edit the following in Create_OpenLink.py


    OPENLINK_DIR= "/path/to/open_link_dir"  # Directory for links that the nginx server also has access to
    SERVER_NAME = "omero-data.myfacility.com"  # name of nginx website
    TYPE_HTTP="https"  # type of hypertext transfer protocol (http or https)
    ADMIN_EMAIL = "myemail@yourfacilitydomain"  # email originator
    LENGTH_HASH = 12  # length of hash string used in the openlink url

    $omero script upload ./omero/util_scripts/Create_OpenLink.py --official # Upload the configured script

1. Add a configuration for nginx (This section assumes that an you use an nginx server.)

For the configuration you have to reuse the specified values for `SERVER_NAME` and `OPENLINK_DIR`.
Specify the URL under which the data should be accessible:

::

    SERVERNAME/SUBGROUP # this could be data.myorg.de/openlink

You can configure your nginx in two way's:

*Option 1:*
Add a new location to your nginx configuration file (etc/nginx/conf.d/omeroweb.conf) like:

::

    location  /openlink {
            proxy_read_timeout 36000;  # 10 hours
            limit_rate 10000M;  # 10 GByte
            gzip on;
            gzip_min_length 10240;
            disable_symlinks off;  # enable symlinks
            autoindex on;
            autoindex_format html; # html, xml, json, or jsonp
            autoindex_exact_size off; # on off
            autoindex_localtime on; # on off  (UTC)
            alias <YOUR OPENLINK_DIR>;  # the links will be created here
    }


*Option 2:*
Or create a new website for nginx by create a new file (e.g. openlink.conf) in /etc/nginx/conf.d/ with:

::

    server {
        listen 80;
        server_name <YOUR SERVER_NAME>;  # url alias to this nginx site

        location /openlink {

            proxy_read_timeout 36000;  # 10 hours
            limit_rate 10000M;  # 10 GByte
            gzip on;
            gzip_min_length 10240;
            disable_symlinks off;  # enable symlinks
            autoindex on;
            autoindex_format html; # html, xml, json, or jsonp
            autoindex_exact_size off; # on off
            autoindex_localtime on; # on off  (UTC)
            alias <YOUR OPENLINK_DIR>;  # the links will be created here
        }
    }

*Note:* To use a special style (like the example in *scripts/nginx/autoIndexStyle.xslt*) for your openlink data representation,
please copy the style file to */etc/nginx* and use the following configuration:

::

    autoindex_format  xml;
    xslt_stylesheet /etc/nginx/autoindexStyle.xslt       path="$uri" schema="$scheme" host="$host";

8. Add a index.html to your OPENLINK_DIR to server to avoid listing all openlinks
If a user navigates to a URL that corresponds to a directory on the server, NGINX looks for an index file to serve. By default, this is usually *index.html*. If this file is present, NGINX will serve its contents instead of displaying a directory listing. It is recommendet to put such a *index.html* in the **OPENLINK_DIR** to avoid the listing of all created openlink data.

Example for *index.html*

::

    <!DOCTYPE html>
    <html lang="de">
      <head>
        <meta charset="utf-8" />
        <meta name="viewport" content="width=device-width, initial-scale=1.0" />
        <title>Omero Downloads</title>
      </head>
      <body>
      <a href="https://<YOUR SERVER_NAME>/openlink">Please go first to the Omero-System to create DownloadLinks!</a>
      </body>
    </html>

9. Test & Troubleshoot. If SELinux is active, it would prevent nginx from following symlinks and the download of these files

Validation
==========

Validation of configuration in *Create_OpenLink.py*
----------------------------------------------------
In order to check whether the values for x have been entered correctly, please test the link that was entered in the log file under URL and also check the entered url's in the batch_download.curl that is available there.

Validation of configuration *omero-openlink*
--------------------------------------------
There is a debug output available for the plugin. Go to subdirectory omero_openlink of the installation directory of *omero-openlink*

::

    $ cd omero-openlink/omero_openlink

open the *urls.py* and delete the leading # in the line

::

    #url(r'^debugoutput/$',views.debugoutput,name='debugoutput'),

After restarting the web server, find the debug output for your Openlink plugin by replacing webclient by oemro_openlink/debugoutput in the URL of the omero.web
(for example: https://server.openmicroscopy.org/webclient -> https://server.openmicroscopy.org/omero_openlink/debugoutput). This output shows you:

 * what is defined under OPENLINK_DIR, SERVER_NAME
 * check if OPENLINK_DIR is accessible
 * check permission of OPENLINK_DIR for omero-web user
 * overview of OpenLink Areas of currently logged-in user


License
==========

OMERO.openlink is released under the AGPL.






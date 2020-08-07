## This is a bare minimum OTA server for Lineage

## Quickstart
1. Create a configuration file named `.lineageupdaterrc` in the home
   directory of the web server user, usually `/var/www`.  This is a
   simple key=value file.  Required keys are:
   * _directory_: the local directory where OTA zip files are stored.
   * _baseurl_: the web accessible URL to the same directory.

2. Create a rewrite rule on the web server to translate the "pretty"
   path to CGI parameters.  This works for lighttpd:

       $HTTP["host"] == "ota.example.com" {
         url.rewrite = (
           "^/api/v1/([^/]+)/([^/]+)/([^/]+)$" => "/cgi-bin/lineage-updater?device=$1&type=$2&incr=$3"
         )
       }

## Usage
The script expects OTA files to be named as follows:

Full OTA files:
* _project_\__device_-ota-_version_-_buildtype_-_incremental_.zip

Incremental OTA files:
* _project_\__device_-ota-_version_-_buildtype_-_base_\__incremental_.zip

Where:
* _project_ is the project/ROM name, eg. "lineage".
* _device_ is the device name, eg. "mako".
* _version_ is the project/ROM version, eg. "16.0".
* _buildtype_ is the buildtype, eg. "unofficial".
* _base_ is the base incremental version (the starting build).
* _incremental_ is the incremental version, eg. "eng.user.20200807.162000".

The default OTA and target-files file names include all of these fields
except _incremental_.  This is the value of `ro.build.version.incremental`
in the system build properties.  It is different from `ro.build.date.utc`.
They are __not__ interchangeable.

Examples:
* lineage_mako-ota-16.0-unofficial-eng.user.20200807.162000.zip
* lineage_mako-ota-16.0-unofficial-eng.user.20200806.162000_eng.user.20200807.162000.zip

The OTA directory may contain subdirectories of arbitrary depth.

## Time stamps
Clients determine whether an update is newer than the currently running
version by comparing the local `ro.build.date.utc` property with the
`datetime` on each OTA file.  The script uses the file modification
time as the `datetime`.  Therefore, you should ensure that these values
match for each available OTA file.

One way to do this is to set the file modification time on your build
server as shown below.  You then only need to ensure that the file
modification time is preserved when you copy it to its destination on
your web server.

    t=$(grep "ro.build.date.utc" "out/target/product/$device/system/build.prop" | cut -d'=' -f2)
    touch -d "@$t" $filename

Another way to do this is to set the file modification time after copying
to your web server.

    t=$(unzip -c $filename "META-INF/com/android/metadata" | grep "^post-timestamp=" | cut -d"=" -f2)
    touch -d "@$t" $filename

## Caching
The script creates and uses a cache file named `.cache` in the OTA
directory.  This is a simple JSON file describing the results of the
last walk through the OTA directory.  If the script does not have
permission to write to this file, caching will fail and the script will
be exceptionally slow.

The cache entry for a given OTA file is updated if the file size or
modification time changes.

Cache entries for files that no longer exist are pruned.

The cache file is updated at most once per minute.  If you do not see
new files immediately, wait 60 seconds and try again.

The cache file is locked during access to ensure its integrity.

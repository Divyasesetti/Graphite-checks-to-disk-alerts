# Graphite-checks-to-disk-alerts
Graphite Installation

--->why graphite??
  In cabot, graphite checks are used to set the disk alerts.

---->How to install??
   https://www.vultr.com/docs/how-to-install-and-configure-graphite-on-ubuntu-16-04
note1:
   need to install apache2 webserver to use the graphite webserver.
---->Graphite Api
   In conf/production.env in cabot file change the configuration of graphite
---->Authentication to graphite
    In /etc/apache2/sites-enabled/apache2-graphite.conf file change the configuration to access the graphite web

<VirtualHost *:80>

     WSGIDaemonProcess _graphite processes=5 threads=5 display-name='%{GROUP}' inactivity-timeout=120 user=_graphite group=_graphite
     WSGIProcessGroup _graphite
     WSGIImportScript /usr/share/graphite-web/graphite.wsgi process-group=_graphite application-group=%{GLOBAL}
     WSGIScriptAlias / /usr/share/graphite-web/graphite.wsgi

     Alias /content/ /usr/share/graphite-web/static/
     <Location "/content/">
             SetHandler None
     </Location>

     ErrorLog ${APACHE_LOG_DIR}/graphite-web_error.log

     # Possible values include: debug, info, notice, warn, error, crit,
     # alert, emerg.
     LogLevel warn

     CustomLog ${APACHE_LOG_DIR}/graphite-web_access.log combined
     <Location "/">
             AuthType Basic
             AuthName "Restricted Content"
             AuthUserFile /etc/apache2/.htpasswd
             Require valid-user
     </Location>

</VirtualHost>

---->Disk monitoring script
#!/bin/sh
usage() {
     echo "Usage:"
     echo "$0 <disk> <metric>"
}

if [ $# -lt 2 ]; then
     usage
else
     disk="$1"
 metric="$2"
 df -H | grep -vE '^Filesystem|tmpfs|cdrom' | awk '{ print $5 " " $1 }' | grep "$disk" | while read output;
 do
     usep=$(echo $output | awk '{ print $1}' | cut -d'%' -f1  )
     echo "$metric $usep `date +%d-%m-%y/%H:%M:%S`"
     echo "$metric $usep `date +%s`" | nc localhost 2003
 done
fi
---->Feeding data to Graphite from any instance
 carbon in graphite will read all the metrics data comming from the disk in instance and calculate the metrics and stored
 in whipser database.
 The graphite will monitor the continuoussly based on running the crontab
 Example:Feeding data into graphite from kafka machine and inserting the script in kafka
    1. How to insert script in kafka?
       Go to the jumpbox from there to kafka machine
        Insert the script in owncloud
        sudo curl -o /usr/local/bin/monitor_disk.sh "https://owncloud.apxor.com/index.php/s/hza3slmnlJXUrUf/download"
        Here /usr/local/bin/monitor_disk.sh this shows the path of the disk to run continuosly
       Note2:
           Running the job continuosly by using crontab.these cronjob script have to store in /usr/local/bin
           https://superuser.com/questions/533732/where-to-store-cronjob-script
           sudo chmod +x /usr/local/bin/monitor_disk.sh [telling that this is an executable file]
     2.Crontab running
        In sudo crontab -e in that machine change the time period that for much time time that script should run,
        path of the disk and metric name
        * * * * * /usr/local/bin/monitor_disk.sh /dev/xvdg disk.kafka-1
        the above example shows that the script has to run for every one minute that having disk name, metric name
        In the above disk monitoring script from this line
        echo "$metric $usep `date +%s`" | nc localhost 2003 the data and metric name will feed in to graphite.
        here the localhost is ip of the machine where graphite is running and the port is carbons port bcze the carbon
        only listens to the data which we have fed
     3.In graphite web we can see the graphs of the corresponding metrics
       those graphs timezone can change in /etc/graphite/localsettings.py configuration file.

---->Whisper storage
   whisper having storage schema and storage aggregation this shows how much data,timeperiod ,time to store the data,historical data
    etc  in whisper database
    storage schema: /etc/carbon/storage-schemas
    storage aggregation: /etc/carbon/storage-aggregation
   In storage schema it having the retention period, pattern of the metrics and
  Retention period :
   Retention having two zones 1.time per point 2.time to store
    ex:pattern = disk
       retentions = 1m:1d(first archive),5m:6d(second archive) ==>24*60+6*24*60/5
           this means the first archive data for every minute will come and store for 1 day,second archive says for every 5 mins the one minute data
    is aggregated .This aggregation function is in storage aggregation in carbon
  aggregation :
   aggregation having xfiles factor,pattern of metrics, aggregation function
    eg:[all_min]
       pattern = \.min$
       xFilesFactor = 0.1
       aggregationMethod = min
eg says 10 % slot first archive will agrregated to second archive with minimum fucntionality
  To see how the data is aggregated and stored in whsiper database by using
      eg: /usr/bin/whisper-info /var/lib/graphite/whisper/disk/kafka1.wsp
       whisper information
      eg: /usr/bin/whisper-dump /var/lib/graphite/whisper/disk/kafka1.wsp
whisper stores the data of each metric in wsp files format /var/lib/graphite/whisper/disk/kafka1.wsp
---->Disk space per datapoint
    How much the space utilizes per datapoint by the disk? by using this we can calculate
    https://gist.github.com/ajayverghese/4955210
    12.2 bytes per datapoint

IMPORTANT NOTES:
 1.After changing the retention period of one metric i fwe change that retention again for that metric it will not effect the past
  metric files so in /usr/bin/whisper-resize.py
  change it to
  eg:find ./ -type f -name '*.wsp' -exec whisper-resize --nobackup {} 1m:7d 5m:30d 30m:90d 24h:1y \;
2.Restarting the carbon cahe after configuration change
  sudo service carbon-cache start
3.old timestamp data cannot be store in graphite
4.To see whether the carbon is storing the data in whisper or not by using vi /var/log/carbon/creates.log
5.Care about ownerships and permissions of graphite

Graphite checks in cabot:
 1.metric should be match
 2.use service to notify
 3.In carbon conf change cabot.example.com to public ip of graphite bcz whenever alerts are came the link will show the reason.

#prime_encoder

prime_encoder is network encoder for SageTV which tunes and streams video from an HDHomerunPrime.

prime_encoder is currently considered beta. There are some things that need to be cleaned up in
the code, but it has been my primary method of recording shows for the last few months with no problems.

##Background

When the Silicon Dust HDHomerunPrime was released using a Cable Card suddenly seemed like a possible
option with SageTV since some cable providers have channels marked as "copy freely". However, getting
a working Linux driver for the Prime seemed to be challenging.

I realized that the hdhomerun_config setup script allowed you to stream content from the Prime directly
(other tools like ffmpeg can do this as well) - I just needed a way to get SageTV to tune the Prime.
Luckily, SageTV support the concept of "network encoders" - other devices on the local network that act
as tuner that SageTV can direct.

So I created prime_encoder to do this middle-man task.

prime_encoder script doesn't have to run on the same box as the master SageTV server. As long as you
have it running somewhere on your network and have the recording directories set up correctly you
should be fine. I happen to run the script on the same box as the SageTV server because the overhead of
running it is fairly small and doesn't seem to impact the main SageTV performance.

The SageTV server will automatically discover network encoders. It sends commands to the network encoders
to record video and tells the encoders where to dump the video file, so the only config option that needs to
be changed on the SageTV server is to enable discovery.

An OS startup script for boot time is not provided, so make sure you start it manually if your server reboots.

Or, you can add this to the bottom of /opt/sagetv/server/sagesettings, and the encoder will start when
the SageTV process starts:

        if [[ ! $(ps aux |grep "[p]rime_encoder: Master") ]]; then /opt/sagetv/hdhomerun/prime_encoder; fi

Note that not all commands are currently supported. Primarily this means you can't yet "preview" channels
from the Setup Video Sources screen.

##Instructions

These directions assume a few things:

* The prime_encoder will run on Linux
* You have perl installed. You'll also need the following non-core modules installed:

        IPC::Run
        Proc::Daemon
   
  (these are libipc-run-perl and libproc-daemon-perl on Ubuntu)

* You have a non-root user to run prime_encoder as.  I use the user name 'sagetv' in the instructions below.
* The sagetv user can write to the recording directories that the SageTV server uses. This might require
  you to set up an SMB mount or similar if your SageTV server is Windows. Since my SageTV server is Linux
  I just run prime_encoder on the server itself.
* The server running the prime_encoder and the server running the SageTV server should be on the same
  network without any firewalls in between.
* If your Linux box is running a host firewall, you'll need to open holes for UDP port 8271 and TCP
  ports you define later for each encoder.

##Install steps

* Create a place for the script:

        mkdir -p /opt/sagetv/hdhomerun
        chown sagetv:sagetv /opt/sagetv/hdhomerun

* Put <tt>prime_encoder</tt> into <tt>/opt/sagetv/hdhomerun</tt> and make sure it's executable:
 
        chown sagetv:sagetv /opt/sagetv/hdhomerun/prime_encoder
        chmod +x /opt/sagetv/hdhomerun/prime_encoder

* Get the Linux <tt>hdhomerun_config</tt> from SD and place it in <tt>/opt/sagetv/hdhomerun/</tt>
  and make sure it's executable:

        chown sagetv:sagetv /opt/sagetv/hdhomerun/hdhomerun_config
        chmod +x /opt/sagetv/hdhomeru/hdhomerun_config

* Make sure your HDHR Prime is already configured and enabled for CC reception. Use the SD provided tools
  to test that you can view video before continuing.
* Mount the recording directory from the SageTV server somewhere on your Linux box. This isn't needed if
  prime_encoder is running on the same server as the SageTV server.
* Make sure the sagetv user can write to this directory
* Run:

        hdhomerun_config discover

* Update the encoders section of the prime_encoder script to have the id(s) of your primes. Each prime
  should have 3 tuners (0,1,2) (although you don't have to enable or even list them all if you don't want
  to). Make sure each encoder has a unique port number and name.
* Update the <tt>prime_encoder</tt> script and change the value of <tt>$ffmpeg_cmd</tt> to point to a
  version of ffmpeg that is 1.2.1 or higher. The version shipped with SageTV seems to be older than this,
  so get a more recent version. (I have seen crashes on versions earlier than 1.2.1.)
* If your SageTV server is running on another host (either Windows or Linux) you'll have to update
  the <tt>remap_file_path</tt> function to make the path from the master server map to the path that the
  prime_encoder process would see.  You may need to run the prime_encoder once and try tuning to see
  what the master server is expecting.
* Run <tt>prime_encoder</tt>:

        cd /opt/sagetv/hdhomerun
        ./prime_encoder

   (or for testing you can add the -d option which stop the process from becoming a daemon and will
   send all logs to STDOUT instead of the log file)

* Watch the log:

        tail -f prime_encoder.log

  You should see something like:

        Sat Aug 10 14:30:52 2013 Starting
        Sat Aug 10 14:30:52 2013 Discovery Listener started
        Sat Aug 10 14:30:52 2013 Tuner 'HDHomerun Prime Tuner 0' not enabled. Ignoring.
        Sat Aug 10 14:30:52 2013 Tuner 'HDHomerun Prime Tuner 1': Waiting for client connection
        Sat Aug 10 14:30:52 2013 Tuner 'HDHomerun Prime Tuner 2': Waiting for client connection

* Stop your SageTV server.

* Make a backup of your Wiz.bin and any other file you think are important (I usually back up the entire
  sage directory).

* Edit the Sage.properties file and make sure this line is there and that it's set to true:

        network_encoder_discovery=true

* Restart your SageTV server. It should auto detect the prime_encoder tuners. In the <tt>prime_encoder.log</tt>
  you should see something like:

        Fri Aug  9 22:44:40 2013 Discovery service announced 'HDHomerun Prime Tuner 2'

* In SageTV, add a new encoder. The ones you defined in the encoders config should be listed. Proceed
  as normal. Note: Your SageTV server may have auto-detected the Prime on the network as a two-tuner
  HDHomerun (non-Prime). Do NOT select the HDHomerun devices. Select the names you gave your devices
  in the prime_encoder config section, which will appear such as:

        HDHomerun Prime Tuner 0 on sage.local:7000 TV Tuner

* Once the new encoder is enabled, try tuning a channel. You should see something similar to this in the
  <tt>prime_encoder.log</tt> file:

        Fri Aug  9 22:06:52 2013 Tuner 'HDHomerun Prime Tuner 1': Tuning channel 506 on device 131760B8 tuner 1 filename: /var/media/tv/TheBigBangTheory-TheInfestationHypothesis-15413766-0.mpg

* If you need to stop prime_encoder, just <tt>ps -ef "grep prime_encoder</tt>. You should see 2 processes plus
  one for each encoder you defined. Killing the "Main" process will stop the others.

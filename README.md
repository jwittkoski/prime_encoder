# prime_encoder

prime_encoder is network encoder for SageTV

prime_encoder has been my primary method of recording shows for about five years and it has been
very reliable.

## Background

When the Silicon Dust HDHomerun Prime was released using a Cable Card with SageTV suddenly seemed like
a possibility since some cable providers have channels marked as "copy freely". However, there was no
direct suport in SageTV for the Prime and no Linux drivers for it (as it is a network attached device).

The Silicon Dust hdhomerun_config setup script allowed you to stream content from the Prime directly
(other tools like ffmpeg can do this as well) so I just needed a way to get SageTV to tune the Prime and
start steaming. Luckily, SageTV supports the concept of "network encoders" - other devices on the local
network that act as tuner/recorders that SageTV can direct.

I created prime_encoder to do this middle-man task.

Originally I was using prime_encoder for recording from an HDHomerun Prime device, but it has
been refactored to be more generic so that any command that generates video can be used.

The full set of network encoding commands is not supported. Primarily this means you can't "preview"
channels from the Setup Video Sources screen.

## Install

### Notes

prime_encoder script doesn't have to run on the same host as the master SageTV server. You only need to
have it running somewhere on your network and have the recording directories set up correctly using SMB or
another sharing protocol.

I run the script on the same host as the SageTV server because the overhead of running it is fairly small
and doesn't seem to impact SageTV performance. This has the added advantage of not having to deal with
network shares or mapping the recording paths across hosts.

When it starts, the SageTV server will automatically discover network encoders. It sends commands to
the network encoders to record video and tells the encoders where to store the video file. The only config
option that needs to be changed on the SageTV server is to enable network encoder discovery.

### Running at startup

prime_encoder needs to start before the main SageTV process starts.

The easiest way to do this is to use the basic example Ubuntu script (<tt>prime_encoder.init</tt>).

Alternatively, you can add this to the bottom of /opt/sagetv/server/sagesettings, and the encoder will start when
the SageTV process starts:

        if [[ ! $(ps aux |grep "[p]rime_encoder: Master") ]]; then /opt/sagetv/hdhomerun/prime_encoder; fi

### Assumptions

These directions assume a few things:

* The prime_encoder will run on Linux
* You have perl installed. You'll also need the following non-core modules installed:

        IPC::Run
        Proc::Daemon
        YAML::Tiny
   
  (these are libipc-run-perl, libproc-daemon-perl, and libyaml-tiny-perl on Ubuntu)

* You have a non-root user to run prime_encoder as.  I use the user name 'sagetv' in the instructions below.
* The sagetv user can write to the recording directories that the SageTV server uses. This might require
  you to set up an SMB mount or similar if your SageTV server is Windows. Since my SageTV server is Linux
  I just run prime_encoder on the server itself.
* The server running the prime_encoder and the server running the SageTV server should be on the same
  network without any firewalls in between.
* If your Linux box is running a host firewall, you'll need to open holes for UDP port 8271 and TCP
  ports you define later for each encoder.

## Installation

* Create a place for everything:

        mkdir -p /opt/sagetv/hdhomerun
        chown sagetv:sagetv /opt/sagetv/hdhomerun

* Put <tt>prime_encoder</tt> into <tt>/opt/sagetv/hdhomerun</tt> and make sure it's executable:
 
        chown sagetv:sagetv /opt/sagetv/hdhomerun/prime_encoder
        chmod +x /opt/sagetv/hdhomerun/prime_encoder

* Copy the <tt>config.example.yml</tt> to <tt>config.yml</tt>

* Get the Linux <tt>hdhomerun_config</tt> from Silicon Dust and place it in <tt>/opt/sagetv/hdhomerun/</tt>
  and make sure it's executable:

        chown sagetv:sagetv /opt/sagetv/hdhomerun/hdhomerun_config
        chmod +x /opt/sagetv/hdhomerun/hdhomerun_config

* Make sure your HDHR Prime is already configured and enabled for CC reception. Use the Silicon Dust
   provided tools to test that you can view video before continuing.
* Mount the recording directory from the SageTV server somewhere on your Linux box. This isn't needed if
  prime_encoder is running on the same server as the SageTV server.
* Make sure the sagetv user can write to this directory
* Discover the ID for the Prime device:

        hdhomerun_config discover

* Update the encoders section of <tt>config.yml</tt> to have the id(s) of your primes. Each prime
  should have 3 tuners (0,1,2) (although you don't have to enable or even list them all if you don't want
  to). Make sure each encoder has a unique port number and name.

* Edit the <tt>config.yml</tt> to match how to tune your devices. See the examples in the file.

* (Optional) Edit the <tt>config.yml</tt> so that the <tt>cmd_postprocess</tt> points to a version of ffmpeg
  that is 1.2.1 or higher. The version shipped with SageTV seems to be older than this,
  but a more recent version as I have seen crashes on versions earlier than 1.2.1.
  You do not need to specify a value for <tt>cmd_postprocess</tt> but I have found that it helps
  fix timing issues in the video stream that can occur when providers switch between shows and commericals.

* (Optional) If your SageTV server is running on another host (either Windows or Linux) you'll have to update
  the <tt>remap_file_path</tt> function in <tt>prime_encoder</tt> to make the path from the master server map to the path that the
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

* *Make a backup of your Wiz.bin and any other files you think are important (I usually back up the entire
  sage directory).*

* Edit the Sage.properties file and make sure this line is there and that it's set to true:

        network_encoder_discovery=true

* Restart your SageTV server. It should auto detect the prime_encoder tuners. In the <tt>prime_encoder.log</tt>
  you should see something like:

        Fri Aug  9 22:44:40 2013 Discovery service announced 'HDHomerun Prime Tuner 2'

* In SageTV, add a new encoder. The ones you defined in <tt>config.yml</tt> should be listed. Proceed
  as normal. Note: Your SageTV server may have auto-detected the Prime on the network as a two or three tuner
  HDHomerun (non-Prime). Do NOT select the HDHomerun devices. Select the names you gave your devices
  in the prime_encoder config section, which will appear such as:

        HDHomerun Prime Tuner 0 on sage.local:7000 TV Tuner

* Once the new encoder is enabled, try tuning a channel. You should see something similar to this in the
  <tt>prime_encoder.log</tt> file:

        Fri Aug  9 22:06:52 2013 Tuner 'HDHomerun Prime Tuner 1': Tuning channel 506 on device 131760B8 tuner 1 filename: /var/media/tv/TheBigBangTheory-TheInfestationHypothesis-15413766-0.mpg

* If you need to stop prime_encoder and you installed the <tt>prime_encoder.init</tt>, you can
   use <tt>service prime_encoder stop</tt>
* If you need to stop prime_encoder and you didn't use <tt>prime_encoder.init</tt>, you can do
  something like: <tt>ps -ef | grep prime_encoder</tt>. You should see 2 processes plus
  one for each encoder you defined. Killing the "Main" process will stop the others.


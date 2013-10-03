prime_encoder

prime_encoder is network encoder for a SageTV server which tunes and streams data from a HDHomerunPrime.

Currently prime_encoder is considered beta. There are some things that need to be cleaned up in the code, but it has been my primary method of recording shows for the last few months with no problems.

Background

When the Silicon Dust HDHomerunPrime was released using a Cable Card suddenly seemed like a possible option with SageTV since some cable providers have channels marked as "copy freely". However, getting a working Linux driver for the Prime seemed to be challenging.

I realized that the hdhomerun_config setup script allowed you to stream content from the Prime directly (other tools like ffmpeg can do this as well) - I just needed a way to get SageTV to tune the Prime. Luckily, SageTV support the concept of "network encoders" - other devices on the local network that act as tuner that SageTV can direct.

So I created prime_encoder to do this middle-man task.

prime_encoder script doesn't have to run on the same box as the master SageTV server. As long as you have it running somewhere on your network and have the recording directories set up correctly you should be fine. I happen to run the script on the same box as the SageTV server, but it's not required.

The SageTV server will automatically discover network encoders. It sends commands to the network encoders to record things and tells the encoders where to dump the file, so the only config option that needs to be changed on the SageTV server is to enable discovery.

An OS startup script for boot time is not provided, so make sure you start it manually if your server reboots.

Or, you can add this to the bottom of /opt/sagetv/server/sagesettings, and the encoder will start when the SageTV process starts:

if [[ ! $(ps aux |grep "[p]rime_encoder: Master") ]]; then /opt/sagetv/hdhomerun/prime_encoder; fi

Note that not all commands are currently supported. Primarily this means you can't "preview" channels from the Setup Video Sources screen.

Instructions:

These directions assume a few things:

1. The prime_encoder will run on Linux
2. You have perl installed. You'll also need the following non-core modules installed:
       IPC::Run
       Proc::Daemon
   (these are libipc-run-perl and libproc-daemon-perl on Ubuntu)
3. You have a non-root user to run prime_encoder as.  I use the user name 'sagetv' in the instructions below.
4. The sagetv user can write to the recording directories that the SageTV server uses. This might require you to set up an SMB mount or similar if your SageTV server is Windows. Since my SageTV server is Linux I just run prime_encoder on the server itself.
5. The server running the prime_encoder and the server running the SageTV server should be on the same network without any firewalls in between.
6. If your Linux box is running a host firewall, you'll need to open holes for UDP port 8271 and TCP ports you define late for each encoder.

Install steps:

1. mkdir -p /opt/sagetv/hdhomerun
   chown sagetv:sagetv /opt/sagetv/hdhomerun
2. put prime_encoder into /opt/sagetv/hdhomerun
   Make sure it's executable:
   chown sagetv:sagetv /opt/sagetv/hdhomerun/prime_encoder
   chown +x /opt/sagetv/hdhomerun/prime_encoder
3. Get the Linux hdhomerun_config from SD and place it
   in /opt/sagetv/hdhomerun/
   Make sure it's executable:
   chown sagetv:sagetv /opt/sagetv/hdhomerun/hdhomerun_config
   chown +x /opt/sagetv/hdhomeru/hdhomerun_config
4. Make sure your HDHR Prime is already configured and enabled for CC reception. Use the SD provided tools to test that you can view video before continuing.
5. Mount the recording directory from the server somewhere on your Linux box. This isn't needed if prime_encoder is running on the same server as the SageTV server.
6. Make sure the sagetv user can write to this directory
7. Run: hdhomerun_config discover
8. Update the encoders section of the prime_encoder script to have the id(s) of your primes. Each prime should have 3 tuners (0,1,2) (although you don't have to enable or even list them all if you don't want to). Make sure each encoder has a unique port number and name.
9. Update the prime_encoder script and change the value of $ffmpeg_cmd to point to a version of ffmpeg that is 1.2.1 or higher. The version shipped with SageTV seems to be older than this, so get a more recent version. (I have seen crashes on versions earlier than 1.2.1.)
10. If your SageTV server is running on another host (either Windows or Linux) you'll have to update the remap_file_path function to make the path from the master server map to the path that the prime_encoder process would see.  You may need to run the prime_encoder once and try tuning to see what the master server is expecting.
11. Run the prime_encoder:
   cd /opt/sagetv/hdhomerun
   ./prime_encoder

   (or for testing you can add the -d option which stop the process from becoming a daemon and will send all logs to STDOUT instead of the log file)

12. Watch the log:
   tail -f prime_encoder.log

   You should see something like:

    Sat Aug 10 14:30:52 2013 Starting
    Sat Aug 10 14:30:52 2013 Discovery Listener started
    Sat Aug 10 14:30:52 2013 Tuner 'HDHomerun Prime Tuner 0' not enabled. Ignoring.
    Sat Aug 10 14:30:52 2013 Tuner 'HDHomerun Prime Tuner 1': Waiting for client connection
    Sat Aug 10 14:30:52 2013 Tuner 'HDHomerun Prime Tuner 2': Waiting for client connection

13. Stop your SageTV server.

14. Make a backup of your Wiz.bin and any other file you think are important (I usually back up the entire sage directory).

15. Edit the Sage.properties file and make sure this line is there and that it's set to true:

network_encoder_discovery=true

16. Restart your SageTV server. It should auto detect the prime_encoder tuners. In the prime_encoder.log you should see something like:

    Fri Aug  9 22:44:40 2013 Discovery service announced 'HDHomerun Prime Tuner 2'

17. In SageTV, add a new encoder. The ones you defined in the encoders config should be listed. Proceed as normal. Note: Your SageTV server may have auto-detected the Prime on the network as a two-tuner HDHomerun (non-Prime). Do NOT select the HDHomerun devices. Select the names you gave your devices in the prime_encoder config section, which will appear such as:

HDHomerun Prime Tuner 0 on sage.local:7000 TV Tuner

18. If you need to stop prime_encoder, just "ps -ef "grep prime_encoder". You should see 2 processes plus one for each encoder you defined. Killing the "Main" process will stop the others.

# Mutiny Fuzzing Framework - Experimental Branch! `<(^_^)>`

The Mutiny Fuzzing Framework is a network fuzzer that operates by replaying
PCAPs through a mutational fuzzer.  The goal is to begin network fuzzing as
quickly as possible, at the expense of being thorough (for now).

The general workflow for Mutiny is to take a sample of legitimate traffic, such
as a browser request, and feed it into a prep script to generate a .fuzzer file.
Then, Mutiny can be run with this .fuzzer file to generate traffic against a
target host, mutating whichever packets the user would like.

Mutiny uses [Radamsa](https://github.com/aoh/radamsa) to perform mutations.

Written by James Spadaro (jaspadar@cisco.com) and Lilith Wyatt (liwyatt@cisco.com)

# Quickstart: Mutiny tutorial

Blog post here:
* http://blog.talosintelligence.com/2018/01/tutorial-mutiny-fuzzing-framework-and.html

Links to this YouTube video demo:
* https://www.youtube.com/watch?v=FZyR6MgJCUs

# Typical Workflow:
1. Find a target to fuzz and a client for the target.
2. Setup the target: 
    * `gdb -ex "source ~/mutiny-fuzzer/harnesses/gdb_fuzz_harness.py" --args <args...>` 
3. Use [Decept Proxy](https://github.com/Cisco-Talos/Decept) for fuzzer dump. 
    * `python ~/decept/decept.py 127.0.0.1 9999 <dst_ip> <dst_port> --timeout .1 --dont_kill --fuzzer dumped.fuzzer` 
4. Run the client through decept to get a fuzzer. (Alternatively, use a pcap with mutiny_prep.py) 
5. Fuzz the target:
    * `python ~/mutiny-fuzzer/mutiny.py corpus/test.fuzzer -i 192.168.0.1 --feedback 192.168.0.1:60000 --timeout .4`
6. Crashes will be written into `mutidumps.txt` on the fuzzer side, and `crashes/<crash_desc_and_time>` on the client.
    * On the fuzzer side, proof of concept python scripts will also be created in `<cwd>/autogen_pocs` folder. 

# Todo:
* Good feedback. There's currently the `fuzzer_feedback` folder which is a dynamorio version of feedback, but I'd consider those a rough draft. Currently working on something more portable and usable.  

# Experimental Branch Overview `<(^.^)>`

There's been a large amount of changes compared to the master branch, mostly things added
to make Mutiny better at longterm fuzzing campaigns. A list of the things I can think of
is found here:

1. Harnesses
    * Some basic harnesses have been included to interact with `mutiny.py --feedback`(preferred) and/or campaign_mode.py  
    * e.g. : `python ~/mutiny-fuzzer/mutiny.py corpus/test.fuzzer -i 192.168.0.1 --feedback 192.168.0.1:60000 --timeout .4`
    * And then on the target: `gdb -ex "source ~/mutiny-fuzzer/harnesses/gdb_fuzz_harness.py" --args <args...>` 
2. mutiny\_prep.py campaign mode:
    * The -c \<folder\> will try to dump all sessions out of a .pcap into different .fuzzers inside \<folder\>. 
3. mutiny.py
    * `-F <ipaddrss>` => Fuzzer harness directly for mutiny. Works with `harnesses/gdb_fuzz_harness.py`
    * `-e <seedNum>` has been added to emulate a given seed.
    * `-x <filename>`  can dump out a POC for a given seed.
    * `-m <numberRange>` allows you to fuzz only a specific message set 
    * `-R <numPerMessage>`  causes mutiny to fuzz each submessage for \<numPerMessage\> iterations. 
    * `-M <mutator>` Use a specific mutator besides default radamsa (it must use stdin/stdout). 
4. More accurate crash detection/fuzzer control
    * Monitor class `mutiny_classes/monitor.py` by default has added granularity. By default, fuzzing will stop when the target port cannot be connected to, but different locking conditions can be set (e.g. pinging, process listenting, never locked)
5. Auto generated POCs (`autogen_pocs/<fuzzername_seed_msg>.py`)
6. Fuzzing callbacks (presend/prefuzz/etc) added into the individual .fuzzers themselves for a per-fuzzer customization.
    * If you still want the `mutiny_classes/message_processor.py` customization, use the `-f, --forceMsgProc` switch. 
7. campaign\_mode.py
    * Run for long periods of time, and dumps crashes accordingly.
    * Can take a folder as it's .fuzzer argument, and it will rotate over the corpus. 

# Assorted Thoughts/Tips:
* Occasionally more than one seed might be needed to cause the crash so the `-r` and `-l` switches are useful for replication.
* If there's no output from mutiny but there is network traffic, it might just be a timeout issue (`--timeout` and `--sleeptime`). 
    
## Setup

Ensure python and scapy (for mutiny_prep.py) are installed. Alternatively,
 [Decept Proxy](https://github.com/Cisco-Talos/Decept) can be used to dump
.fuzzer files from traffic that has been piped through it.

Untar Radamsa and `make`  (You do not have to make install, unless you want it
in /usr/bin - it will use the local Radamsa) Update `mutiny.py` with path to
Radamsa if you changed it.

## .fuzzer prep  

Save pcap into a folder.  Run `mutiny_prep.py` on `<XYZ>.pcap` (also optionally
pass the directory of a custom processor if any, more below).  Answer the
questions, end up with a `<XYZ>.fuzzer` file in same folder as pcap.

Run `mutiny.py <XYZ>.fuzzer <targetIP>` This will start fuzzing. Logs will be
saved in same folder, under directory
`<XYZ>_logs/<time_of_session>/<seed_number>`

Alternatively, `python decept.py 127.0.0.1 9999 <dstIP> <dstPort> --fuzzer <fuzzer name>`
and then send your traffic through (127.0.0.1,9999).


## More Detailed Usage

### .fuzzer Files

The .fuzzer files are human-readable and commented.  They allow changing various
options on a per-fuzzer-file basis, including which message or message parts are
fuzzed.

### Message Formatting

Within a .fuzzer file is the message contents.  These are simply lines that
begin with either 'inbound' or 'outbound', signifying which direction the
message goes.  They are in Python string format, with '\xYY' being used for
non-printable characters.  These are autogenerated by 'mutiny_prep.py' and
Decept, but sometimes need to be manually modified.

### Message Formatting - Manual Editing

If a message has the 'fuzz' keyword after 'outbound', this indicates it is to be
fuzzed through Radamsa.  A given message can have line continuations, by simply
putting more message data in quotes on a new line.  In this case, this second
line will be merged with the first.

Alternatively, the 'sub' keyword can be used to indicate a subcomponent.  This
allows specifying a separate component of the message, in order to fuzz only
certain parts and for convenience within a Message Processor.

Here is an example arbitrary set of message data:
```
outbound 'say hi'
more fuzz ' and fuzz this'
more ' but not this\xde\xad\xbe\xef'
inbound 'this is the server's'
    ' expected response'
```

### Customization

mutiny_classes/ contains base classes for the Message Processor, Monitor, and
Exception Processor.  Any of these files can be copied into the same folder as
the .fuzzer (by default) or into a separate subfolder specified as the
'processor_dir' within the .fuzzer file.

### Customization - Message Processor

Contains different hooks for different stages of the fuzzing process. 
Just write python for grooming the messages however you want. 

#### Example:
* For keeping an accurate size field, just put "SIZE" inside the outbound message and then, in the .fuzzer itself:
```
    def preSendProcess(self, message):

        if "SIZE" in message:
            try:
                ret = "\x80\x00"
                ret+=struct.pack(">I",len(message)-6)
                ret+=message[6:]
                return ret
            except Exception as e:
                print e
                pass

        return message
```



### Customization - Monitor

Contains different ways to lock the execution of the fuzzer. By default, will
lock if it cannot connect to the port that is being fuzzed, but there are other
ways to lock in case this is not sufficient.




# GR-MultipleReceivers
GNURadio project to split a single RTL into multiple subbands of audio to feed to multiple decoding software.

# Installation
Right now, these are my notes while building the system.

## OS
I did my initial development on my laptop running Ubuntu 16.04 and GNURadio 3.7.

I'm moving to PiSDR, 2020-11-13 "vanilla" image.
* Install PiSDR following their instructions.
* I enabled SSH, by creating a zero byte file called `ssh` on the `/boot` partition.  (This is how you do it in Raspbian, so I assumed it would work here.)
* I hate loging in as `pi` so I changed my user to `mark` and set a strong password.
* I changed the hostname to `flyspy` (a Shadowrun reference.)
* My goal is to do this completely headless (I'm out of monitors!), entirely over SSH.
  * I'm starting using X11 forwarding in SSH. Windows users may not have this option.  (I'm noticing this is kinda laggy. But it works.)
  * Another option is to use VNC.  If I run into problems with X11, I'll try VNC.
* GNU Radio Companion needs `xterm`, so install it: `sudo apt install xterm`.  Alternatively, you can configure it to use `gnome-terminal`.

Testing the image:
* `ssh -Y mark@flyspy` will enable X11 forwarding.  When I run a GUI command on `flyspy`, that GUI will display on my laptop.
* Test with `gnuradio-companion` on flyspy.  The GRC window should show up on your desktop.

Optimizing GNU Radio for the hardware platform:
* Run `volk_profile` and let it complete.  This will help gnuradio run more efficiently later.  It only needs to be done once.

## SDR hardware:
* I use the rtl-sdr.com V3 dongles. They are reasonably priced, solidly built, and will directly receive HF without the use of an upconverter.
* But I can't think of any reason why any other hardware that can receive the frequencies you're interested in couldn't also work.

### Making the system work with multiple dongles
* USB is not good about statically identifying devices.  So we want to tag the receivers in a way to easily identify them in the OS.
  * This process is unique to the RTL-SDR driver. If you're using another driver, you'll have to figure out how to do this yourself.
* Physically label your receivers for the bands they'll work on.  For these instructions, I'll talk about a 20m and 40m receiver.
* For this process, only connect a single receiver to the RPi at a time.
  * Connect the 20m dongle to the RPi, then run:
    ```
rtl_eeprom
    ```
    You should see an output similar to this:
    ```
[11:16:48] mark@flyspy:~/src/GR-MultipleReceivers$ rtl_eeprom 
Found 1 device(s):
  0:  Generic RTL2832U OEM

Using device 0: Generic RTL2832U OEM
Detached kernel driver
Found Rafael Micro R820T tuner

Current configuration:
__________________________________________
Vendor ID:		0x0bda
Product ID:		0x2838
Manufacturer:		Realtek
Product:		RTL2838UHIDIR
Serial number:		00000000
Serial number enabled:	yes
IR endpoint enabled:	yes
Remote wakeup enabled:	no
__________________________________________
Reattached kernel driver
[11:25:59] mark@flyspy:~/src/GR-MultipleReceivers$ 
    ```
  * The things to look for here are:
    * `Found 1 device(s)` If it fines more than one, remove all but one at a time.  This just removes any ambiguity of which device you're changing.
    * `Serial number:`  A new one will be zero, or one, I can't remember.  But they'll all be the same.
  * Change the serial number to some marker of the band it will operate on. I use Frequency in MHz:
    ```
    rtl_eeprom -s 00000014
    ```
    * You could use the band in meters, but I might some day want to do something similar on 70cm, etc. But you might also want to do 472kHz or 135kHz, so it's really up to you.  The exact value here is arbitrary, but you'll use it later when configuring GNU Radio for the RTLs.
  * Type `y` to write the new config to the device.
  * Remove the 20m receiver, insert the 40m receiver, and do the same thing with a different serial number.  Do this for all your receivers if you've got more than 2.


## RF hardware:
The goal here is to get RF from antenna(s) into the RTL receivers.  Any way you can make that happen will work.  Here are some ideas:

### Single Antenna
* You'll need a receive antenna on all the bands you intend to receive.  For development, I'm using my Comet H422, which works on 40m, 20m, 15m, and 10m.  Eventually, I intend to build an [Active Antenna by LZ1AQ](https://active-antenna.eu/).
* Then you'll need to split that antenna into multiple bands.
  * You MIGHT be able to use a totally passive splitter and connect all receivers in parallel, but that puts multiple 50ohm receiver loads in parallel, splitting the receive signal available to each receiver.
  * I use [HF triplexers by SV1AFN](https://www.sv1afn.com/en/products/hf-antenna-multiplexer.html).  These use T type bandpass filters, meaning they are a high impedance outside the bandpass.  So the energy at 20m sees a 50ohm impedance at the 20m bandpass filter, and a high impedance at all the other bandpass filters; most of the energy flows to the 50ohm load.  Same with all the other bands.  In this way, each band gets "full energy" from the receive antenna.
    * As a bonus, the band-pass filters act as very good pre-selection for the RTL-SDR dongles, which have _NO_ preselection and are easily wiped out by near-by strong transmitters.  Such as the 15kW AM broadcast station that's 5km east of my QTH. Without pre-filtering, RTL dongles are completely useless.  These triplexers solve that problem for me.
    * You can connect two (probably more?) triplexers in series to get 6 (more?) bands.  My setup gives me 80m, 40m, 30m, 20m, 15m and 10m.

### Multiple Antennas
* Alternatively, you can use multiple antennas, connecting one to each receiver directly.
* This obviously uses a lot of coax, and might take up more space for the antennas (depending on your situation, obviously.)
* But, if you've already got the antennas setup, there's no need to build a new system.
* Note above about the pre-selection benefit of running the triplexer.  If you connect an RTL-SDR receiver directly to an antenna, it will get blown out by any near by transmitter, no matter what frequency those transmitters are on.  I'd encourage you to put some sort of preselection on the receivers: a band-pass, an AM or FM broadcast blocker, etc.


## Virtual Audio Devices
In Linux (well, Ubuntu and Raspian anyway), there are two systems that work together to make sound work:
* ALSA is the name of the low-level kernel driver level sound stuff.  It deals with physical devices, setting levels, etc.
* PulseAudio is an "audio server." It sits on top of ALSA and does things like allowing multiple applications to output audio to the same physical device by mixing their streams, etc.

Unfortunately, PulseAudio only likes to run when a user is logged in to the console, its very difficult to make it work correctly as a system service.  So we will be using ALSA exclusively for this project.

### Installing Loopback Kernel Module
```
modprobe snd_aloop
```

### Creating Virtual Interfaces
Create `/etc/alsa/conf.d/80-gnuradio-wsjtx-glue.conf` and put the following content in it:
<details><summary>Click here to show the file.  It's long so it's "hidden" by default.</summary>
<p>
```
# GNU Radio side of loopbacks
pcm.loop2_0_0 {
    type plug
        slave {
            pcm "hw:2,0,0"
        }
}

pcm.loop2_0_1 {
    type plug
        slave {
            pcm "hw:2,0,1"
        }
}

pcm.loop2_0_2 {
    type plug
        slave {
            pcm "hw:2,0,2"
        }
}

pcm.loop2_0_3 {
    type plug
        slave {
            pcm "hw:2,0,3"
        }
}

pcm.loop2_0_4 {
    type plug
        slave {
            pcm "hw:2,0,4"
        }
}

pcm.loop2_0_5 {
    type plug
        slave {
            pcm "hw:2,0,5"
        }
}

pcm.loop2_0_6 {
    type plug
        slave {
            pcm "hw:2,0,6"
        }
}

pcm.loop2_0_7 {
    type plug
        slave {
            pcm "hw:2,0,7"
        }
}

# WSJT-X side of loopbacks
pcm.loop2_1_0 {
    type plug
        slave {
            pcm "hw:2,1,0"
        }
}

pcm.loop2_1_1 {
    type plug
        slave {
            pcm "hw:2,1,1"
        }
}

pcm.loop2_1_2 {
    type plug
        slave {
            pcm "hw:2,1,2"
        }
}

pcm.loop2_1_3 {
    type plug
        slave {
            pcm "hw:2,1,3"
        }
}

pcm.loop2_1_4 {
    type plug
        slave {
            pcm "hw:2,1,4"
        }
}

pcm.loop2_1_5 {
    type plug
        slave {
            pcm "hw:2,1,5"
        }
}

pcm.loop2_1_6 {
    type plug
        slave {
            pcm "hw:2,1,6"
        }
}

pcm.loop2_1_7 {
    type plug
        slave {
            pcm "hw:2,1,7"
        }
}


# Gluing it all together
pcm.gr_20m_wspr {
    type asym
    playback.pcm "loop2_0_0"
    capture.pcm "loop2_0_0"
    hint {
        show on
        description "GNU Radio 20m WSPR"
    }
}

pcm.gr_20m_ft8 {
    type asym
    playback.pcm "loop2_0_1"
    capture.pcm "loop2_0_1"
    hint {
        show on
        description "GNU Radio 20m FT8"
    }
}

pcm.gr_40m_wspr {
    type asym
    playback.pcm "loop2_0_2"
    capture.pcm "loop2_0_2"
    hint {
        show on
        description "GNU Radio 40m WSPR"
    }
}

pcm.gr_40m_ft8 {
    type asym
    playback.pcm "loop2_0_3"
    capture.pcm "loop2_0_3"
    hint {
        show on
        description "GNU Radio 40m FT8"
    }
}

pcm.wsjtx_40m_ft8 {
    type asym
    playback.pcm "loop2_1_3"
    capture.pcm "loop2_1_3"
    hint {
        show on
        description "WSJTX 40m FT8"
    }
}

pcm.wsjtx_20m_wspr {
    type asym
    playback.pcm "loop2_1_0"
    capture.pcm "loop2_1_0"
    hint {
        show on
        description "WSJTX 20m WSPR"
    }
}

pcm.wsjtx_20m_ft8 {
    type asym
    playback.pcm "loop2_1_1"
    capture.pcm "loop2_1_1"
    hint {
        show on
        description "WSJTX 20m FT8"
    }
}

pcm.wsjtx_40m_wspr {
    type asym
    playback.pcm "loop2_1_2"
    capture.pcm "loop2_1_2"
    hint {
        show on
        description "WSJTX 40m WSPR"
    }
}
```
</p>
</details>

This block creates the ALSA objects needed to send sound to an interface, and have that sound appear on another interface.  The interface names are arbitrary, but I've structured them to be easy to use.  In this case, the `gr_bla` interfaces are for GNU Radio to send sound output to, and the `wsjtx_bla` interfaces are for (you guessed it) WSJT-X to receive the audio.  The band and mode are also encoded into the interface names to make it easier to pair up matching interfaces between the two programs.

You'll also note several `loop*` interfaces. Those are used internally by the config here, you shouldn't be using them. (They don't perform any resampling; if you wanted to use them, you'd have to make sure both sides are sending exactly the same bitrate, word size, signing, etc.  Setting it up this way, ALSA will do any required conversions for us.)

## GNU Radio
**NOTE** Make sure you've run `volk_profile` and let it complete.  This will help gnuradio run more efficiently later.  It only needs to be done once.

PiSDR comes with GNU Radio pre-installed.  Run it:
```
gnuradio-companion
```

### Loading and compiling the GNU Radio blocks
**TODO** All this needs a lot more work.

There are three GNU Radio modules here:
* `ssb_demodulator.grc`: Takes a Complex stream at base-band (eg: as generated internall in the `dual_demodulator` block below), and performs the Weaver Method to select upper or lower sideband and demodulate the signal into a Float stream of audio.
* `dual_demodulator.grc`: Takes a Complex stream of RF (eg: from the RTL-SDR receiver block), extracts two sub-bands from the RF stream, passes those to `ssb_demodulator` above, and returns the resultant audio streams.  The parameters to this block will look familiar to Ham operators: "Dial Frequencies" for each the FT8 and WSPR sections of the band.
* `dual-ssb-receiver.grc`: Includes the RTL-SDR receiver blocks, feeds the RF output from RTL to `dual_demodulator` blocks, then feeds the audio output to the audio interfaces.  You can also include Waterfalls and other visualizations here, but they all take CPU so I have them off by default.

Load them in that order, then Generate their flow graph, reload the Blocks list, and open the next one.

### Configure GNU Radio inputs and outputs
On the final code block you loaded, `dual-ssb-receiver.grc`, you'll need to configure for your chosen bands.  It comes configured for 20m and 40m by default (because that's what I'm doing.)

Note that the blocks are in horizontal layers:
* Variables on the left that configure everything:
  * `rtl_freq_[band]`: Sets the center frequency for the RTL dongle.  Not critical, so long as the two "dial" frequencies are within the sample rate of this center frequency.  By default the sample rate is 240kHz.  So for example, if you set the `rtl_freq` to 7.100MHz, it will receive everything from 6.980MHz (7.100 - rtl_samp_rate/2) to 7.220MHz (7.100 + rtl_samp_rate/2).  Just make sure the two frequencies below are within this range.
  * `wspr_freq_[band]`: The USB (upper side band, not universal serial bus) dial frequency to receive the WSPR sub-band.  A 400Hz band around 1.5kHz is passed.
  * `ft8_freq_[band]`: The USB dial frequency to receive the FT8 sub-band.  300Hz to 3kHz is passed.
* `RTL-SDR Source` inputs:
  * You'll at least need to pass "Device Arguments".  `rtl=[serial number],direct_samp=2`
    * `rtl=[serial number]` tells it which RTL-SDR dongle to use.  Earlier, you used `rtl_eeprom` to set the serial number on each dongle to something representing the band.  That's the value you put here.
      * DISCLAIMER: Serial numbers SHOULD work, but I never got them to work. I have to pass the "device" number here. Unfortunately, those can change if receivers are disconnected and reconnected in different orders.  If you know how to make serial numbers work, please reach out to me on Twitter: @smittyhalibut
    * `direct_samp=2` tells the rtl-sdr.com branded v3 dongle to receive HF, not VHF/UHF.  If you're using a different dongle, you MIGHT need to change this to something else.
  * If you know what you're doing, you might be able to further optimize things here.  I don't, so I just leave them all default. :-)
* `Audio Sink` outputs:
  * Set the Device Names to `gr_[band]_[mode]` interfaces as appropriate.

### Debugging GNU Radio
By default, `dual-ssb-receiver.grc` is setup to run completely headless, no GUI at all.  This give no feedback on how well its working, other than error messages if it completely fails.

To turn on Waterfalls for testing (or if your computer has the CPU capacity for it):
* In the `Options` block, change "Generate Options" from "No GUI" to "QT GUI".  This turns off the "headless mode."
* Right click on the `QT GUI Waterfall Sink` blocks and "Enable" them.

Then (re)start the GNU Radio execution.  You can do this without stopping WSJT-X (if they're running yet), they'll just get periods of silence while GNU Radio isn't running.

You should see a Window open with Waterfalls showing the receive signals on the respective subbands.


## Decoding software, eg: WSJT-X
PiSDR, our OS, doesn't include this software, so we have to install it ourselves.

Go to the [WSJT-X home page](https://physics.princeton.edu/pulsar/k1jt/wsjtx.html) and download the latest GA release (or the release candidate if you're feeling saucy) and install it:
```
wget https://physics.princeton.edu/pulsar/k1jt/wsjtx_2.2.2_armhf.deb
sudo apt install ./wsjtx_2.2.2_armhf.deb
```
Using `apt` to install the package instead of `dpkg` will download the dependencies.  Note that `apt` has a version of wsjtx as well, but as of this writing, it's v2.0.0 where v2.2.2 is available.  So downloading the `.deb` and using `apt` to install it gets you the newest version of WSJT-X, and automatically manages its dependencies. The best of all worlds.  It does mean you're responsible for downloading new versions from the `princeton.edu` site when they're available.

### Running multiple copies of WSJT-X
Running WSJT-X from the command line, you can pass `-r [radio name]` to it and it will create a unique context for this instance of the program, allowing you to run multiple instances of WSJT-X at the same time.  Again, the names are arbitrary, but I suggest using the band and mode in the "radio" name to make it easy to differentiate them, like so:
```
wsjtx -r 20m-FT8
```

### Configuring WSJT-X
When you start a WSJT-X instance like that, it starts with a completely fresh, default configuration.  Configure it as you would normally:
* Main Screen:
  * Set the band and mode
* Settings:
  * General Tab: Enter your callsign and grid
  * Audio Tab: For Input, select the `wsjtx_[band]_[mode]` interface as appropriate.  Keep it on Mono.  Don't worry about Output, leave it blank.
  * Reporting Tab: Check "Enable PSK Reporter Spotting".  (That's why we're here, right?)
  * All other settings are ok defaulted, but you're welcome to change anything else you want.

At this point, I usually exit WSJT-X and restart it, to make sure it saved the configuration.

### Optimizing WSJT-X for performance
On my Raspberry Pi4, I can run 2 RTL-SDR dongles, GNU Radio, and 4 instances of WSJT-X, but only just barely.  I have to do the following to make it not run out of CPU at decode times:
* Turn off the Waterfall displays in WSJT-X. I love watching them, and we can still use them for debugging, but they take a fair bit of CPU to drive.
* Set `Decode` to `Normal`, at least on WSPR.  WSPR runs on 2 minutes worth of data in a spike, which consumes 8x the CPU of the FT8 decoder (which only runs on 15 seconds of data.)  The spikes at the 2-minutes are much bigger than the spikes at the 15-seconds.
* Even so, over time, the buffer between GNU Radio and WSJT-X will start lagging behind.  Restarting GNU Radio will reset this lag, but it will just start lagging behind over time again.

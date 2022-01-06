# CAKE with Adaptive Bandwidth - "autorate"

## About autorate
**autorate** is a program that automatically adapts the
CAKE Smart Queue Management (SQM) bandwidth settings
by measuring traffic load and RTT times.
This is designed for variable bandwidth connections such as LTE,
and is not intended for use on connections that have a
stable, fixed bandwidth.

CAKE is an algorithm that manages the buffering of data
being sent/received by an OpenWrt router so that no more
data is queued than is necessary,
minimizing the latency ("bufferbloat")
and improving the responsiveness of a network.

### Requirements

This _sqm-autorate_ program is written primarily for OpenWrt 21.02.
The current developers are not against extending it for OpenWrt 19.07,
however it is not the priority as none run 19.07.
If it runs, that's great.
If it doesn't run and someone works out why, and how to fix it,
that's great as well.
If they supply patches for the good of the project, that's even better!

_For Testers, Jan 2022:_ For those people running OpenWrt snapshot builds,
a patch is required for Lua Lanes.
Details can be found here:
[https://github.com/Fail-Safe/sqm-autorate/issues/32#issuecomment-1002584519](https://github.com/Fail-Safe/sqm-autorate/issues/32#issuecomment-1002584519)

### Installation

1. Install the **SQM QoS** package (from the LuCI web GUI) or `opkg install sqm-scripts` from the command line
2. Configure [SQM for your WAN link,](https://openwrt.org/docs/guide-user/network/traffic-shaping/sqm) setting its interface, download and upload speeds, and
checking the **Enable** box.
In the **Queue Discipline** tab, select _cake_ and _piece\_of\_cake.qos._
If you have some kind of DSL connection, read the
**Link Layer Adaptation** section of the
[SQM HOWTO.](https://openwrt.org/docs/guide-user/network/traffic-shaping/sqm)
3. Run the following command to run the setup script that downloads and installed the required files and packages:

   ```bash
   sh -c "$(wget -q -O- https://raw.githubusercontent.com/Fail-Safe/sqm-autorate/testing/lua-threads/sqm-autorate-setup.sh)"
   ```
4. If the setup script gives a warning about a configuration file `sqm-autorate-NEW`, use that file to replace `/etc/config/sqm-autorate`
5. When the setup script completes, edit the config file `/etc/config/sqm-autorate` to set:
   * `transmit_kbits_base` to the "nominal" upload speed that your connection provides on a good day
   * `receive_kbits_base` to the "nominal" download speed
   * `transmit_kbits_min` to the lowest upload rate you would accept when controllling bufferbloat
   * `receive_kbits_min` to the lowest acceptable download rate.
   
   Note that these four values are in kilobits/second. If you want the value to be 30 megabits/second, enter `30000`.
   
   Note too that the script uses "acceptable" rates as its lowest setting it will use to control latency.
In certain situations, the script may transition abruptly to either of these lower limits.
Set these values high enough to avoid cutting off your communications entirely.
A good choice might be 15-20% of the nominal rates for mid-range to high-speed connections (above 20 Mbps).
For very slow connections (below 1Mbps) use 50% of the nominal rate.
6. Run these commands to
start and enable the _sqm-autorate_ service that runs continually:

   ```
   service sqm-autorate enable && service sqm-autorate start
   ```

### Requests to Testers

Please post your overall experience on this
[OpenWrt Forum thread.](https://forum.openwrt.org/t/cake-w-adaptive-bandwidth/108848/312)
Your feedback will help improve the script for the benefit of others.

Bug reports and/or feature requests [should be added on Github](https://github.com/Fail-Safe/sqm-autorate/issues/new/choose) to allow for proper prioritization and tracking.

Read on to learn more about how the _sqm-autorate_ algorithm works,
and the [More Details](#More_Details) section for troubleshooting.

## What to expect

In normal operation, _sqm-autorate_ monitors the utilization and latency
of the wide area interface,
and adjusts the parameters to the CAKE algorithm to maximize throughput
while minimizing latency.

On fixed speed connections from the ISP, _sqm_autorate_ won't make much difference.
However, on varying speed links, such as LTE modems or even
cable modems where the traffic rates vary widely by time of day,
_sqm_autorate_ can continually adjust the parameters so that you
get the fastest speed available, while keeping latency low.

**Note:** This script "learns" over time and takes
somewhere between 30-90 minutes to "stabilize".
You can expect some
initial latency spikes when first running this script.
These will smooth out over time.

### Why is _sqm-autorate_ necessary?

The CAKE algorithm does a good job of managing latency for
fixed upload and download bandwidth settings,
that is for ISPs that offer relatively constant speed links.
Variable bandwidth connections present a challenge because
the actual bandwidth at any given moment is not known.

In the past, people using CAKE generally picked
a compromise bandwidth setting,
a setting lower than the maximum speed available from the ISP.
This compromise is hardly ideal:
it meant lost bandwidth in exchange for latency control.
If the compromise setting is too low,
the connection is unnecessarily throttled back
to the compromise setting (yellow);
if the setting is too high, CAKE will still buffer
too much data (green) and induce unwanted latency.
(See the image below.)

With _sqm-autorate_, the CAKE settings are adjusted for current conditions,
keeping latency low while always giving the maximum available throughput.

![Image of Bandwidth Compromise](.readme/Bandwidth-Compromise.png)

## About the Lua implementation

**sqm-autorate.lua** is a Lua implementation of an SQM auto-rate algorithm and it employs multiple [preemptive] threads to perform the following high-level actions in parallel:

- Ping Sender
- Ping Receiver
- Baseline Calculator
- Rate Controller
- Reflector Selector

_For Test builds, Jan 2022:_ In its current iteration this script can react poorly under conditions with high latency and low load, which can force the rates down to the minimum.
If this happens to you, please try to adjust the
`max_delta_owd` variable to a higher value. And be sure to set your minimum speeds to something reasonable so that your connection isn't shut off almost entirely.

The functionality in this Lua version is a culmination of progressive iterations to the original shell version as introduced by @Lynx (OpenWrt Forum). ~~Refer to the [Original Shell Version](#original-shell-version) (below) for details as to the original goal and theory.~~

### Lua Threads Algorithm

Per @dlakelan (OpenWrt Forum):

The script operates in essentially three "normal" regimes (and one unfortunate tricky regime):
1) Low latency, low load: in this situation, the script just monitors latency leaving the speed setting constant. If you don't stress the line much, then it can stay constant for long periods of time. As long as latency is controlled, this is normal.
2) Low latency, high load: As the load increases above 80% of the current threshold, the script opens up the threshold so long as latency stays low. In order to find what is the true maximum it is expected that it will increase so long as latency stays low. When it starts much lower than the nominal rate, the increase is exponential, and this gradually tapers off to become linear as it increases into "unknown territory". As it increases the threshold, it constantly updates a database of actual loads at which it was able to increase. So it learns what speeds normally allow it to increase. The script may choose rates above the nominal "base" rates, and even above what you know your line can handle. This is ok because:
3) High latency, high load: When the load increases beyond what the ISP's connection can actually handle, latency will increase and the script will detect this through the pings it sends continuously. When this occurs, it will use its database of speeds to try to pick something that will be below the true capacity. It ensures that this value is also always less than 0.9 times the current actual transfer rate, ensuring that the speed plummets extremely rapidly (at least exponentially). This can be seen as discontinuous drops in the speed, typically choking off latency below the threshold rapidly. However:
4) There is no way for the script to easily distinguish between high latency and low load because of a random latency fluctuation vs because the ISP capacity suddenly dropped. Hence, if there are random increases in latency that are not related to your own load, the script will plummet the speed threshold rapidly down to the minimum. Ensure that your minimum really is acceptably fast for your use! In addition, if you experience random latency increases even without load, try to set the trigger threshold higher, perhaps to 20-30ms or more.

### Algorithm In Action

Examples of the algorithm in action over time.

_Needs better narrative to explain these charts,
and possibly new charts showing new axis labels._

![Down Convergence](.readme/9e03cf98b1a0d42248c19b615f6ede593beebc35.gif)

![Up Convergence](.readme/5a82f679066f7479efda59fbaea11390d0e6d1bb.gif)

![Fraction of Down Delay](.readme/7ef21e89d37447bf05fde1ea4ba89a4b4b74e1f9.png)

![Fraction of Up Delay](.readme/6104d5c3f849d07b00f55590ceab2363ef0ce1e2.png)

## More Details

### Removal

_(We hope that you will never need to uninstall this autorate program, but if you want to...)_
Run the following removal script to remove the operational files:

```bash
sh -c "$(wget -q -O- https://raw.githubusercontent.com/Fail-Safe/sqm-autorate/testing/lua-threads/sqm-autorate-remove.sh)"
```

### Configuration

Generally, configuration should be performed via the `/etc/config/sqm-autorate` file.

#### Config File Options

| Section | Option Name | Value Description | Default |
| - | - | - | - |
| network | transmit_interface | The transmit interface name which is typically the physical device name of the WAN-facing interface. | 'wan' |
| network | receive_interface | The receive interface name which is typically created as a virtual interface when CAKE is active. This typically begins with 'ifb4' or 'veth'. | 'ifb4wan' |
| network | transmit_kbits_base | The highest speed in kbit/s at which bufferbloat typically is non-existent for outbound traffic on the given connection. This is used for reference in determining safe speeds via learning, but is not a hard floor or ceiling. | '10000' |
| network | receive_kbits_base | The highest speed in kbit/s at which bufferbloat typically is non-existent for inbound traffic on the given connection. This is used for reference in determining safe speeds via learning, but is not a hard floor or ceiling. | '10000' |
| network | transmit_kbits_min | The absolute minimum outbound speed in kbits/s the autorate algorithm is allowed to fall back to in cases of extreme congestion. | '1500' |
| network | receive_kbits_min | The absolute minimum inbound speed in kbits/s the autorate algorithm is allowed to fall back to in cases of extreme congestion. | '1500' |
| network | reflector_type | This is intended for future use and details are TBD. | 'icmp' |
| output | log_level | Used to set the highest level of logging verbosity. e.g. setting to 'INFO' will output all log levels at the set level or lower (in terms of verbosity). [Verbosity Options](#verbosity-options) | 'INFO' |
| output | stats_file | The location to which the autorate OWD reflector stats will be written. | '/tmp/sqm-autorate.csv' |
| output | speed_hist_file | The location to which autorate speed adjustment history will be written. | '/tmp/sqm-speedhist.csv' |
| output | hist_size | The amount of "safe" speed history which the algorithm will maintain for reference during times of increased latency/congestion. | '100' |

Advanced users may override values (following comments) directly in `/usr/lib/sqm-autorate/sqm-autorate.lua` as comfort level dictates.

#### Manual Execution (for Testing and Tuning)

For testing/tuning, invoke the `sqm-autorate.lua` script from the command line:

```bash
# Use these optional PATH settings if you see an error message about 'vstruct'
export LUA_CPATH="/usr/lib/lua/5.1/?.so;./?.so;/usr/lib/lua/?.so;/usr/lib/lua/loadall.so"
export LUA_PATH="/usr/share/lua/5.1/?.lua;/usr/share/lua/5.1/?/init.lua;./?.lua;/usr/share/lua/?.lua;/usr/share/lua/?/init.lua;/usr/lib/lua/?.lua;/usr/lib/lua/?/init.lua"

# Run this command to execute the script
lua /usr/lib/sqm-autorate/sqm-autorate.lua
```

(_Is this next paragraph correct?_) When you run a speed test, you should see the `current_dl_rate` and
`current_ul_rate` values change to match the current conditions.
They should then drift back to the configured download and update rates
when the link is idle.

The script logs information to `/tmp/sqm-autorate.csv` and speed history data to `/tmp/sqm-speedhist.csv`.
See the [Verbosity](#Verbosity_Options) options (below)
for controlling the logging messages.

#### Service Execution (for Steady-State Execution)

As noted above, the setup script installs `sqm-autorate.lua` as a service,
so that it starts up automatically when you reboot the router.

```bash
service sqm-autorate enable && service sqm-autorate start
```

### Output and Monitoring

#### View of Processes

A properly running instance of sqm-autorate will indicate seven
total threads when viewed (in a thread-enabled view) `htop`.
Here is an example:

![Image of Htop Process View](.readme/htop-example.png)

Alternatively, in the absense of `htop`, one can find the same detail with this command:

```bash
# cat /proc/$(ps | grep '[sqm]-autorate.lua' | awk '{print $1}')/status | grep 'Threads'
Threads:    7
```

#### Verbosity Options

The script can output statistics about various internal variables to the terminal. To enable higher levels of verbosity for testing and tuning, you may toggle the following setting:

```bash
local enable_verbose_baseline_output = false
```

The overall verbosity of the script can be adjusted via the `option log_level` in `/etc/config/sqm-autorate`.

The available values are one of the following, in order of decreasing overall verbosity:

- TRACE
- DEBUG
- INFO
- WARN
- ERROR
- FATAL

#### Log Output

- **sqm-autorate.csv**: The location to which the autorate OWD reflector stats will be written. By default, this file is stored in `/tmp`.
- **sqm-speedhist.csv**: The location to which autorate speed adjustment history will be written. By default, this file is stored in `/tmp`.

#### Output Analysis

Analysis of the CSV outputs can be performed via MS Excel, or more preferably, via Julia (aka [JuliaLang](https://julialang.org/)). The process to analyze the results via Julia looks like this:

1. Clone this Github project to a computer where Julia is installed.
2. Copy (via SCP or otherwise) the `/tmp/sqm-autorate.csv` and `/tmp/sqm-speedhist.csv` files within the `julia` sub-directory of the cloned project directory.
3. [First Time Only] In a terminal:
    ```bash
    cd <github project dir>/julia
    julia
    using Pkg
    Pkg.activate(".")
    Pkg.instantiate()
    include("plotstats.jl")
    ```
4. [Subsequent Executions] In a terminal:
    ```bash
    cd <github project dir>/julia
    julia
    include("plotstats.jl")
    ```
5. After some time, the outputs will be available as PNG and GIF files in the current directory.

#### Error Reporting Script

The `/usr/lib/getstats.sh` script in this repo writes a lot
of interesting information to `tmp/openwrtstats.txt`.
You can send portions of this file with
your trouble report.

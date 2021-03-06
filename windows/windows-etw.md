
Event Tracing for Windows (ETW)
===============================

## System Tools

### Event Keywords and Levels

For **manifest-based** providers set MatchAnyKeywords to 0x00 to receive all events. Otherwise you need to create a bitmask which will be or-ed with event keywords. Additionally when MatchAllKeywords is set, its value is used for events that passed the MatchAnyKeywords test and providers additional and filtering.

For **classic providers** set MatchAnyKeywords to 0xFFFFFFFF to receive all events.

Up to 8 sessions may collect manifest-based provider events, but only 1 session may be created for a classic provider (when a new session is created the provider switches to the session).

When creating a session we may also specify the event's level:

- `TRACE_LEVEL_CRITICAL 0x1`
- `TRACE_LEVEL_ERROR 0x2`
- `TRACE_LEVEL_WARNING 0x3`
- `TRACE_LEVEL_INFORMATION 0x4`
- `TRACE_LEVEL_VERBOSE 0x5`

### Enum providers

List all providers: **logman query providers**

List provider details: `logman query providers ".NET Common Language Runtime"`

With logman you can also query providers in a given process: 

```
logman query providers -pid 808
```

You use logman or wevtutil: `wevtutil ep`

Find MSMQ publishers: `wevtutil ep | findstr /i msmq`

using Powershell: `Get-WinEvent -ListProvider`

### Extract details about a given provider

`wevtutil gp Microsoft-Windows-MSMQ /ge /gm /f:xml`

### Recording trace in logman

The following commands start and stop a tracing session that is using one provider:

`logman start mysession -p {9744AD71-6D44-4462-8694-46BD49FC7C0C} -o c:\temp\test.etl -ets & timeout -1 & logman stop mysession -ets`

For the provider options you may additionally specify the keywords (flags) and levels that will be logged:  -p provider [flags [level]]

You may also use a file with a list of providers:

`logman start mysession -pf providers.guids -o c:\temp\test.etl -ets & timeout -1 & logman stop mysession -ets`

And the providers.guids file content is: {guid} {flags} {level} [provider name]

Example for ASP.NET:

<code>{AFF081FE-0247-4275-9C4E-021F3DC1DA35} 0xf    5  ASP.NET Events
{3A2A4E84-4C21-4981-AE10-3FDA0D9B0F83} 0x1ffe 5  IIS: WWW Server</code>

If you want to record events from the kernel provider you need to name the session: "NT Kernel Logger", eg.:

`logman start "NT Kernel Logger" -p "Windows Kernel Trace" "(process,thread,file,fileio)" -o c:\kernel.etl -ets & timeout -1 & logman stop "NT Kernel Logger" -ets`

### Convert .etl file to .evtx

`tracerpt -of EVTX test.etl -o test.evtx -summary test-summary.xml`

Dump etl file to xml:  `tracerpt test.etl -o test.xml -summary test-summary.xml`

To dump etl file to a text file you may use: `xperf -i test.etl -o test.csv`

## XPerf

### Tracing with xperf

For kernel tracing you just need to specify kernel flags or a kernel group: `xperf -on DiagEasy`

In user-mode tracing you may still use kernel flags and groups, but for each user-trace provider you need to add some additional parameters: `-on (GUID|KnownProviderName)[:Flags[:Level[:0xnnnnnnnn|'stack|[,]sid|[,]tsid']]]`

To stop run: `xperf -stop [session-name] -d c:\temp\merged.etl`

The best option is to combine all the commands together, eg.:

`xperf -on latency -stackwalk profile -buffersize 2048 -MaxFile 1024 -FileMode Circular && timeout -1 && xperf -d C:\highCPUUsage.etl`

### Collecting stack walk events

To be able to view stack traces of the ETW events we need to enable special stack walk events collections. For kernel events there is a special **-stackwalk** switch. For user providers it's more complicated and requires a special **:::'stack'** string to be appended to the provider's name (or GUID).

To get more info on kernel stack walk settings execute `xperf -help stackwalk` command.

To be able to see stack traces remember to enable PROC\_THREAD and LOADER kernel flags as they are required to correctly decode the modules in the trace file. Otherwise you will see ?!? symbols instead of valid strings. The next thing to check is that you merged the events from your trace session with the kernel rundown information on the machine where the ETW trace capture was performed (by using the ???d command-line option of Xperf).

### Query provider information

Using the -providers switch in xperf you can get a list of installed/known and registered providers, as well as all known kernel flags and groups: `xperf -providers [Installed|I] [Registered|R] [PerfTrack|PT] [KernelFlags|KF] [KernelGroups|KG] [Kernel|K]`

xperf also lists kernel flags that may be used in collection of the kernel events:

### Query active sessions

You can use `xperf -loggers`

### Trace file postprocessing with xperf

Based on http://randomascii.wordpress.com/2014/02/04/process-tree-from-an-xperf-trace/

You may generate a process tree from xperf using `xperf -i foo.etl -a process -tree` command. Other analysis command allow you to extract thread stacks, modules etc.

### Boot tracing

from http://www.fixitpc.pl/topic/11092-analiza-dlugiego-startu-systemu/

1. Make sure stackwalking is disabled: `REG ADD "HKLM\System\CurrentControlSet\Control\Session Manager\Memory Management" -v DisablePagingExecutive -d 0x1 -t REG\_DWORD -f`

2. Start tracing (best for traces use another drive then the one being analysed): `xbootmgr -trace boot -traceflags latency+dispatcher -stackwalk profile+cswitch+readythread -notraceflagsinfilename -postbootdelay 180 -resultpath d:\temp`

3. After booting your file should be created.

## PerfView

See perfview /?

### Collect traces from the command line

To collect traces into a 500MB file (in circular mode) run the following command:

`perfview -AcceptEULA -ThreadTime -CircularMB:500 -Circular:1 -LogFile:perf.output -Merge:TRUE -Zip:TRUE -noView  collect`

A new console window will open with the following text:

<code>Pre V4.0 .NET Rundown enabled, Type 'D' to disable and speed up .NET Rundown.
Do NOT close this console window.   It will leave collection on!
Type S to stop collection, 'A' will abort.  (Also consider /MaxCollectSec:N)</code>

Type 'S' when you are done with tracing and wait (DO NOT CLOSE THE WINDOW) till you see `Press enter to close window`. Then copy the files: PerfViewData.etl.zip and perf.output to the machine when you will perform analysis.

If you are also interested in the network traces append the -NetMonCapture option. This will generate additional PerfViewData_netmon.cab file which you may open in the Message Analyzer.

Open Perfview trace in WPA: `perfview /wpr unzip test.etl.zip`

This should create two files (.etl and .etl.ngenpdb): `wpa test.etl`

### Issues

**(0x800700B7): Cannot create a file when that file already exists**

If you receive:

<code>[Kernel Log: C:\tools\PerfViewData.kernel.etl]
    Kernel keywords enabled: Default
    Aborting tracing for sessions 'NT Kernel Logger' and 'PerfViewSession'.
    Insuring .NET Allocation profiler not installed.
    Completed: Collecting data C:\tools\PerfViewData.etl   (Elapsed Time: 0,858 sec)
    Exception Occured: System.Runtime.InteropServices.COMException (0x800700B7): Cannot create a file when that file already exists. (Exception from HRESULT: 0x800700B7)
       at System.Runtime.InteropServices.Marshal.ThrowExceptionForHRInternal(Int32 errorCode, IntPtr errorInfo)
       at Microsoft.Diagnostics.Tracing.Session.TraceEventSession.EnableKernelProvider(Keywords flags, Keywords stackCapture)
       at PerfView.CommandProcessor.Start(CommandLineArgs parsedArgs)
       at PerfView.CommandProcessor.Collect(CommandLineArgs parsedArgs)
       at PerfView.MainWindow.c__DisplayClass9.b__7()
       at PerfView.StatusBar.c__DisplayClass8.b__6(Object param0)
    An exceptional condition occurred, see log for details.
</code>

make sure that no kernel log is running: `perfview listsessions`

and eventually kill it: `perfview abort`

## WPA and WPR

### Problems with symbols loading

Copy dbghelp.dll and symsrv.dll from the Debugging Tools for Windows to the WPA installation folder (remember about the correct bitness).

### Get information on running profiles

To find active WPR recording you can run: `wpr -status profiles`

### Profiling using predefined profiles

To start profiling with CPU,FileIO and DiskIO profile run: 

`wpr -start CPU -start FileIO -start DiskIO`

To save the results run: 

`wpr -stop C:\temp\result.etl`

To completely turn off wpr logging run: `wpr -cancel`.

### Profiling using custom profiles

Start tracing: 

`wpr.exe -start GeneralProfile -start Audio -start circular-audio-glitches.wprp!MediaProfile -filemode`

Stop tracing and save the results to a file (say, my-wpr-glitches.etl):  

`wpr.exe -stop my-wpr-glitches.etl` 

(Optional) if you want to cancel tracing: `wpr.exe -cancel`

(Optional) if you want to see whether tracing is currently active:  `wpr.exe -status`

### Profiling a system boot

To collect general profile traces use: 

`wpr -start generalprofile -onoffscenario boot -numiterations 1`

All options are displayed after executing: `wpr -help start`.

### WPR schema analysis

I guess that the name in the Profile tag is used to group the profiles for the UI. Those names are displayed when we call wpr -profiles. Interestingly WPR finds the most thorough profile from the available profiles in a given wprp file.

### Reading stack

When analyzing traces we have the usual stack available, but you may often see an option to display stack tags. Those are special values which replace the well-known stack frames with tags. This way we can group the events by a stack tag and have a better understand which types of operation built a given request.

The default stacktag file is here: c:\Program Files (x86)\Windows Kits\10\Windows Performance Toolkit\Catalog\default.stacktags
---
keywords: powershell,cmdlet
locale: en-us
ms.date: 11/26/2019
online version: 1.0.0
schema: 2.0.0
title: about_Thread_Jobs
---

# About Thread Jobs

## SHORT DESCRIPTION

Provides information about PowerShell thread-based jobs. A thread job is a type
of background job that runs a command or expression in a separate thread within
the current session process.

## LONG DESCRIPTION

This article explains how to run thread jobs in PowerShell on a local computer.
For information about running background jobs on a local computer, see
[about_Jobs](about_Jobs.md).

Start a thread job by using the `Start-ThreadJob` cmdlet. This cmdlet is
available in the **ThreadJob** module that ships with PowerShell.
`Start-ThreadJob` returns a single job object that encapsulates the running
command or script, and can be used with all PowerShell job manipulating cmdlets.

## THE JOB CMDLETS

|Cmdlet           |Description                                            |
|-----------------|-------------------------------------------------------|
|`Start-ThreadJob`|Starts a thread job on a local computer.               |
|`Get-Job`        |Gets the jobs that were started in the current session.|
|`Receive-Job`    |Gets the results of jobs.                              |
|`Stop-Job`       |Stops a running job.                                   |
|`Wait-Job`       |Suppresses the command prompt until one or all jobs are|
|                 |complete.                                              |
|`Remove-Job`     |Deletes a job.                                         |

## HOW TO START A THREAD JOB ON THE LOCAL COMPUTER

To start a thread job on the local computer, use the `Start-ThreadJob` cmdlet.

To write a `Start-ThreadJob` command, enclose the command or script the job runs
in curly braces (`{ }`).

The following command starts a thread job that runs a `Get-Process` command on
the local computer.

```powershell
Start-ThreadJob -ScriptBlock { Get-Process }
```

The `Start-ThreadJob` command returns a `ThreadJob` object that represents the
running job. The job object contains useful information about the job including
its current running status. It collects the results of the job as the results
are being generated.

## HOW TO WAIT FOR A JOB TO COMPLETE AND RETRIEVE JOB RESULTS

You can use PowerShell job cmdlets, such as `Wait-Job` and `Receive-Job` to wait
for a job to complete and then return all results generated by the job.

The following command starts a thread job that runs a `Get-Process` command,
then waits for the command to complete, and finally returns all data results
generated by the command.

```powershell
Start-ThreadJob -ScriptBlock { Get-Process } | Wait-Job | Receive-Job
```

## POWERSHELL CONCURRENCY AND JOBS

PowerShell concurrently runs commands and script through jobs. There are three
jobs-based solutions provided by PowerShell to support concurrency.

|Job            |Description                                                  |
|---------------|-------------------------------------------------------------|
|`RemoteJob`    |Command and script run on a remote computer.                 |
|`BackgroundJob`|Command and script run in a separate process on the local    |
|               |machine.                                                     |
|`ThreadJob`    |Command and script run in a separate thread within the same  |
|               |process on the local machine.                                |

Each type of job has benefits and drawbacks. Running script remotely on a
separate machine or in a separate process has great isolation. Any errors won't
affect other running jobs or the client that started the job. But the remoting
layer adds overhead, including object serialization. All objects passed to and
from the remote session must be serialized and then deserialized as it passes
between the client and the target session. The serialization operation can use
many compute and memory resources for large complex data objects.

## POWERSHELL THREAD BASED JOBS

Thread based jobs are not as robust as Remote and Background jobs, because they
run in the same process on different threads. If one job has a critical error
that crashes the process, then all other jobs in the process will also fail.

However, thread-based jobs have much less overhead. They don't need to use the
remoting layer or serialization. The result is that thread-based jobs tend to
run much faster and use far less resources than the other job types.

## THREAD JOB PERFORMANCE

Thread jobs are faster and lighter weight than other types of jobs. But they
still have overhead that can be large when compared to work the job is doing.

PowerShell runs commands and script in a session. Only one command or script can
run at a time in a session. So when running multiple jobs, each job runs in a
separate session. Each session contributes to the overhead.

Thread jobs will provide the best performance when the work they perform is
greater than the overhead of the session used to run the job. There are
typically two cases for good performance.

- Work is compute intensive  
Running script on multiple thread jobs can take advantage of multiple process
cores, and complete faster.

- Work consists of significant waiting  
Script that spends time waiting for I/O or remote call results. Running in
parallel will usually complete much quicker than if run sequentially.

```powershell
(Measure-Command {
    1..1000 | ForEach { Start-ThreadJob { Write-Output "Hello $using:_" } } | Receive-Job -Wait
}).TotalMilliseconds
10457.962


(Measure-Command {
    1..1000 | ForEach-Object { "Hello: $_" }
}).TotalMilliseconds
24.9277
```

The first example above shows a foreach loop that creates 1000 thread jobs to do
a simple string write. Due to job overhead, it takes over 33 seconds to
complete.

The second example runs the `ForEach` cmdlet to do the same 1000 operations and
each string write is executed sequentially without any job overhead. It
completes in a mere 25 milliseconds.

```powershell
$logNames.count
10

Measure-Command {
    $logs = $logNames | ForEach {
        Start-ThreadJob {
            Get-WinEvent -LogName $using:_ -MaxEvents 5000 2>$null
        } -ThrottleLimit 10
    } | Wait-Job | Receive-Job
}

TotalMilliseconds : 115994.3 (1 minute 56 seconds)
$logs.Count
50000
```

In the above example, up to 5000 entries are collected for 10 separate system
logs. Since the script involves reading a number of logs, it makes sense to do
the operations in parallel. And the job completes over twice as fast as when the
script is run sequentially.

```powershell
Measure-Command {
    $logs = $logNames | ForEach-Object {
        Get-WinEvent -LogName $_ -MaxEvents 5000 2>$null
    }
}

TotalMilliseconds : 252398.4321 (4 minutes 12 seconds)
$logs.Count
50000
```

## THREAD JOBS AND VARIABLES

Variables are passed into thread jobs in various ways.

`Start-ThreadJob` can accept variables that are piped to the cmdlet, passed in
to the script block via the `$using` keyword, or passed in via the
`-ArgumentList` parameter.

```powershell
$msg = "Hello"

$msg | Start-ThreadJob { $input | Write-Output } | Wait-Job | Receive-Job

Start-ThreadJob { Write-Output $using:msg } | Wait-Job | Receive-Job

Start-ThreadJob { param ([string] $message) Write-Output $message } -ArgumentList @($msg) | Wait-Job | Receive-Job
```

Since thread jobs run in the same process, any variable reference type passed
into the job has to be treated carefully. If it is not a thread safe object,
then it should never be assigned to, and method and properties should never be
invoked on it.

```powershell
$threadSafeDictionary = [System.Collections.Concurrent.ConcurrentDictionary[string,object]]::new()
$jobs = Get-Process | ForEach {
    Start-ThreadJob {
        $proc = $using:_
        $dict = $using:threadSafeDictionary
        $dict.TryAdd($proc.ProcessName, $proc)
    }
}
$jobs | Wait-Job | Receive-Job

$threadSafeDictionary.Count
96

$threadSafeDictionary["pwsh"]

NPM(K) PM(M) WS(M) CPU(s) Id SI ProcessName
------ ----- ----- ------ -- -- -----------
112 108.25 124.43 69.75 16272 1 pwsh
```

The above example passes a thread safe dotNet `ConcurrentDictionary` object to
all child jobs to collect uniquely named process objects. Since it is a thread
safe object, it can be safely used while the jobs run concurrently in the
process.
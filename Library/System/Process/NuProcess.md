# NuProcess

[GitHub](https://github.com/brettwooldridge/NuProcess)

A low-overhead, non-blocking I/O, external Process execution implementation for Java. It is a replacement for java.lang.ProcessBuilder and java.lang.Process.
- Java 7+

----

`LinuxProcess start()`
```mermaid
sequenceDiagram
  participant LP as LinuxProcess
  participant H as NuProcessHandler
  participant NP as NativeProcess
  #participant BasePosixProcess
  #participant NuProcessHandler
  Note over LP: callsPreStart()
  LP->>H: onPreStart()
  H-->>LP: log Exception
  Note over LP: prepareProcess()
  LP->>NP: forkAndExec()
  break Exception
    LP--)H: onExit(Integer.MIN_VALUE)<br>on Exception
  end
  Note over LP: - create pendingWrites queue<br/>- create buffers for stdin/stdout/stderr<br/>- maybe sleep for "nuprocess.test.afterStartSleep" nanos<br>- set isRunning=true
  create actor EP as ProcessEpoll
  LP->EP: register process on one of the static IEventProcessors, and start if not running.
  Note over LP: callStart()
  LP->>H: onStart()
  H-->>LP: log Exception
```

----

`LinuxProcess run()`
```mermaid
sequenceDiagram
  participant LP as LinuxProcess
  participant H as NuProcessHandler
  participant NP as NativeProcess
  #participant BasePosixProcess
  #participant NuProcessHandler
  Note over LP: callsPreStart()
  LP->>H: onPreStart()
  H-->>LP: log Exception
  Note over LP: prepareProcess()
  LP->>NP: forkAndExec()
  break Exception
  LP--)H: onExit(Integer.MIN_VALUE)<br>on Exception
  end
  Note over LP: - create pendingWrites queue<br/>- create buffers for stdin/stdout/stderr<br/>- maybe sleep for "nuprocess.test.afterStartSleep" nanos<br>- set isRunning=true
  create actor EP as ProcessEpoll
  LP->>EP: new ProcessEpoll(this)
  Note over EP: registerProcess()<br/>- get fds for stdin,stdout,stderr<br>- put process in pidToProcessMap<br>- put fds in fildesToProcessMap
  Note over EP: checkAndSetRunning()
  Note over LP: callStart()
  LP->>H: onStart()
  H-->>LP: log Exception
  LP->>EP: run()
  Note over EP: - await startBarrier<br/>- process() until isRunning false or Exception 
```

----

`BasePosixProcess stdin wantWrite` Tells process to wait for stdin
```mermaid
sequenceDiagram
  participant LP as LinuxProcess
  participant EP as ProcessEpoll
  Note over LP: wantWrite()<br/>- if stdin.acquire() != -1, then<br>userWantsWrite = true, then
  LP-->>EP: queueWrite(this)
```

----

`BasePosixProcess writeStdin` Queues up data to send to process.
```mermaid
sequenceDiagram
  participant LP as LinuxProcess
  participant EP as ProcessEpoll
  Note over LP: writeStdin(ByteBuffer)<br/>- if stdin.acquire() != -1, then<br>pendingWrites.add(buffer), then
  LP-->>EP: queueWrite(this)
```

----

`BasePosixProcess closeStdin` Tells process to no more stdin
```mermaid
sequenceDiagram
  participant LP as LinuxProcess
  participant EP as ProcessEpoll
  Note over LP: closeStdin(ByteBuffer)<br/>- if stdin.acquire() != -1, then<br>pendingWrites.add(STDIN_CLOSED_PENDING_WRITE_TOMBSTONE), then
  LP-->>EP: queueWrite(this)
```

----

`ProcessEpoll process()`
```mermaid
sequenceDiagram
  participant EP as ProcessEpoll
  participant LP as LinuxProcess
  participant P as LibEpoll
  break
  Note over EP: no events
  end
  critical poll LibEpoll for events
  option No events
  option EPOLLIN event and stdout indent
  EP->>LP: readStdout(BUFFER_CAPACITY)
  option not EPOLLIN event and stderr indent
  EP->>LP: readStderr(BUFFER_CAPACITY)
  option EPOLLOUT event and stdin open
  EP->>LP: writeStdin(BUFFER_CAPACITY)
  option (EPOLLHUP or EPOLLRDHUP or EPOLLERR event)
  critical indent test
  option stdout
    EP->>LP: readStdout(-1)
  option stderr
    EP->>LP: readStderr(-1)
  option stdin
    EP->>LP: closeStdin(true)
    end
  end 
```

----


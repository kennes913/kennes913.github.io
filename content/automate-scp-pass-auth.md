+++
title = "control scp with python"
date = 2020-03-01
+++

A common task in day-to-day engineering is trasferring files from host to host. The optimal tooling choice in many cases is a binary like **rsync** or **scp**. What would this look like if we used python to control these lower level binaries instead? Can you get anything out of doing this?

*TL;DR; The short answer is yes. The 1 main benefit you get is being able to use the niceties of python's flow control to handle lower level binaries.*

In this example, we automate password authentatication. You can do this with python using [pexpect](https://github.com/pexpect/pexpect). pexpect is _a pure python module for spawning child applications; controlling them; and responding to expected patterns in their output_. The idea is that we can use the pexpect API to control system binaries and manipulate them using python's flow control.

Here is some sample code I've written that moves target files to a remote host:

```python
...
@log
def put(
    files: str, arget_host: str, dest_fpath: str, user: str, password: str, recursive=False,
) -> Generator[Union[str, pexpect.EOF], None, None]:
    """Create a pty process that uses SCP to move a single file or set of files
    to a remote SFTP server.
    """
    # -v enables verbose logging
    # -p preserves modification, access times, and modes of file
    args: list = ["-v", "-p"]

    if recursive:
        # Match a directory or glob match contents
        if os.path.isdir(files):
            args.append("-r")
        else:
            raise TypeError(
                f"Recursive mode enabled but {files} is not directory. "
                f"Unexpected behavior will occur."
            )

    args.append(files)  # add source files
    args.append(f"{user}@{target_host}:{dest_fpath}")  # add destination path

    with pexpect.spawn("scp", args, timeout=120, encoding="utf-8") as pty:
        auth = False
        while True:
            try:
                pat = pty.expect(["Password:", "\(yes\/no\)\?", "\n"])
                if not auth:
                    # Authentication case
                    if pat == 0:
                        pty.sendline(password)
                        auth = True
                    # Host verification
                    if pat == 1:
                        pty.sendline("yes")
                yield pty.before
            except pexpect.EOF:
                yield pexpect.EOF
                break
    return 0

```

Here are the rough steps of the example:
1. spawn an application running scp
2. handle each buffered line of the output
3. `yield` each line to a logging decorator

The interesting part of this whole thing is handling **pexpect**-ed things. Typically, scp requires manual user authentication or else the process halts, but we handle this easily with pexpect (using regex). You can imagine extending this to other binaries that are typically restricted by manual user inputs.

Here is some log output from the above process:

```bash
19:52:38 forked.pty.process | 2020-02-09 19:52:38.640 | debug1: Authentication succeeded (keyboard-interactive).
19:52:38 forked.pty.process | 2020-02-09 19:52:38.640 | Authenticated to super.secure.host.com ([10.32.0.111]:22).
19:52:38 forked.pty.process | 2020-02-09 19:52:38.641 | debug1: channel 0: new [client-session]
19:52:38 forked.pty.process | 2020-02-09 19:52:38.641 | debug1: Entering interactive session.
19:52:38 forked.pty.process | 2020-02-09 19:52:38.641 | debug1: pledge: network
19:52:38 forked.pty.process | 2020-02-09 19:52:38.642 | debug1: Sending environment.
19:52:38 forked.pty.process | 2020-02-09 19:52:38.642 | debug1: Sending env LANG = en_US.UTF-8
19:52:38 forked.pty.process | 2020-02-09 19:52:38.642 | debug1: Sending command: scp -v -p -t some_file.csv
19:52:38 forked.pty.process | 2020-02-09 19:52:38.687 | File mtime 1581277956 atime 1581277956
19:52:38 forked.pty.process | 2020-02-09 19:52:38.687 | Sending file timestamps: T1581277956 0 1581277956 0
19:52:38 forked.pty.process | 2020-02-09 19:52:38.688 | Sending file modes: C0644 2670 some_file.csv
19:52:39 forked.pty.process | 2020-02-09 19:52:39.028 |
some_file.csv                     0%    0     0.0KB/s   --:-- ETA
some_file.csv                   100% 2670     8.0KB/s   00:00
19:52:39 forked.pty.process | 2020-02-09 19:52:39.029 | debug1: client_input_channel_req: channel 0 rtype exit-status reply 0
19:52:39 forked.pty.process | 2020-02-09 19:52:39.071 | debug1: channel 0: free: client-session, nchannels 1
19:52:39 forked.pty.process | 2020-02-09 19:52:39.072 | debug1: fd 0 clearing O_NONBLOCK
19:52:39 forked.pty.process | 2020-02-09 19:52:39.072 | debug1: fd 1 clearing O_NONBLOCK
19:52:39 forked.pty.process | 2020-02-09 19:52:39.072 | Transferred: sent 5744, received 2688 bytes, in 0.4 seconds
19:52:39 forked.pty.process | 2020-02-09 19:52:39.072 | Bytes per second: sent 13346.0, received 6245.5
19:52:39 forked.pty.process | 2020-02-09 19:52:39.072 | debug1: Exit status 0
19:52:39 forked.pty.process | 2020-02-09 19:52:39.073 | File transfer complete. Closing pexpect.spawn().
```

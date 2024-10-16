# Design Spec: The *rnr* Job Runner

This spec covers a prototype job-worker service that provides a [gRPC][grpc] API
to run arbitrary Linux processes, and a command-line interface to the same.
It is part of the take-home coding challenge used by [Teleport][tlpt], Level 5.

## Requirements & Assumptions

As stated in the [requirements][reqs] the system consists of:

1. A reusable library implementing the functionality for working with jobs.
2. An API server that wraps the functionality of the library.
3. A command line interface (CLI) that communicates with the API server.

We assume:

* Unlimited CPU, memory and disk are available.
* State does not persist across server restarts.

## Implementation Plan

### Library

The reusable library handles the running of Linux jobs under their own
cgroups and namespaces.  Authorization logic is included in the library, but
is optional for running jobs. (The caller may use different authorization,
or none at all.)

__The job-lifecycle features must run as root on Linux with cgroups v2.__

Reasonable measures will be taken to ensure that unit tests do not require
root privileges.  Jobs will be run as user `nobody` by default.

> __Rationale:__ being root gives us a shortcut around some quite complex
> (possibly even insurmountable) problems making cgroups and especially
> namespaces work: in particular, mounts and networking for namespaces.
> 
> With cgroups, v2 has two advantages that outweigh the fact that most
> available documentation is for v1: first, v2 is the [default][cgdistros] for
> the newer versions of major distros, and is favored by [Kubernetes][cgk8s];
> second, the v2 hierarchy makes creating cgroups here much simpler.

#### Per-Job Resource Limits (cgroups)

Each job is run under a new cgroup, which is a child of an overall cgroup
controlling resources for the whole set of jobs.  Specific limits can be set
by the caller, but defaults are available and are relatively low, in order to
observe the limits in action.

The cgroup controls will apply to `cpu.max`, `memory.max` and `io.max`.

The cgroup is removed when the job finishes or upon orderly shutdown of the
process.

The standard cgroup virtual directory is used.

> __Rationale:__ the requirement is "per job" but not necessarily _different_
> for each job.  Creating a new cgroup is necessary in order to have the
> limits apply to the one process (if this is not correct, please include a
> reference).
>
> __Future:__ it would be great to have a limit-setting API, in fact my first
> draft had exactly that but it was trimmed as out of scope.  Ideally there
> would be a hierarchy of app > user > process limits, and the ability to
> use any limits allowed by the parent cgroup.

#### Namespace isolation

Jobs will be started with isolated namespaces using cloning, with the mount,
networking and PID isolation set up using [reexec][reexec] approximately as
described in the tutorial by [Ed King][edking].

Thus each job process should only "see" itself.

> __Rationale:__ after much digging, I was unable to find a reliable method of
> doing `setns` and what I did find was neither simpler nor more isolated than
> this solution. Forking and namespaces are a major incompatibility between
> Linux and Go.

#### Process Output and Streaming

The STDOUT and STDERR output of each process will be captured in a struct from
which a gRPC streaming endpoint can read.

It will return one line of output at a time, with a received timestamp and an
indicator of which handle was used. Thus, STDOUT and STDERR are streamed
together.

Due to the line-by-line output, behaviour in the case of very long lines or
non-text data is undefined and may not be useful.  The last output from a
command, when the channel is closed, will be sent regardless of whether it
has a newline.

By default, the server prints all output to its own STDOUT as it arrives.

> __Rationale:__ cutting corners.  Chunking out data by other means is more
> predictable; having an option for binmode is very useful (you'd need it to
> download a file); separate streams for STDOUT and STDERR would be nifty;
> but all that goes against the simplicity directive.
> 
> Having the server log everything is basic good security manners, but also
> useful for testing: the stream and the server logs should obviously agree.
>
> If we did not have infinite CPU and memory, writing the command output to
> persistent storage would be better.  It would be fun to benchmark SQLite3
> for that.

#### Job Lifecycle

First a job is __started__:

* The structures to monitor the job are created.
* The job is executed with [reexec][reexec] magic for namespace isolation.
* A cgroup is created for it and its PID is added to that cgroup.
* It is registered internally as running.

> __Danger:__ it should be possible to run a job that completes, or causes
> damage, before the cgroup is created and the PID added to it.  So far this
> appears to be accepted behavior, but I will at least try to roll the cgroup
> actions into the reexec init stage, which theoretically (?) would solve the
> problem.  Proving this would be very hard either way.

Then its __status__ may be queried at will to confirm the process is still
running.

If a client wants to __stream output__ then the structs holding Stdout and
Stderr for the process are read and streamed to the client.

To __stop__ a job, first we check whether it's still running.  If so, we send
it either a SIGTERM or (only at the client's request) a SIGKILL and report
the result.

Stopped processes are still available for __streaming__ as long as the server
is running (we have infinite CPU etc).

However, the cgroup virtual directory is removed as soon as the server is
aware that a job is no longer running.  These virtual directories are also
removed on an orderly shutdown of the server, as running jobs are terminated
or killed at that stage.

### gRPC Server

The gRPC server will be built with the standard grpc codegen methods from the
[jobsapi/jobsapi.proto](jobsapi/jobsapi.proto) spec file.

Functions will be fairly thin wrappers around the Library.

#### Authentication

Authentication is via mTLS, with users identified by email as a User Principal
Name (UPN) [embedded][upns] within the Subject Alternative Name (SAN) of the
client certificate.  The user thus extracted is used for authorization.

> __Rationale:__ this appears to be the most "standard" way of embedding a
> user in a non-email mTLS client cert, and it's more than enough for our
> purposes.  A potential downside is that you could have many active certs
> for a given user, but since we who are authing should control the cert
> issuance, I don't see this as a big deal.  It could even be a feature, if
> we wanted to embed roles etc -- think of how some APIs have per-app login
> credentials.

Authentication will use TLS 1.3 and the `TLS_AES_256_GCM_SHA384` cipher
suite for maximum future-proofing.

> __Note:__ as I understand it, I will be able to set this up precisely in
> the grpc `advancedtls` package; if I'm wrong, some details may change, but
> I can't imagine it not supporting 1.3 given the improved security and
> performance.

Certificate generation and management is out of scope of the system, but
helper scripts for prototype cert generation will be included.

#### Authorization

A user is *allowed* to run a set of know processes, or all processes in a
directory or set of directories.  This is conceptually structured as a map:

```go
allowed := map[string][]string{
    "tim@tlpt.com": []string{"/usr/local/bin/","/opt/teleport/bin/foo"},
    "joe@tlpt.com": []string{"/usr/bin/perl"},
}
```

When a request is made to `StartJob` then the user's permission to run the
requested job will be checked.

A system for managing these permissions over gRPC is _out of scope,_ but they
can be set with command-line options when starting the server.

> __Rationale:__ the writ was *simple* and I find this to be also useful.
> Not implementing the management falls under the "cut corners" directive.
> However it's very simple to do, so I'm happy to add it if desired!
> 
> __Note:__ it could be nice to limit the view of a process (e.g. to stream)
> to its submittor; however there is also a use-case for being more liberal,
> and that is if a subordinate is queuing jobs for you and you want to monitor
> them yourself.  Granular permissions for all that are far out of scope, and
> with no requirement one way or the other, I propose to let you stream the
> output of my jobs if you know the PID.
> 
> __Future:__ have admins who can set permissions (and probably limits too)
> per user, or for groups, etc.


#### Endpoints

##### StartJob

Starts a job, returning its PID on success.  The PID is the job identifier for
the other endpoints.

> __Note:__ I *think* that from the server's point of view the PIDs will be
> unique and unaffected by the namespacing, but if I'm wrong -- if the reexec
> messes with the PID in the parent namespace for instance -- then this may
> need to change.

##### GetJobStatus

Returns the status of the job: running, stopped, error.

##### StopJob

Stops a job via `SIGTERM` by default or, optionally, `SIGKILL`.

##### StreamJobOutput

Streams the output of a job, starting from the beginning of output and
continuing until all output is streamed.

> __Note:__ if we did not have infinite memory and CPU, we would obviously
> need a RemoveStoppedJob or equivalent, because you need to stream even if a
> job has already completed.

### Command-Line Interface

Starting the server with various settings:

```sh
$ rnr serve # all defaults
2024/11/10 23:00:00 rnr listening on :8088
2024/11/10 23:01:00 Job from tim@tlprt.co has PID 1234: /usr/bin/echo hello world
2024/11/10 23:01:01 1234: STDOUT: hello world
^C
$ rnr serve --port=8080 --allow='tim@tlprt.co:/usr/bin/:/opt/' --noecho
rnr listening on :8080
2024/11/10 23:01:01 Job from tim@tlprt.co has PID 1234: /usr/bin/echo hello world
```

Very simple job submission with default settings as echoed above:
```sh
$ rnr run /usr/bin/echo hello world
PID: 1234
```

Submitting a job with default settings and monitoring then stopping it:

```sh
$ rnr run /usr/bin/perl -- -nE 'say $$; sleep 1' /dev/random
PID: 2345
$ rnr stream 2345
2024/11/10 23:01:01 2345: STDOUT: 7734
2024/11/10 23:01:02 2345: STDOUT: 7734
2024/11/10 23:01:03 2345: STDOUT: 7734
2024/11/10 23:01:04 2345: STDOUT: 7734
^C
$ rnr status 2345
Status: running
$ rnr stop 2345
Exit 1
$ 
```


<!-- LINKS: -->

[grpc]: https://grpc.io/
[tlpt]: https://goteleport.com/
[reqs]: https://github.com/gravitational/careers/blob/main/challenges/systems/challenge-1.md
[reexec]: https://pkg.go.dev/github.com/docker/docker/pkg/reexec
[cgdistros]: https://github.com/opencontainers/runc/blob/main/docs/cgroup-v2.md
[cgk8s]: https://kubernetes.io/docs/concepts/architecture/cgroups/
[edking]: https://medium.com/@teddyking/namespaces-in-go-basics-e3f0fc1ff69a
[upns]: https://docs.venafi.com/Docs/24.1/TopNav/Content/Certificates/r-UEP-support-SANs.php

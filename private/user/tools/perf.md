# Hardware Performance Counters

Hardware Performance Counters in general can be accessed via the Linux [`perf_event`](https://www.kernel.org/doc/html/latest/admin-guide/perf-security.html) subsystem. Access from within containers is possible when installing the necessary user space tools, namely `perf` for OpenSUSE and `linux-perf` for Ubuntu.

## Access

The Linux `perf_event` subsystem distinguishes access levels on the basis of the [`perf_event_paranoid`](https://man7.org/linux/man-pages/man2/perf_event_open.2.html) system setting. For security reasons, OpenCUBE sets this parameter to `2`. This implies that `perf` can only perform performance monitoring on a per-process level, system-wide measurements are not possible. 

## Usage

You can use `perf_event` either via the C interface or via the `perf` utility. The latter is restricted to monitoring of specific processes either directly or via the `--pid` parameter:

```bash
user@containerssh-user-12345:~> perf stat sleep 1
 Performance counter stats for 'sleep 1':

              0.98 msec task-clock:u                     #    0.001 CPUs utilized
                 0      context-switches:u               #    0.000 /sec
                 0      cpu-migrations:u                 #    0.000 /sec
                66      page-faults:u                    #   67.328 K/sec
            455458      instructions:u                   #    1.09  insn per cycle
                                                  #    0.25  stalled cycles per insn
            417585      cycles:u                         #    0.426 GHz
            112549      stalled-cycles-frontend:u        #   26.95% frontend cycles idle
            103065      stalled-cycles-backend:u         #   24.68% backend cycles idle
            100182      branches:u                       #  102.197 M/sec
              7135      branch-misses:u                  #    7.12% of all branches

       1.002151883 seconds time elapsed

       0.002069000 seconds user
       0.000000000 seconds sys

user@containerssh-user-12345:~> sleep 5 &
[1] 80
user@containerssh-user-12345:~> perf stat --pid "$!"

 Performance counter stats for process id '80':
 [...]
```

As outlined above, system-wide monitoring is not supported:

```bash
user@containerssh-user-12345:~> perf stat
Error:
Access to performance monitoring and observability operations is limited.
[...]
```
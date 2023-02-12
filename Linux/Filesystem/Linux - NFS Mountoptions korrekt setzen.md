
# Abstract

Diese Seite beschreibt die korrekten Mountoptionen für NFS Mounts unter Linux.

# Quelle

Die Informationen auf dieser Seite stammen aus dem Oracle's Whitepaper Optimizing Storage for Oracle Database 11g Reelase 2 with the Oracle ZFS Storage Appliance und aus der MOS Note "Mount Options for Oracle files for RAC databases and Clusterware when used with NFS on NAS devices" (Doc ID 359515.1)

# Mountoptions

| --- | --- | --- |
|​Installation & OS | Filetypen | ​Mountoptions|
​|Single Instanz Linux | DB ​Datafiles | ​rw,bg,hard,rsize=1048576,wsize=1048576,nointr,timeo=600,tcp,actimeo=0,noac,vers=3|
​​|Single Instanz Linux & RAC unter Linux | admin-FS oder usr-local-FS | ​​rw,bg,hard,rsize=32768,wsize=32768,nointr,timeo=600,tcp,actimeo=0,noac,vers=3|
​|Single Instanz Linux | Oracle Binaries | rw,bg,hard,rsize=1048576,wsize=1048576,nointr,tcp,actimeo=600,noac,vers=3|
​|RAC unter Linux | DB ​Datafiles | rw,bg,hard,nointr,rsize=1048576,wsize=1048576,timeo=600,tcp,actimeo=0,noac,vers=3|
​​|RAC unter Linux | Oracle Binaries | ​rw,bg,hard,nointr,rsize=1048576,wsize=1048576,timeo=600,tcp,actimeo=0,noac,vers=3|
|​​​|RAC unter Linux | CRS Voting Disk oder OCR | ​rw,bg,hard,nointr,rsize=rsize=1048576,wsize=1048576,tcp,noac,timeo=600,actimeo=0,noac,vers=3
​|Linux | ​RMAN Backup Files | rw,bg,hard,nointr,rsize=1048576,wsize=1048576,tcp,noac,timeo=600,vers=3|
​|Engineered Systems mit Infiniband | ​RMAN Backup Files | ​rw,bg,hard,nointr,rsize=1048576,wsize=1048576,tcp,timeo=600,noac,vers=3|

Bei nicht korrekten NFS Optionen (v.a. ohne "actimeo=0") kann es zu masiven DB-Hängern und komischen Timeout-Phänomenen (z.B. TNS-12535 bei Data Guard) kommen!

# Relevante NFS Optionen

Auszug aus 'man nfs':

       soft / hard    Determines the recovery behavior of the NFS client after an NFS request times out.  If neither option is specified (or if the hard option is specified),  NFS  requests  are

                      retried  indefinitely.   If  the  soft option is specified, then the NFS client fails an NFS request after retrans retransmissions have been sent, causing the NFS client to

                      return an error to the calling application.

                      NB: A so-called "soft" timeout can cause silent data corruption in certain cases. As such, use the soft option only when client responsiveness is more important  than  data

                      integrity.  Using NFS over TCP or increasing the value of the retrans option may mitigate some of the risks of using the soft option.

       timeo=n        The time in deciseconds (tenths of a second) the NFS client waits for a response before it retries an NFS request.

                      For NFS over TCP the default timeo value is 600 (60 seconds).  The NFS client performs linear backoff: After each retransmission the timeout is increased by timeo up to the

                      maximum of 600 seconds.

                      However, for NFS over UDP, the client uses an adaptive algorithm to estimate an appropriate timeout value for  frequently  used  request  types  (such  as  READ  and  WRITE

                      requests),  but uses the timeo setting for infrequently used request types (such as FSINFO requests).  If the timeo option is not specified, infrequently used request types

                      are retried after 1.1 seconds.  After each retransmission, the NFS client doubles the timeout for that request, up to a maximum timeout length of 60 seconds.

       rsize=n        The maximum number of bytes in each network READ request that the NFS client can receive when reading data from a file on an NFS server.  The actual data  payload  size  of

                      each NFS READ request is equal to or smaller than the rsize setting. The largest read payload supported by the Linux NFS client is 1,048,576 bytes (one megabyte).

                      The  rsize  value  is  a  positive  integral  multiple of 1024.  Specified rsize values lower than 1024 are replaced with 4096; values larger than 1048576 are replaced with

                      1048576. If a specified value is within the supported range but not a multiple of 1024, it is rounded down to the nearest multiple of 1024.

                      If an rsize value is not specified, or if the specified rsize value is larger than the maximum that either client or server can support, the client and server negotiate the

                      largest rsize value that they can both support.

                      The  rsize  mount  option as specified on the mount(8) command line appears in the /etc/mtab file. However, the effective rsize value negotiated by the client and server is

                      reported in the /proc/mounts file.

       wsize=n        The maximum number of bytes per network WRITE request that the NFS client can send when writing data to a file on an NFS server. The actual data payload size  of  each  NFS

                      WRITE request is equal to or smaller than the wsize setting. The largest write payload supported by the Linux NFS client is 1,048,576 bytes (one megabyte).

                      Similar  to  rsize , the wsize value is a positive integral multiple of 1024.  Specified wsize values lower than 1024 are replaced with 4096; values larger than 1048576 are

                      replaced with 1048576. If a specified value is within the supported range but not a multiple of 1024, it is rounded down to the nearest multiple of 1024.

                      If a wsize value is not specified, or if the specified wsize value is larger than the maximum that either client or server can support, the client and server negotiate  the

                      largest wsize value that they can both support.

                      The  wsize  mount  option as specified on the mount(8) command line appears in the /etc/mtab file. However, the effective wsize value negotiated by the client and server is

                      reported in the /proc/mounts file.

       ac / noac      Selects whether the client may cache file attributes. If neither option is specified (or if ac is specified), the client caches file attributes.

                      To improve performance, NFS clients cache file attributes. Every few seconds, an NFS client checks the server���s version of each file���s attributes for updates.  Changes that

                      occur  on  the  server in those small intervals remain undetected until the client checks the server again. The noac option prevents clients from caching file attributes so

                      that applications can more quickly detect file changes on the server.

                      In addition to preventing the client from caching file attributes, the noac option forces application writes to become synchronous so that local changes to  a  file  become

                      visible on the server immediately.  That way, other clients can quickly detect recent writes when they check the file���s attributes.

                      Using  the  noac  option provides greater cache coherence among NFS clients accessing the same files, but it extracts a significant performance penalty.  As such, judicious

                      use of file locking is encouraged instead.  The DATA AND METADATA COHERENCE section contains a detailed discussion of these trade-offs.

       actimeo=n      Using actimeo sets all of acregmin, acregmax, acdirmin, and acdirmax to the same value.  If this option is not specified, the NFS client uses the defaults for each of these

                      options listed above.

       acregmin=n     The minimum time (in seconds) that the NFS client caches attributes of a regular file before it requests fresh attribute information from a server.  If this option  is  not

                      specified, the NFS client uses a 3-second minimum.

       acregmax=n     The  maximum  time (in seconds) that the NFS client caches attributes of a regular file before it requests fresh attribute information from a server.  If this option is not

                      specified, the NFS client uses a 60-second maximum.

       acdirmin=n     The minimum time (in seconds) that the NFS client caches attributes of a directory before it requests fresh attribute information from a server.   If  this  option  is  not

                      specified, the NFS client uses a 30-second minimum.

       acdirmax=n     The  maximum  time  (in  seconds)  that the NFS client caches attributes of a directory before it requests fresh attribute information from a server.  If this option is not

                      specified, the NFS client uses a 60-second maximum.

       bg / fg        Determines  how  the  mount(8)  command behaves if an attempt to mount an export fails.  The fg option causes mount(8) to exit with an error status if any part of the mount

                      request times out or fails outright.  This is called a "foreground" mount, and is the default behavior if neither the fg nor bg mount option is specified.

                      If the bg option is specified, a timeout or failure causes the mount(8) command to fork a child which continues to attempt to mount  the  export.   The  parent  immediately

                      returns with a zero exit code.  This is known as a "background" mount.

                      If  the local mount point directory is missing, the mount(8) command acts as if the mount request timed out.  This permits nested NFS mounts specified in /etc/fstab to pro-

                      ceed in any order during system initialization, even if some NFS servers are not yet available.  Alternatively these issues can be addressed using an automounter (refer  to

                      automount(8) for details).

       intr / nointr  Selects whether to allow signals to interrupt file operations on this mount point. If neither option is specified (or if nointr is specified), signals do not interrupt  NFS

                      file operations. If intr is specified, system calls return EINTR if an in-progress NFS operation is interrupted by a signal.

                      Using the intr option is preferred to using the soft option because it is significantly less likely to result in data corruption.

                      The  intr / nointr mount option is deprecated after kernel 2.6.25.  Only SIGKILL can interrupt a pending NFS operation on these kernels, and if specified, this mount option

                      is ignored to provide backwards compatibility with older kernels.

# Tags:

[[HowTo]] - [[Linux]] - [[Filesystem]]
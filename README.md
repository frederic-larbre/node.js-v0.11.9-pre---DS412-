# How to install node.js v0.11.9-pre on a Synology DS412+

Installing latest node.js on a Synology DS412+ is really a pain in the ass. I have spent almost a day doing this and that is what is motivating me for sharing this.

All along this post I am assuming installing node.js on a __Intel Atom x86 64bits__ hardware. 

## Installing

Connect to your Synology using `ssh` and __root__ account:

	ssh root@your_synology_ip_address
	
### ipkg

`ipkg` is the package manager of your Synology. we need to install it as we require some packages that do not come along with your original installation.

__ipkg bootstrap__ is the script installing `ipkg`. You have to download and install __bootstrap__ according to your synology hardware version.  
Synology hardware/CPU details can be found [here][cpu].
__bootstrap__ versions repositories can be found [here][bootstrap].

	wget http://ipkg.nslu2-linux.org/feeds/optware/syno-i686/cross/unstable/syno-i686-bootstrap_1.2-7_i686.xsh
	sh syno-i686-bootstrap_1.2-7_i686.xsh

Update and upgrade your packages so as to ensure that your installation is up to date:

	ipkg update
	ipkg upgrade
	
### Prior compiling node.js
Some prerequities have to be meet...

#### Get working libraries
Install __optware__ development libraries:

	ipkg install optware-devel

Remove the not compatible `wget` package, you will remain on the `wget-ssl` package:

	ipkg remove wget
	
Install packages __pythons 2.7__ and  __openssl__

	ipkg install python27 open-ssl openssl-dev
	cd /opt/bin
	ln -s python2.7 python

The __libpthread__ library you just installed is well known for being corrupted. You will not be able to compile node.js with this library. Replace it this way:

	mkdir /opt/i686-linux-gnu/lib_disabled
	mv /opt/i686-linux-gnu/lib/libpthread* /opt/i686-linux-gnu/lib_disabled
	cp /lib/libpthread.so.0 /opt/i686-linux-gnu/lib/
	cd /opt/i686-linux-gnu/lib/
	ln -s libpthread.so.0 libpthread.so
	ln -s libpthread.so.0 libpthread-2.5.so
	
#### Get node.js sources
Install `git`:

	ipkg install git
	
Download latest node.js sources:

	git clone https://github.com/joyent/node.git

Configure the build:

	cd node
	./configure --prefix=/usr/local/node --without-snapshot
	
#### Fix node.js sources towards your installation

Alter the following three source files according to the `diff` files. I did not gave a `patch` file since it will be obsolete in a minute or so, feel free to create your own based on the diff samples.

##### diff for __deps/uv/include/uv-unix.h__  

	diff --git a/deps/uv/include/uv-unix.h b/deps/uv/include/uv-unix.h
	index 965fbaf..85a4d3e 100644
	--- a/deps/uv/include/uv-unix.h
	+++ b/deps/uv/include/uv-unix.h
	@@ -37,9 +37,7 @@
	 
	 #include <semaphore.h>
	 #include <pthread.h>
	-#ifdef __ANDROID__
	 #include "pthread-fixes.h"
	-#endif
	 #include <signal.h>
	 
	 #if defined(__linux__) 

##### diff for __deps/uv/src/unix/thread.c__

	diff --git a/deps/uv/src/unix/thread.c b/deps/uv/src/unix/thread.c
	index 8c38c7f..da58872 100644
	--- a/deps/uv/src/unix/thread.c
	+++ b/deps/uv/src/unix/thread.c
	@@ -276,15 +276,8 @@ int uv_cond_init(uv_cond_t* cond) {
	   pthread_condattr_t attr;
	   int err;
	 
	-  err = pthread_condattr_init(&attr);
	-  if (err)
	-    return -err;
	-
	-#if !defined(__ANDROID__)
	-  err = pthread_condattr_setclock(&attr, CLOCK_MONOTONIC);
	-  if (err)
	-    goto error2;
	-#endif
	+  if(pthread_condattr_init(&attr))
	+    return -1;
	 
	   err = pthread_cond_init(cond, &attr);
	   if (err)

##### diff for __deps/v8/src/platform/condition-variable.cc__

	diff --git a/deps/v8/src/platform/condition-variable.cc b/deps/v8/src/platform/condition-variable.cc
	index e2bf388..f7b6f8f 100644
	--- a/deps/v8/src/platform/condition-variable.cc
	+++ b/deps/v8/src/platform/condition-variable.cc
	@@ -48,8 +48,6 @@ ConditionVariable::ConditionVariable() {
	   pthread_condattr_t attr;
	   int result = pthread_condattr_init(&attr);
	   ASSERT_EQ(0, result);
	-  result = pthread_condattr_setclock(&attr, CLOCK_MONOTONIC);
	-  ASSERT_EQ(0, result);
	   result = pthread_cond_init(&native_handle_, &attr);
	   ASSERT_EQ(0, result);
	   result = pthread_condattr_destroy(&attr);

#### Set `gcc` __POSIX__ and general compatibility flags
You will need to build with the following flags:

* __\_XOPEN\_SOURCE=700__ for __POSIX__ compatibility
* __\_GNU\_SOURCE__ since it is not set on the complete build process and required on Synology
* __ULONG\_LONG\_MAX=18446744073709551615ULL__ as a consequence of the __POSIX__ setting  
 
Set the __CFLAGS__ environment variable:

	CFLAGS="-D_XOPEN_SOURCE=700 -D_GNU_SOURCE -DULONG_LONG_MAX=18446744073709551615ULL"
	export CFLAGS
	
### Compiling and installing node.js

	make
	make install
	
You should be done.


[cpu]: http://forum.synology.com/wiki/index.php/What_kind_of_CPU_does_my_NAS_have "Synology hardware/CPU"
[bootstrap]: http://forum.synology.com/wiki/index.php/Overview_on_modifying_the_Synology_Server,_bootstrap,_ipkg_etc#How_to_install_ipkg "Synology bootstrap repositories"



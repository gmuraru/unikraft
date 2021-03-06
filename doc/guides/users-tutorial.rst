************************************
Tutorial
************************************

For this tutorial you will need a Linux host running Xen and/or KVM in
order to run Unikraft images. Please check the manual of your Linux
distribution regarding how to install these environments. This
tutorial expects that you have the essential build and debugging tools
installed (see :ref:`rst_users_getting_started`). In addition, for
Xen you will need to have the ``xl`` toolstack installed and running,
and for KVM ``qemu``

===========================
Cloning Repositories
===========================
You can easily build Unikraft Unikernels on your Linux host. If you
have all tools and libraries installed to compile a Linux kernel you
are ready to do this with Unikraft too.

A Unikraft build consists mostly of a combination of multiple
repositories. We differentiate them into: (1) Unikraft, (2) external
libraries, (3) application.  The build system assumes these to be
structured as follows: ::

  my-workspace/
  ├── apps/
  │   └── helloworld/
  │   └── httpreply/
  ├── libs/
  │   ├── lwip/
  │   └── newlib/
  └── unikraft/

Clone the following repositories with git ::

  git clone [URL] [DESTINATION PATH]

* Unikraft base repository directly under your workspace root
   * `Unikraft repo <https://github.com/unikraft/unikraft.git>`_
* External libraries into a `libs` subdirectory:
	* `newlib repo <https://github.com/unikraft/lib-newlib.git>`_
	* `lwip repo <https://github.com/unikraft/lib-lwip.git>`_
* Applications into an `apps` subdirectory:
	* `helloworld repo <https://github.com/unikraft/app-helloworld.git>`_
	* `httpreply repo <https://github.com/unikraft/app-httpreply.git>`_

Make sure that the directory structure under your workspace is exactly
the same as shown in the overview ahead.

===========================
Your First Unikernel
===========================

-------------------
Configuring
-------------------
After you cloned the repos, go to the ``helloworld`` application and
run ``make menuconfig`` to configure the build. Unikraft uses the same
configuration system as the Linux kernel (Kconfig). We will build
Unikraft images for Xen, KVM, and Linux, so the first step is to go to
the ``Platform Configuration`` option and make the following changes:

* select ``Xen guest image``
* select ``KVM guest``
* select ``Linux user space``
  
Under ``Library configuration`` we also need to choose a scheduler:
select ``ukschedcoop``.

-------------------
Building
-------------------
Save your configuration and build the image by typing ``make``. The
build system will create three binaries, one for each platform: ::

  $ ls -sh build/
   [...]
   88K helloworld_kvm-x86_64
   40K helloworld_linuxu-x86_64
   72K helloworld_xen-x86_64
   [...]

-------------------
Running
-------------------

Let's execute the unikernel.

* The easiest is to run the one built as a Linux user space
  application. It should execute on any Linux environment: ::

   $ ./build/helloworld_linuxu-x86_64
   Welcome to  _ __             _____
    __ _____  (_) /__ _______ _/ _/ /_
   / // / _ \/ /  '_// __/ _ `/ _/ __/
   \_,_/_//_/_/_/\_\/_/  \_,_/_/ \__/
                     Titan 0.2~10ce3f2
   Hello world!

* You can execute the KVM image (``helloworld_kvm-x86_64``) on the KVM
  host: ::

   $ qemu-system-x86_64 -nographic -vga none -device sga -m 4 -kernel
   build/helloworld_kvm-x86_64

* For Xen you first need to create a VM configuration (save it under
  ``helloworld.cfg``): ::
  
   name          = 'helloworld'
   vcpus         = '1'
   memory        = '4'
   kernel        = 'build/helloworld_xen-x86_64'

Start the virtual machine with: ::

  $ xl create -c helloworld.cfg

---------------------------------
Modifying the Application
---------------------------------
After ``Hello world!`` is printed, the unikernel shuts down right
away. We do not have a chance to see that a VM was actually created,
so let's modify the source code. Open ``main.c`` in your favorite
editor (``nano``, ``vim``, ``emacs``) and add the following busy loop
inside the ``main`` function: 

.. code-block:: c
		
   for (;;);

Rebuild the images with ``make`` and execute them. The shell prompt
should not return. With a second shell you can check that the
unikernel is still executing:

* Use ``top`` or ``htop`` for Linux and KVM.
* Use ``xl top`` in Xen.

**Note**: You can terminate the KVM and Linux unikernel with
 ``CTRL`` + ``C``, and on Xen with ``CTRL`` + ``]``.


===========================
External Libraries
===========================

The ``helloworld`` application uses a very minimalistic ``libc``
implementation of libc functionality called ``nolibc`` which is part
of the Unikraft base, and so it is an "internal" library. Internal
libraries are located within the ``lib`` directory of Unikraft.

In order to enhance the functionality provided by Unikraft, "external"
libraries can be added to the build. In the following we want to swap
``nolibc`` with `newlib <https://github.com/unikraft/lib-newlib>`_, a
standard libc implementation that you can find in various Linux
distributions and embedded environments.

We need to add newlib to the library includes. Edit the ``Makefile``
of the ``helloworld`` application and put the text below in it. Please
type ``make properclean`` before; this will delete the build directory
(but not your configuration) and will force a full rebuild later. ::

  diff --git a/Makefile b/Makefile
  --- a/Makefile
  +++ b/Makefile
  @@ -1,6 +1,6 @@
   UK_ROOT ?= $(PWD)/../../unikraft
   UK_LIBS ?= $(PWD)/../../libs
  -LIBS :=
  +LIBS := $(UK_LIBS)/newlib
  
   all:
          @make -C $(UK_ROOT) A=$(PWD) L=$(LIBS)

Run ``make menuconfig``; ``newlib`` should now appear in the ``Library
Configuration`` menu. Select it, save and exit the menu, and type
``make``. Unikraft's build system will download newlib's sources and
build it together with all the other Unikraft libraries and
application. Our ``newlib`` repository consists only of glue code that
is needed to port ``newlib`` to Unikraft.

You will notice that the unikernels are now bigger than before. Try to
run them again.


=========================
Code Your Own Library
=========================
Let's add some functionality to our unikernel. Create a directory
``libs/mylib``, this will be the root folder of your library.

As mentioned earlier, Unikraft uses Linux's kconfig system. In order
to make your library selectable in the "menuconfig", create the file
``Config.uk`` with the following content: ::

  config LIBMYLIB
             bool "mylib: My awesome lib"
             default n

To test if it worked, we need to tell Unikraft's build system to pick
this library. Go back to your ``helloworld`` application and edit it
its ``Makefile``. Earlier we added newlib to the ``LIBS`` variable,
let's now add the new library: ::

  LIBS := $(UK_LIBS)/newlib:$(UK_LIBS)/mylib

Now if you run ``make menuconfig`` you should see your library under
the "Library Configuration" sub-menu: ::

  [ ] mylib: My awesome lib

Select it, exit the configuration menu, and save the changes. If you
run ``make`` right now, the build will produce an error about a
missing ``Makefile.uk``: ::

  make[1]: *** No rule to make target '/root/demo/libs/mylib/Makefile.uk'.  Stop.

Go back to your library directory and create one with the following
content: ::

  # Register your lib to Unikraft's build system
  $(eval $(call addlib_s,libmylib,$(CONFIG_LIBMYLIB)))

  # Add library source code to compilation
  LIBMYLIB_SRCS-y += $(LIBMYLIB_BASE)/mylib.c

  # Extend the global include paths with library's folder
  CINCLUDES-$(CONFIG_LIBMYLIB) += -I$(LIBMYLIB_BASE)/include

And finally the library code ``mylib.c``:

.. code-block:: c
		
  #include <stdio.h>
  
  void libmylib_api_func(void)
  {
          printf("Hello from my awesome lib!\n");
  }

Now in your helloworld's ``main.c`` add a call to
``libmylib_api_func()``.


=========================
Socket Example
=========================
As a last task, we are going to build a small webserver that replies
with a single page. The server uses ``lwip`` for creating a socket and
to accept incoming connections. Go to the ``httpreply`` application
directory. Have a look at ``main.c``: it is a really short program and
looks similar to what you would write as a user-space Linux
program. Its dependencies are defined within ``Config.uk``. Having
this, there is actually not much left to configure. Any mandatory
options are locked in ``make menuconfig``. All we need to do is select
our target platforms, select network drivers, save the config, and
type ``make``.

For now, we support virtio for networking only (but more functionality
is coming). You can enable the driver by going to the KVM platform
configuration and selecting ``Virtio PCI device support`` and ``Virtio
Net device``.

The image can be started on the KVM host. Replace ``br0`` with the
name of your local bridge on your system and make sure you have a DHCP
server listening there (e.g., ``dnsmasq``): ::

  $ qemu-system-x86_64 -nographic -vga none -device sga -m 8 -netdev bridge,id=en0,br=br0 -device virtio-net-pci,netdev=en0 -kernel build/httpreply_kvm-x86_64

Please also ensure that you have built your image with the lwip menu
option "DHCP client" enabled. This unikernel is requesting an IPv4
address via DHCP. In case you enabled ``ICMP`` in the ``lwip``
configuration, you should also be able to ping the host from a second
terminal (replace the IP with yours): ::

  $ ping 192.168.1.100

For debugging, you can also try to enable ``Debug messages`` in
``lwip``. With this you can now have a deeper look in the network
stack.

If networking is working well, you can use the text-based browser
``lynx`` (or any other that you like) to see the web page served on a
second terminal (replace the IP with yours): ::

  $ lynx 192.168.1.100:8123


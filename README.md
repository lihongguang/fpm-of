This is a hack of a simple program that plays the rold of a Forwarding Plane Manager and prints out messages received from the RIB in the format of ovs-ofctl commands.

*fpm-of* can be built as follows:

<pre>
  $ make QUAGGA_DIR=&lt;location-of-quagga-code&gt;
</pre>

The *QUAGGA\_DIR* variable is required because *fpm\_of* depends on the
header file that defines the FPM interface (*fpm/fpm.h*).

To run the program, just invoke it as follows -- it will start
listening for a connection from the RIB.

<pre>
  $ ./fpm-of
</pre>

This will allow the program to connect to the FPM service in Quagga

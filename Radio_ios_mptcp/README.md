# How we modified a radio application on ios to integrate Multipath TCP.

To begin, we downloaded on github an open radio application on ios written in swift language. Then we opened the source code on Xcode (version 9.3) for the 11.3 version of ios.

The first thing to do for MPTCP integration in the application, is to modify in the project and in the developer account the authorisations. We went to the 'Capabilities' section of the project and we activated the 'multipath'. Then we did the same for the developer account. Without these two modifications, the following steps are useless. It is also necessary to activate the 'WifiAssist' in the options of the i-phone for the MPTCP connection.

We seek in the code to find the place where we connect to the server. And we have found the method wich use URLSession to do the request (the application is written in swift but it is possible to find the correspondance with objective-c on the website of apple).

We have found a perfectly fonctional connection which was performing by TCP. We just had to modify the URLSessionConfiguration. The
configuration was initated like the following line.

`let sessionConfiguration : URLSessionConfiguration.default`

We changed the 'default' by 'ephemeral' so the i-phone don't stock the result of the request in cache et we changed more easily check the exchanged packets with the server. The modification which is necessary to pass to MPTCP is trivial and you
can do it with this line of code:

`sessionConfiguration.multipathServiceType = URLSessionConfiguration.multipathServiceType.handover`

There is interactive mode too for MPTCP. For more informations about the different MPTCP modes, see URLSessionConfiguration documentation on Apple's site.

After this, we have changed the address on the server on which we go to have the document .json with the list of the radios.
We have also installed on our server compatible with MPTCP the file .json that interessed us.

Once this step finished, we have observed with 'tcpdump' that the connection was negotiating with the option mptcp_capable and launched
with MPTCP.

There 2 options were in front of us:
1. We could implement our own radio on the server.
2. We could also program the server to relay an other radio. Like that the connection between our phone and the server will be with MPTCP and the server ask for the radio with TCP to the other server.

We were through both.

**For the first one:**
We have installed Icecast2, MPD and MPC on the MPTCP server. Then we have changed the configuration files.
To do so, we opened the files mpd.conf and icecast.xml and we have changed all the addresses ipv4 by the address of our server.
We have also changed the port number because we were blocked by a firewall that we couldn't modify.
We have then add a file .mp3 on the server at the directory indicate by 'music_directory' in the mpd.conf. After that, we have used the
following commands:

`sudo mpc -h [addrOfTheServer] play`

`sudo mpc -h [addrOfTheServer] stop`

to respectively playing and stop the music.

It is played on the port indicate by the file icecast.xml. You just have to listen on this port to receive the radio.
Nevertheless we had a little issue. We didn't succeed to read the flow with the application because our server was reading in .ogg which is an open format.
So we just did a http get request to our server of the file we wanted and we finally succeeded to have it with MPTCP. But it is not a radio and if we want an other file, we would have to do another http get request of the other file.

**For the second option:**
We have done a mirror with a proxy file on the server in the directory nginx/sites-available.
We have made a symbolic link to nginx/sites-enabled with the command 'ln'.
We have tried to connect with MPTCP with the stream but that didn't work. AVAudio didn't allow us to choose ourself the URLSession but uniquely the url. So we have sent a bug report to apple with the suggestion to add this functionnality.

Luis Tascon Gutierrez & Maxime Mawait.

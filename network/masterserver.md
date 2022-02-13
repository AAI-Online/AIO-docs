# AIO masterserver protocol specification
This document explains the networking protocol between:
* Masterserver and AIO clients
* Masterserver and AIO servers

It is used in AIO versions 0.4.x and below. In the upcoming v0.5 it will be scrapped in favor of the new protocol specification that makes use of C structs (python struct library).

This protocol is similar to Attorney Online; it uses the hash and percent symbols `#%` as a delimiter between network messages.

Hash and percent symbols in str arguments are escaped as `<num>` and `<percent>` respectively.

Example: `1#2#%`<br/>
`1` is the message header.<br/>
`2` (and beyond) is considered as the arguments of the message.

## Network messages
* `1`: Sent from masterserver to client upon connecting. Contains client ID. _(Unused)_
    * Client connects
    * Masterserver sends `1#(int: clientID)#%`


* `12`: Server list.
    * Client sends `12#%` to request server list
    * Masterserver replies with `12#(str: name)#(str: description)#(str: ip)#(int: port)#...#%`. Each server contains name, description, ip and port. Length of arguments / 4 is the amount of servers in the list.


* `13`: Publish a server, or update the already published server.
    * Client sends `13#(str: name)#(str: description)#(int: port)#%`
    * Masterserver replies with `SUCCESS#13#%`


* `KEEPALIVE`: Sent by AIO server regularly to tell the masterserver it is still connected.
    * AIO server sends `KEEPALIVE#%`
    * Masterserver replies with `SUCCESS#KEEPALIVE#%`
    * Servers are deleted from the masterserver if 30 seconds pass without a keepalive message during that time.


* `NEWS`: AIO game news requested by AIO client.
    * Client sends `NEWS#%`
    * Server replies with `NEWS#(str: news_html)#%`


* `SUCCESS`: Sent from masterserver to indicate an action was successful.
	* Client or AIO server sends something
	* Masterserver sends `SUCCESS#(str: header)#%`


* `OKNOBO`: Sent from masterserver to indicate there was an error performing an action.
	* Client or AIO server sends something
	* Masterserver sends `OKNOBO#(str: header)#(str: reason)#%`
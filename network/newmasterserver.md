# AIO New masterserver protocol specification
This document explains the networking protocol between:
* Masterserver and AIO clients
* Masterserver and AIO servers

This is used in AIO versions 0.5 and forward.

It makes use of C structs through the python struct library.

## Message header values
```
MS_CONNECTED = 0
MS_NEWS = 1
MS_LIST = 2
MS_PUBLISH = 3
MS_KEEPALIVE = 4
```
_(Header only)_ means that the message contains no extra data after the header.

## Network protocol

First 4 bytes of the network message contains the packet size. The rest of the message received using the packet size is a zlib-compressed string.<br/>
```
import zlib

# For sending:
# imagine you have a "data" variable that contains the data you want to send
data = zlib.compress(data)
data = struct.pack("I", len(data)) + data # add length before data itself

# For receiving:
data = tcp.recv(4) # receive first 4 bytes
size, = struct.unpack("I", data) # read uint32
data = tcp.recv(size) # read the entire packet
data = zlib.decompress(data) # salanto don't die on me yet
# (or you could just do "data = zlib.uncompress(tcp.recv(size))" and save a line of code)
```

### Custom variable types
* `CString8:` String that uses uint8 (unsigned char) to define the length.
    ```
    unsigned char m_length;
    char m_string[m_length];
    ```
* `CString16:` String that uses uint16 (unsigned short) to define the length.
    ```
    unsigned short m_length;
    char m_string[m_length];
    ```

<font size="0.1"><sup>the char variables should probably be `char* m_string;` instead if you make your own code at some point...</sup></font>

### Network messages
* `MS_CONNECTED`: _(Header only)_ Sent from the masterserver to the client upon connection.
    * Client will send `MS_LIST` to request server list.
    * Server will send `MS_PUBLISH` to publish their server.

* `MS_NEWS`: Request game news.
    * Client sends `MS_NEWS` _(Header only_)
    * Masterserver replies with `MS_NEWS` header:
    ```
    # CString16 news;

    import packing
    news = packing.unpackString16(data)
    ```

* `MS_LIST`: Contains the server list.
    * Client sends `MS_LIST` _(Header only_)
    * Masterserver replies with `MS_LIST` header:
    ```
    uint16 amount;
    for i in range(amount): # pretend this is pseudocode instead of some cursed python+C combo
        CString8 ip; # use packing.unpackString8()
        uint16 port;
    ```

* `MS_PUBLISH`: Publish a server to the list.
    * AIO server sends `MS_PUBLISH`:
    ```
    uint16 port;
    ```
    * Masterserver replies with `MS_PUBLISH` to indicate success. _(Header only)_

* `MS_KEEPALIVE`: Sent by AIO server regularly every 20 seconds to keep the server in the list. If not sent before 30 seconds the server is de-listed.
    * AIO server sends `MS_KEEPALIVE` _(Header only)_
    * Masterserver replies with the same header to indicate success, and the timer restarts.
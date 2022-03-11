# AIO New masterserver protocol specification
This document explains the networking protocol between:
* Masterserver and AIO clients
* Masterserver and AIO servers

This is used in AIO versions 0.5 and forward.

It makes use of C structs through the python struct library.

## Message header values
```python
MS_CONNECTED = 0
MS_NEWS = 1
MS_LIST = 2
MS_PUBLISH = 3
MS_KEEPALIVE = 4
MS_SUCCESS = 5
MS_OKNOBO = 6
```
_(Header only)_ means that the message contains no extra data after the header.

## Network protocol

### Initial packet message
First 4 bytes of the network message contains the packet size.<br/>
```python
import zlib
from packing import makeAIOPacket, readAIOHeader

# For sending:
# imagine you have a "data" variable that contains the data you want to send
def sendBuffer(data):
    data = makeAIOPacket(data)
	tcp.send(data)

# you can also specify compression type and the receiver will handle it on its' own:
def sendBuffer(data):
    data = makeAIOPacket(data, 1) # zlib
	tcp.send(data)


# For receiving:
data = tcp.recv(4) # receive first 4 bytes
size, compression = readAIOHeader(data) # get total packet size and compression type
data = tcp.recv(size) # read the entire packet
if compression == 1: # compressed with zlib
	data = zlib.decompress(data) # salanto don't die on me yet
```

After you read the whole packet size and handle compression, unpack the header.
```python
from packing import buffer_read
data, header = buffer_read("B", data) # uint8
# Returned values:
# data: The rest of the message with the byte you just read removed from it
# header: The header byte itself
```

### Custom variable types
* `String8` String that uses uint8 (unsigned char) to define the length.
You can import packing.py and use the functions packString8/unpackString8 to manipulate them.
    ```c
    // C code equivalent:
    unsigned char m_length;
    char m_string[m_length];
    ```
* `String16` String that uses uint16 (unsigned short) to define the length.
You can import packing.py and use the functions packString16/unpackString16 to manipulate them.
    ```c
    // C code equivalent:
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
    ```python
    data, news = packing.unpackString16(data)
    ```

* `MS_LIST`: Contains the server list.
    * Client sends `MS_LIST` _(Header only_)
    * Masterserver replies with `MS_LIST` header:
    ```python
	data, amount = packing.buffer_read("H", data) # uint16
    for i in range(amount):
		data, ip = packing.unpackString8(data)
        data, port = packing.buffer_read("H", data) # uint16
    ```

* `MS_PUBLISH`: Publish a server to the list.
    * AIO server sends `MS_PUBLISH`:
    ```python
	data = struct.pack("B", AIOprotocol.MS_PUBLISH)
    data += struct.pack("H", port) # uint16
	tcp.send(packing.makeAIOPacket(data))
    ```
    * If successful, masterserver replies with header MS_SUCCESS + success type MS_PUBLISH to indicate success.
	* If unsuccessful, masterserver replies with header MS_OKNOBO + error type MS_PUBLISH + reason string.

* `MS_KEEPALIVE`: Sent by AIO server regularly every 20 seconds to keep the server in the list. If not sent before 30 seconds the server is de-listed.
    * AIO server sends `MS_KEEPALIVE` _(Header only)_
    * If a server was published, masterserver replies with header MS_SUCCESS + success type MS_KEEPALIVE, and the timer restarts.
	* If no server was published, masterserver replies with header MS_OKNOBO + error type MS_KEEPALIVE + reason string.
	```python
	data = struct.pack("B", AIOprotocol.MS_KEEPALIVE)
	tcp.send(packing.makeAIOPacket(data)
	```

* `MS_SUCCESS`: Sent by masterserver on a successful action.
    * AIO server sends something.
	* Masterserver replies with:
	```python
	data, successtype = packing.buffer_read("B", data) # this is the header from AIOprotocol that the server first sent
	```

* `MS_OKNOBO`: Sent by masterserver on an unsuccessful action.
    * AIO server sends something.
	* Masterserver replies with:
	```python
	data, errortype = packing.buffer_read("B", data) # this is the header from AIOprotocol that the server first sent
	data, errorReason = packing.unpackString16(data)
	```

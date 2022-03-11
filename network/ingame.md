# AIO Ingame protocol specification
This document explains the networking protocol between:
* AIO client and server

## Message header enum values
Taken from AIOprotocol.py

### For TCP
```python
CONNECT = 1
DESTROY = 2
MSCHAT = 3
EXAMINE = 5
SETZONE = 6
REQUEST = 7
MUSIC = 8
EMOTESOUND = 9
MOVE = 10
CREATE = 11
WARN = 14
KICK = 15
BROADCAST = 16
OOC = 17
EVIDENCE = 18
CHATBUBBLE = 19
SETCHAR = 20
PING = 21
BARS = 22 # penalty bars
WTCE = 23 # witness testimony / cross examination
```

### For UDP
```python
UDP_REQUEST = 0
```

## Custom variable types
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

## Network protocol (TCP)

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

### Network messages
* AIOprotocol.CONNECT:
  * Sent by client first upon connect, contains game version
  * Server replies with info about the client ID and the server itself
  ```python
  # Client:
  buf = struct.pack("B", AIOprotocol.CONNECT)
  buf += packing.packString8(GAME_VERSION)
  sendBuffer(buf) # sendBuffer is a shortcut function to sending the Initial packet message described above.


  # Server:
  # Read the packet and header as described in Initial packet message, then:
  data, version = packing.unpackString8(data)

  # Check version if it matches the server's. if not, kick. Assuming we're good, we reply:
  buf = struct.pack("B", AIOprotocol.CONNECT)
  buf += struct.pack("I", ClientID)
  buf += struct.pack("I", self.maxchars) # Maximum amount of characters in server (not players)
  buf += packing.packString8(self.zonelist[self.defaultzone][0]) # Default zone to spawn in (filename)
  buf += struct.pack("I", len(self.musiclist)) # Length of music list
  buf += struct.pack("I", len(self.zonelist)) # Length of zone list
  buf += packing.packString16(self.motd) # Message Of The Day displayed upon finishing connection
  sendBuffer(buf)
  ```

* AIOprotocol.REQUEST:
  * Sent by client after AIOprotocol.CONNECT to get character, music, zone and evidence list, in that order
  * Server replies adequately.
  * Request values: 0=characters, 1=music, 2=zones, 3=evidence list
  ```python
  # Client:
  buf = struct.pack("B", AIOprotocol.REQUEST)
  buf += struct.pack("B", 0) # request value. we start by asking character list
  sendBuffer(buf)


  # Server:
  data, requestType = packing.buffer_read("B", data)
  # Constructing the reply
  buf = struct.pack("B", AIOprotocol.REQUEST)
  buf += struct.pack("B", requestType)
  if requestType == 0: # characters
      for char in self.charlist:
	      buf += packing.packString16(char)

  elif requestType == 1: # music
      for song in self.musiclist:
          buf += packing.packString16(song)

  elif requestType == 2: # zones
      for zone in self.zonelist:
	      buf += packing.packString8(zone[0]) # Zone filename
          buf += packing.packString16(zone[1]) # Zone display name
      buf += struct.pack("H", self.defaultzone) # default zone ID (uint16)

  elif requestType == 3: # evidence on current zone
      buf += struct.pack("B", len(self.evidencelist[self.defaultzone]))
      for evidence in self.evidencelist[self.defaultzone]:
          buf += packing.packString8(evidence[0]) # name
          buf += packing.packString16(evidence[1]) # description
          buf += packing.packString8(evidence[2]) # image filename

  sendBuffer(buf) # Send the reply to the client
  
  
  # On the receiving client, use buffer_read and unpackString functions to read the message in the same order.
  # The requestType value will tell you what it contains.
  # Repeat this for requestTypes 1, 2, 3 to get music, zone and evidence.
  # After this, the server considers the client "ready", and can now send game messages.
  ```

* AIOprotocol.CREATE:
  * Sent from server to all clients to tell them to create a new player object on their end.
  ```python
  # Server:
  buf = struct.pack("B", AIOprotocol.CREATE)
  buf += struct.pack("I", clientID) # uint32
  buf += struct.pack("h", charID) # int16. can be -1 to indicate in character select screen.
  buf += struct.pack("H", zoneID) # uint16
  sendBuffer(buf)

  # Client:
  data, clientID = packing.buffer_read("I", data)
  data, charID = packing.buffer_read("h", data)
  data, zoneID = packing.buffer_read("H", data)
  # Handle player object creation...
  ```

* AIOprotocol.DESTROY:
  * Sent from server to all clients to tell them to delete an existing player object.
  ```python
  # Server:
  buf = struct.pack("B", AIOprotocol.DESTROY)
  buf += struct.pack("I", clientID) # uint32
  sendBuffer(buf)
  
  # Client:
  data, clientID = packing.buffer_read("I", data)
  # Handle player object deletion...
  ```

* AIOprotocol.SETCHAR:
  * Sent from server to all clients to tell them a player has changed character.
  * Client can also send this to server to request a change to a different character.
  ```python
  # Sending from either client or server:
  buf = struct.pack("B", AIOprotocol.SETCHAR)
  buf += struct.pack("H", clientID) # server only
  buf += struct.pack("h", charID) # can be -1 to indicate in character select screen.
  sendBuffer(buf)
  
  # Receiving:
  data, clientID = packing.buffer_read("H", data) # receiving on client only
  data, charID = packing.buffer_read("h", data)
  # Handle character change. if clientID is yours and charID is -1, boot yourself to the character select screen.
  ```

* AIOprotocol.SETZONE:
  * Sent from server to all clients to tell them a player has moved to another zone.
  * Client can also send this to server to request a change to a different zone.
  ```python
  # Sending from either client or server:
  buf = struct.pack("B", AIOprotocol.SETZONE)
  buf += struct.pack("H", clientID) # server only
  buf += struct.pack("H", zoneID)
  sendBuffer(buf)
  
  # Receiving:
  data, clientID = packing.buffer_read("H", data) # receiving on client only
  data, zoneID = packing.buffer_read("H", data)
  # Handle zone change. if clientID is yours, move yourself to that zone.
  ```

* AIOprotocol.MOVE:
  * This packet is sent periodically. (10 ticks on client, 6 ticks on server, although they can be altered. a tick is 1./60 of a sec)
  * Client sends this to server to update their position ingame.
  * Server sends this to all clients to update positions.
  ```python
  # Client:
  buf = struct.pack("B", AIOprotocol.MOVE)
  buf += struct.pack("f", x) # float
  buf += struct.pack("f", y) # float
  buf += struct.pack("h", hspeed)
  buf += struct.pack("h", vspeed)
  buf += packing.packString16(sprite) # "charname/filename.gif". filename does not include "charprefix" variable (e.g. "lilmiles") if any.
  buf += struct.pack("B", emoting) # emote status: 0=not using emote, 1=yes
  buf += struct.pack("B", dir_nr) # directions from 0 (south) to 7 (southeast). see AIOprotocol.py
  buf += struct.pack("B", currentemote+1) # if 0, no emote is playing, else corresponds to emoteID on char.ini Emotions
  sendBuffer(buf)

  # Server:
  data, x = packing.buffer_read("f", data)
  data, y = packing.buffer_read("f", data)
  data, hspeed = packing.buffer_read("h", data)
  data, vspeed = packing.buffer_read("h", data)
  data, sprite = packing.unpackString16(data)
  data, emoting = packing.buffer_read("B", data)
  data, dir_nr = packing.buffer_read("B", data)
  data, currentemote = packing.buffer_read("B", data)
  currentemote -= 1 # GET REAL number. if -1 = no emote

  # Server sending everyone's positions to one client:
  buf = struct.pack("B", AIOprotocol.MOVE)
  # We will need to use a tempbuf variable and add positions for all ACTIVE and READY clients.
  total = 0 # count up for every client position added to tempbuf
  tempbuf = ""
  for clientLoop in self.clients.keys():
      if not self.clients[clientLoop].ready or clientLoop == clientID: # do not count this client.
          continue

      total += 1
      tempbuf += struct.pack("I", clientLoop)
      tempbuf += struct.pack("f", self.clients[client].x)
      tempbuf += struct.pack("f", self.clients[client].y)
      tempbuf += struct.pack("h", self.clients[client].hspeed)
      tempbuf += struct.pack("h", self.clients[client].vspeed)
      tempbuf += packing.packString16(self.clients[client].sprite)
      tempbuf += struct.pack("B", self.clients[client].emoting)
      tempbuf += struct.pack("B", self.clients[client].dir_nr)
      tempbuf += struct.pack("B", self.clients[client].currentemote+1) # although you could just skip "currentemote -= 1" above and don't do +1 here if you want...

  if total > 0: # don't send anything if it's empty
      # After looping through all possible clients, we get the final buf:
      buf += struct.pack("H", total) # total number of clients moved.
      buf += tempbuf
      sendBuffer(buf)

  # are you still with us?
  ```

* AIOprotocol.MSCHAT:
  * Client sends this to server to send a chat message on IC (In-Character) chat.
  * Server sends it to everyone else to see.
  * Unlike AO there's not much here, so you can rest easy.
  ```python
  # Client:
  buf = struct.pack("B", AIOprotocol.MSCHAT)
  buf += packing.packString8(chatmsg)
  buf += packing.packString8(blip) # sfx-blip gender
  buf += struct.pack("I", color) # uint32 color code hex. e.g. 0xffffffff is white
  buf += struct.pack("B", realization) # realization/lightbulb sound
  buf += struct.pack("B", evidence) # presented evidence, 0 if none
  buf += packing.packString8(showname)
  sendBuffer(buf)
  
  # Server:
  data, chatmsg = packing.unpackString8(data)
  data, blip = packing.unpackString8(data)
  data, color = packing.buffer_read("I", data)
  data, realization = packing.buffer_read("B", data)
  data, showname = packing.unpackString8(data)
  # Server sending to clients:
  buf = struct.pack("B", AIOprotocol.MSCHAT)
  buf += packing.packString8(name) # variable "name" is either character name or showname
  buf += packing.packString8(chatmsg)
  buf += packing.packString8(blip)
  buf += struct.pack("I", zone)
  buf += struct.pack("I", color)
  buf += struct.pack("B", realization)
  buf += struct.pack("I", clientID)
  buf += struct.pack("B", evidence)
  sendBuffer(buf) # to all clients
  ```

* AIOprotocol.MUSIC:
  * Client sends to request music change.
  * Server sends music change to all clients.
  ```python
  # Client:
  buf = struct.pack("B", AIOprotocol.MUSIC)
  buf += packing.packString16(filename)
  buf += packing.packString8(showname)
  sendBuffer(buf)

  # Server:
  data, filename = packing.unpackString16(data)
  data, showname = packing.unpackString16(data)
  # Do a check to make sure filename exists on the server's music list.
  # Assuming we're good, we send the change.
  buf = struct.pack("B", AIOprotocol.MUSIC)
  buf += packing.packString16(filename)
  buf += struct.pack("I", charID)
  buf += packing.packString8(showname)
  sendBuffer(buf)
  ```

* AIOprotocol.OOC:
  * Client sends this to server to send an OOC (Out-of-Character) chat message.
  * Server sends it to everyone to see, or parses a slash-command if any.
  ```python
  # Client:
  buf = struct.pack("B", AIOprotocol.OOC)
  buf += packing.packString8(name)
  buf += packing.packString16(chatmsg)
  sendBuffer(buf)

  # Server:
  data, name = packing.unpackString8(data)
  data, chatmsg = packing.unpackString16(data)
  # Check the name if it isn't empty, and
  # execute a slash-command if any... if none, send the message to everyone else,
  # creating the buf variable the same way as the client.
  ```

* AIOprotocol.EMOTESOUND:
  * Sent by client when playing an emote that contains a sound.
  * Server sends it to everyone else.
  ```python
  # Client:
  buf = struct.pack("B", AIOprotocol.EMOTESOUND)
  buf += packing.packString8(soundname) # filename.
  buf += struct.pack("I", delay) # in milliseconds.
  sendBuffer(buf)
  
  # Server:
  data, soundname = packing.unpackString8(data)
  data, delay = packing.buffer_read("I", data)
  # Sending to all clients:
  buf = struct.pack("B", AIOprotocol.EMOTESOUND)
  buf += struct.pack("I", clientID)
  buf += packing.packString8(soundname)
  buf += struct.pack("I", delay)
  sendBuffer(buf)
  ```

* AIOprotocol.BROADCAST:
  * Sent by server to all clients on a zone or all zones which displays a broadcast message on the game viewport.
  ```python
  # Server:
  buf = struct.pack("B", AIOprotocol.BROADCAST)
  buf += packing.packString16(message)
  sendBuffer(buf)

  # Client:
  data, message = packing.unpackString16(data)
  # Handle showing message on viewport...
  ```

* AIOprotocol.CHATBUBBLE:
  * Displays a chat bubble above a player to indicate typing status. 0=off, 1=typing, 2=message ticking on chatbox
  ```python
  # Client:
  buf = struct.pack("B", AIOprotocol.CHATBUBBLE)
  buf += struct.pack("B", chatBubbleValue) # 0, 1 or 2
  sendBuffer(buf)

  # Server:
  data, chatBubbleValue = packing.buffer_read("B", data)
  # Sending to all clients:
  buf = struct.pack("B", AIOprotocol.CHATBUBBLE)
  buf += struct.pack("I", clientID)
  buf += struct.pack("B", chatBubbleValue)
  sendBuffer(buf)
  ```

* AIOprotocol.EXAMINE:
  * Displays an Ace Attorney-like crosshair on the game viewport to simulate the "Examine" mechanic.
  ```python
  # Client:
  buf = struct.pack("B", AIOprotocol.EXAMINE)
  buf += struct.pack("f", x)
  buf += struct.pack("f", y)
  buf += packing.packString8(showname)
  sendBuffer(buf)

  # Server:
  data, x = packing.buffer_read("f", data)
  data, y = packing.buffer_read("f", data)
  data, showname = packing.unpackString8(data)
  # Send to all clients:
  buf = struct.pack("B", AIOprotocol.EXAMINE)
  buf += struct.pack("H", charID)
  buf += struct.pack("H", zone)
  buf += struct.pack("f", x)
  buf += struct.pack("f", y)
  buf += packing.packString8(showname)
  sendBuffer(buf)
  ```

* AIOprotocol.BARS:
  * Alters blue/red penalty bars.
  ```python
  # Client:
  buf = struct.pack("B", AIOprotocol.BARS)
  buf += struct.pack("B", bar) # 0=blue, 1=red
  buf += struct.pack("B", health) # min 0, max 10
  sendBuffer(buf)

  # Server:
  data, bar = packing.buffer_read("B", data)
  data, health = packing.buffer_read("B", data)
  # Do some checks to make sure bar is either 0 or 1, and health is in the range of 0-10.
  # If we're all good, set the bar health and sync it to all clients
  # the same way the client made the buf with struct.pack().
  ```

* AIOprotocol.WTCE:
  * Displays a Witness Testimony, judge verdict or AAI Argument/Rebuttal overlay on the game viewport.
  ```python
  # Client:
  buf = struct.pack("B", AIOprotocol.WTCE)
  buf += struct.pack("B", wtcetype) # 0=testimony, 1=crossexamination, 2=notguilty, 3=guilty, 4=argument, 5=rebuttal
  sendBuffer(buf)

  # Server:
  data, wtcetype = packing.buffer_read("B", data)
  # Send it to all clients same way the client constructed the buf.
  ```

* AIOprotocol.EVIDENCE:
  * Provides evidence management on a zone: add, edit and delete.
  ```python
  # Client:
  # pretend you have "*args"
  buf = struct.pack("B", AIOprotocol.EVIDENCE)
  buf += struct.pack("B", type) # see AIOprotocol.EV_* for all values.
  if type == AIOprotocol.EV_ADD:
      name, desc, image = args
      buf += packing.packString8(name)
      buf += packing.packString16(desc)
      buf += packing.packString8(image)
  elif type == AIOprotocol.EV_EDIT:
      ind, name, desc, image = args
      buf += struct.pack("B", ind) # index of evidence being edited
      buf += packing.packString8(name)
      buf += packing.packString16(desc)
      buf += packing.packString8(image)
  elif type == AIOprotocol.EV_DELETE:
      ind, = args
      buf += struct.pack("B", ind) # index of evidence being deleted
  sendBuffer(buf)

  # Server:
  # Just use packing.buffer_read and unpackString8/unpackString16 where necessary.
  # If adding, do a check to make sure the evidence list on a zone doesn't exceed 255. Send a warning if it does and abort.
  # If editing or deleting, do a check to make sure ind exists in evidence list. Send a warning if it doesn't and abort.
  # When sending new evidence to clients, there is a minor change:
  buf = struct.pack("B", AIOprotocol.EVIDENCE)
  buf += struct.pack("B", type) # see AIOprotocol.EV_* for all values.
  buf += struct.pack("B", zone) # zone where evidence was edited.
  # Add to buf the added, edited or deleted evidence as explained above in client and do sendBuffer(buf).
  ```
  * Server can also send AIOprotocol.EV_FULL_LIST as a value:
  ```python
  # Server:
  buf = struct.pack("B", AIOprotocol.EVIDENCE)
  buf += struct.pack("B", AIOprotocol.EV_FULL_LIST)
  buf += struct.pack("B", zone)
  buf += struct.pack("B", len(self.evidencelist[zone]))
  for evidence in self.evidencelist[zone]:
      buf += packing.packString8(evidence[0]) # name
      buf += packing.packString16(evidence[1]) # description
      buf += packing.packString8(evidence[2]) # image filename
  sendBuffer(buf)
  ```

* AIOprotocol.WARN:
  * Sent to a client to display a warning message. Can be triggered with OOC /warn command.
  ```python
  # Server:
  buf = struct.pack("B", AIOprotocol.WARN)
  buf += packing.packString16(message)
  sendBuffer(buf)

  # Client:
  data, message = packing.unpackString16(data)
  # Handle displaying message clientside...
  ```

* AIOprotocol.KICK:
  * Sent to a client to notify them they have been kicked from the server with a specified reason.
  ```python
  # Server:
  buf = struct.pack("B", AIOprotocol.KICK)
  buf += packing.packString16(reason)
  sendBuffer(buf)
  if self.clients[clientID].ready: # if this player was ingame at the time of the kick,
    sendDestroy(clientID) # destroy their instance from everyone else's view with AIOprotocol.DESTROY as explained above,
  del self.clients[ClientID] # and delete it from clients dict so that their traffic is ignored. closing the socket might make them not receive the kick reason

  # Client:
  data, reason = packing.unpackString16(data)
  # display reason message and close socket. server can't read your traffic anyway
  ```

* AIOprotocol.PING:
  * Keepalive message sent by client to tell the server it is still connected.
  * Maximum time without sending a ping message is 60 seconds. (can be altered)
  ```python
  # Client and server:
  buf = struct.pack("B", AIOprotocol.PING) # pong
  sendBuffer(buf)

  # Server will reply to client with the same message as confirmation.
  # You can also use this to calculate and display ingame ping in ms!
  ```


## Network protocol (UDP)
Right now, UDP is only used to get the server info on the server list, such as name, description, players online, version and ping.

### Packet structure
```python
import zlib, packing
# Sending on client:
def sendBuffer(buf, ip, port):
    udp.sendto(buf, (ip, port))

# Receiving on server:
data, addr = udp.recvfrom(65535) # could be implemented better...

# Sending on server:
def sendBuffer(buf, ip, port):
    udp.sendto(zlib.compress(buf), (ip, port))

# Receiving on client:
data, addr = udp.recvfrom(65535) # could be implemented better...
data = zlib.decompress(data)

# The header is read by using only this:
data, header = packing.buffer_read("B", data)
```

### Network messages
* AIOprotocol.UDP_REQUEST:
  * Client sends this to server to request info about name, desc, players and version (ping is calculated clientside)
  * Server replies adequately
  ```python
  # Client:
  data = struct.pack("B", AIOprotocol.UDP_REQUEST)
  sendBuffer(data, ip, port)

  # Server:
  response = struct.pack("B", header)
  response += packString16(self.servername)
  response += packString16(self.serverdesc)
  response += struct.pack("IIH", len(self.clients), self.maxplayers, packing.versionToInt(GameVersion)) # three in one, how's that?
  sendBuffer(response, ip, port)
  # client can convert version using packing.versionToStr(). see packing.py for more on this
  ```

Hope you're not dead yet after sifting through all this!
<font size="0.1"><sup>looking at you salanto</sup></font>

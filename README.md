
Input Director UDP Protocol Description
=======================================
v1.07


Introduction
------------

This document describes version 8 of the Input Director (ID) UDP protocol. This
protocol version is used by v1.2.2 of Input Director.
All information in this document has been obtained by using [Wireshark][] and
other network tools. No reverse engineering of the ID binaries or libraries 
was performed (or needed).

The TCP protocol used for the *Share Clipboard* feature of ID is not described
in this document.

See [uix5/packet-inpdir-udp][] for a Wireshark dissector for the protocol
described here.




Outline of This Document
------------------------

 * Introduction
 * Outline
 * General Protocol Information
 * List of Packet Types
 * ID Packet Formats
 * TODO



General protocol information
----------------------------

ID uses UDP packets, normally on port 31234 (configurable on the 
*global preferences* tab). Each datagram contains only one packet, even if
payloads are smaller than the maximum UDP payload allowed.

Byte order is almost exclusively Little Endian, with the exception of the 
various IP address fields.

Most traffic in ID networks flows from the master to the slaves, with 
slaves occasionally notifying the master about their status. Such updates 
include: cursor needs to transition away from slave, slave is going into 
standby, reboots or ID is being shutdown or disabled. Masters send their
slaves configuration updates, information on which screen currently has input
focus, keyboard / mouse input events and commands such as lock workstation, 
shutdown or (de)activate the screensaver.

TODO: add info on the GUID in the packets




List of Packet Types
--------------------

The list of observed packet types and their names, as described in this 
document.

    | ID   | Name
    +------+-------------------
    | 0x01 | Unknown
    | 0x02 | Input Event
    | 0x03 | Cursor Enter
    | 0x04 | Cursor Enter ACK
    | 0x05 | Slave Config Request
    | 0x06 | Slave Config Reply
    | 0x07 | Cursor Exit
    | 0x08 | Unknown
    | 0x09 | Shortcut Cursor Return
    | 0x0A | Slave Clipboard Status
    | 0x0B | Input Mirror
    | 0x0C | Unknown
    | 0x0D | Unknown
    | 0x0E | Unknown
    | 0x0F | Screensaver State
    | 0x10 | Slave Lock
    | 0x11 | Slave Shutdown
    | 0x12 | Master Heartbeat
    | 0x13 | Slave Ctrl+Alt+Delete
    | 0x14 | Configuration Update
    | 0x15 | Unknown
    | 0x16 | Disable Edge Transitions
    | 0x17 | Session Termination
    | 0x18 | Unknown
    | 0x19 | Encryption Config Mismatch
    | 0x1A | Session Setup
    | 0x1B | Session Setup ACK
    | 0x1C | Slave Announce
    | 0x1D | Slave Heartbeat




ID Packet Formats
-----------------

This section describes the various packets sent and received by ID during 
normal program usage. They are listed in order of their packet id. Their names
(and those of the fields they contain) are purely derived from their content 
and / or perceived purpose and are most certainly not their 'official' names.

All fields are Little Endian, unless indicated otherwise.



### Notation ###

    | Symbol | Short for:    | Description
    +--------+---------------+------------
    | D      | Default value | Most frequently observed value for this field
    | BE     | Big Endian    | Byte order specification
    | N,C,.. |               | Pseudo variable name, used in calculations
    | Xz     | Size          | Calculated size of a particular field
    | ?      | Unsure        | Probably correct, but still unsure

Overview of the use of symbols in this document.



### Common part ###

    | Size | Name / Description
    +------+-------------------
    |   12 | Magic
    |    4 | Header Length?
    |    4 | Protocol Version
    |    4 | Packet Type
    |    4 | Packet Length
    |    4 | Sequence Number
    |   16 | Session Key
    |    4 | Source Address (BE)
    |    4 | Source Listening Port
    |    4 | Unknown0
    |    1 | Encryption Type
    |    1 | Encryption Password Test
    |    2 | Unknown1

Each packet contains a common part of 64 bytes:

`Magic`: 0x17220100, 0x038f3bd4 and again 0x17220100. 

The `Header Length` field seems to indicate the size of a common part shared by 
all packets.

`Packet Length` is the total length (in bytes) of the ID packet, including 
`Header Length`.

The `Session Key` is a 16 byte GUID that is set by the master at session setup 
and is used by the slave to be able to tell masters apart (example: 
`e2b6f7ed-d670-e041-a0df-2f810a05f1fa`). Packets with unknown keys (those that 
have not been presented in a *Session Setup* exchange) are considered 
unauthenticated and ignored.

The `Unknown` field seems a repeat of `Header Length`.

`Encryption Type` contains the type of encryption used in a packet. Observed values are:

 * 0x00: No encryption
 * 0x01: AES 128 bit key
 * 0x02: AES 192 bit key
 * 0x03: AES 256 bit key

`Encryption Password Test` seems to be dependent on the password entered 
in the *Set Security* dialog. Entering the same password always results in the 
same value for this byte, independent of the encryption key length set.



### 0x01 - Unknown ###

Unknown. Never seen.



### 0x02 - Input Event (M → S) ###

    | Size | Name / Description
    +------+-------------------
    |   64 | Common part
    |    4 | Number of Input Events (N)
    |    4 | Unknown1 (D:1)
    |    4 | Slave Number
    |    2 | Master KB Layout
    |    2 | Flags
    |   16 | Unknown3
    |    4 | Unknown4
    |    4 | Unknown5
    |    4 | Unknown6
    |    4 | Cursor Origin?
    |    4 | Clipboard format list len?
    |    4 | IP/Mask?
    |    4 | Port?
    |   20 | Mouse Settings
    |    4 | Cursor Transition Settings
    |   16 | Unknown7
    |   sz | Input events (sz = 24 * N)

Used to send mouse and keyboard data to the slave that is currently being 
directed.

Packets containing mouse input always seem to contain only one struct. Multiple 
key presses can occur in one packet, but only if they combine into an ID 
configured shortcut (fi: switching slaves via keyboard). This is then clear
from the `Number of Input Events` field.

The `Slave Number` field contains a unique integer which allows ID to determine
from which slave a *Cursor Exit* packet originated. It is unclear why the 
`Source Address` field from the *Common Part* isn't used for this or why this 
number is sent with every *Input Event*. Each number seems to correspond to 
an index on an ID internal list of known slaves and changes whenever the slaves
are changed on the *Master Configuration* tab.

`Master KB Layout`: masters send their currently configured keyboard layout with 
each *Input Event* in case that option is enabled on the *Master Preferences* 
tab of ID. The number is the language identifier of the keyboard layout. See
[Language Identifier Constants and Strings][] on MSDN for more information.

Observed values for `Flags`:

 * 0x00: slave uses its own KB layout 
 * 0x02: slave uses master's KB layout

The `Cursor Origin` and `Clipboard format list len` fields could be 'ghosts' of 
those fields in *Cursor Enter* packets, which are not properly cleared. See 
*0x03 - Cursor Enter* for details.

The function of the `IP/Mask` and `Port` fields is unknown. No other values
than 0xFFFFFFFF and 0x7A02 (31234) have been observed.

The `Mouse Settings` field could be a 'ghost' of the same field in *Cursor Enter*
packets, which is not properly cleared. See *0x03 - Cursor Enter* for details.

`Unknown7` is mostly 0, except for when encryption is used. Unclear whether it
is then part of the encrypted payload, or contains some sort of key.

`Input Events` are Win32 [INPUT structure][] structs. See [KEYBDINPUT structure][] 
and [MOUSEINPUT structure][] on the MSDN for more information. As 
`sizeof(KEYBDINPUT)` != `sizeof(MOUSEINPUT)`, there seem to be 8 bytes of 
garbage after each KEYBDINPUT. The `time` and `dwExtraInfo` fields appear not 
to be used for either type, and are always equal to 0.



### 0x03 - Cursor Enter (M → S) ###

    | Size | Name / Description
    +------+-------------------
    |   64 | Common part
    |    4 | Slave Number
    |    4 | Unknown1 (D:0)
    |    4 | Neighbours
    |   20 | Unknown2
    |    4 | Unknown3 (D:6)
    |    4 | Unknown4 (D:0xC0565000)
    |    4 | Enter Abs Coord
    |    4 | Enter Side
    |    4 | Clipboard format list len? (N)
    |    4 | IP/Mask?
    |    4 | Port?
    |   20 | Mouse Settings
    |    4 | Cursor Transition Settings
    |   16 | Unknown5
    |   45 | Slave Screen Setup
    |   sz | Clipboard format list (sz = 4 * N)
    |   ?? | Zeros / Garbage

Sent by a master when the cursor transitions to a slave (using a shortcut or
via mouse movement), either from the master itself, or from another slave. 
Should be followed by a *Cursor Enter ACK* within a configurable time-out 
(under *Advanced* on the *Master Preferences* tab), otherwise the slave is 
considered to be unresponsive and ID will ask the user if that slave should 
be skipped.

The `Slave Number` field is identical to the one in the *Input Event* packet.
It seems to be used here to be able to identify later from which slave a cursor 
transition originated.

The `Neighbours` field contains a bitmask indicating at which edges of its
screen a slave has other slaves. Mask values observed:

 * 0x01: Left
 * 0x02: Right
 * 0x04: Top
 * 0x08: Bottom

These are bitwise OR-ed in case there are multiple neighbours for one slave.

`Unknown2` is almost always 0. If not, values appear to be random?

`Unknown3` has only been observed to be equal to 6. Function unknown.

`Unknown4` is either 0 or 0xC0565000. Function unknown.

The `Enter Abs Coord` is the absolute mouse coordinate at the edge where the 
cursor entered the screen.
ID sends only the coordinate on the axis that is parallel to the transitioned 
edge. So for a transition across the `Left` edge of a screen, the `y` 
coordinate is sent, for a transition across `Top`, the `x` coordinate is sent.

Absolute mouse coordinates can be calculated using the formulae: 

    x_abs = (x / screenWidth) * 65536 
    y_abs = (y / screenHeight) * 65536

The inverse should be used to calculate the correct coordinates for the screen
resolutions of the slaves. This allows the mouse cursor to appear at the proper 
location (see also [MOUSEINPUT structure][]).

`Enter Side` indicates which screen edge was transitioned by the cursor. Values
are identical to those in the `Neighbours` field, with one addition:

 * 0x10: No side (transition initiated through a hotkey)

The function of the `IP/Mask` and `Port` fields is unknown. No other values
than 0xFFFFFFFF and 0x7A02 (31234) have been observed.

The `Mouse Settings` field contains the values of the mouse preferences of
the *master* in case ID is configured to use the master's preferences when 
directing slaves. Otherwise these fields are all 0, except for 
`Master Mouse Prefs`.

    | Size | Name / Description
    +------+-------------------
    |    4 | Master Mouse Prefs (D:0x08)
    |    4 | MouseThreshold1
    |    4 | MouseThreshold2
    |    4 | MouseSpeed
    |    4 | MouseSensitivity + Flags

These values are read from the registry of the master. The `Master Mouse Prefs` 
seems to be a bitmask, with the following values observed:

 * 0x02: Use master mouse preferences on slave
 * 0x08: Unknown (use slave mouse preferences?)

In case the MSB of the `MouseSensitivity` field is high, the primary and
secondary mouse buttons have been switched (using the setting on the *Buttons* 
tab of the Mouse Control Panel applet).

The `Cursor Transition Settings` field represents the settings under 
*Transition at screen-edge* on the *Master Preferences* tab.

    | Size | Name / Description
    +------+-------------------
    |    1 | Transition Flags
    |    1 | Transitions Near Corners
    |    2 | Transition Delay

First `Flags` field indicates when a transition should occur:

 * 0x00: Immediately
 * 0x01: On double tap of edge
 * 0x02: On cursor lingering

The `Transitions Near Corners` field indicates whether transitions at 
*monitor's corners* are allowed:

 * 0x00: allowed
 * 0x01: not allowed

The value of the `Transition Delay` field is only significant if 
`Transition Flags` is not 'Immediately'. It specifies the delay before 
transitioning in ms.

`Unknown5` is almost always 0. In case it is not, values appear to be random.
Its function is unknown.

The `Slave Screen Setup` field contains a representation of what the *master*
believes is the screen setup of the slave. This includes the number of monitors
and their physical location relative to each other. ID transmits this 
information (or grid) to the slave so it can check the validity.

    | Size | Name / Description
    +------+-------------------
    |    1 | Unknown0
    |    1 | Unknown1
    |    1 | Grid Columns (C)
    |    1 | Grid Rows (R)
    |    1 | Use Windows Monitor IDs
    |   sz | Cells (sz = C x R x 2)
    |   gz | Garbage / Zeros (gz = 40 - sz)

`Unknown0` and `Unknown1` seem to be related to the position of the 'main' 
screen of the slave with respect to the master, but their exact function is
currently unknown.

The `Use Windows Monitor IDs` field corresponds to the setting on the 
*Slave Configuration* dialog (via the *Master Configuration* tab):

 * 0x00: Let ID figure it out
 * 0x01: Use Windows monitor IDs

The `Cells` field contains a column-major ordered array of two-byte entries, 
one for each cell.

    | Size | Name / Description
    +------+-------------------
    |    1 | Occupied
    |    1 | Monitor ID

The `Occupied` flag indicates whether the cell contains a screen. If Windows
monitor IDs are used, the `Monitor ID` field contains it, otherwise it is equal 
to 0xFF.

Maximum number of configurable screens in ID 1.2.2 is 9, resulting in a maximum 
size grid of 5x4. At 2 bytes per cell, this gives (5x4)x2 = 40 bytes for the
cell array.


The `Clipboard format list` contains `Clipboard format list len` entries, one 
for each of the formats in which the information on the clipboard of the 
*master* can be represented. This is used in the synchronisation of the 
clipboards in case that feature is enabled. See [Standard Clipboard Formats][] 
on the MSDN for more information on the entries. There does not seem to be a
maximum to the number of entries (packet grows with each extra format).



### 0x04 - Cursor Enter ACK (S → M) ###

    | Size | Name / Description
    +------+-------------------
    |   64 | Common part
    |    4 | ACK on Sequence Number
    |    4 | Lock Flags
    |    4 | Unknown0
    |    4 | Unknown1
    |   84 | Zeros / Garbage

Sent by a slave upon receiving a *Cursor Enter* packet from a known master.

`Lock Flags` contains a bitmask representing the state of the Scroll, Num and
Caps Lock keys of the slave. Mask values:

 * 0x01: Num Lock
 * 0x02: Scroll Lock
 * 0x04: Caps Lock

Depending on the configuration of the master (*Mouse / Keyboard Preferences*
on the *Master Preferences* tab), the state of those keys on the master is
updated to reflect those of the slave.



### 0x05 - Slave Config Request (M → S) ###

    | Size | Name / Description
    +------+-------------------
    |   64 | Common part
    |  100 | Zeros / Garbage

Sent by a master as part of the session setup exchange after it has received
a *Session Setup ACK* from a prospective slave.



### 0x06 - Slave Config Reply (S → M) ###

    | Size | Name / Description
    +------+-------------------
    |   64 | Common part
    |    4 | ACK on Sequence Number
    |    4 | Screen Resolution Width
    |    4 | Screen Resolution Height
    |    4 | Flags?
    |   84 | Zeros / Garbage

Sent by a slave upon receiving a *Slave Config Request* in a session setup
exchange. Contains information on the screen resolution of the slave.

`ACK on Sequence Number` contains the `Sequence Number` of the 
*Slave Config Request* packet that was received from the master.

`Flags` is not always 0, but could be a 'ghost' of a field from another packet
that is not properly cleared.



### 0x07 - Cursor Exit (S → M) ###

    | Size | Name / Description
    +------+-------------------
    |   64 | Common part
    |    4 | Slave Number
    |    4 | Unknown1
    |    4 | Exit Abs Coord
    |    4 | Exit Side
    |   84 | Zeros / Garbage

Sent by a slave upon switching input focus to the master or another slave  
by mouse movement (as opposed to via a hotkey, see 
*0x09 - Shortcut Cursor Return*).

The `Slave Number` field is used by the master to identify from which slave 
the transition originated. See *0x03 - Cursor Enter* for more information.

For a description of the `Exit Abs Coord` and `Exit Side` fields, see their
complementary fields in *0x03 - Cursor Enter*.



### 0x08 - Unknown ###

Unknown. Never seen.



### 0x09 - Shortcut Cursor Return (M → S) ###

    | Size | Name / Description
    +------+-------------------
    |   64 | Common part
    |    4 | Shortcut Key Count?
    |    4 | Unknown1
    |    4 | Slave Number
    |   88 | Zeros / Garbage

Sent by a master upon switching input focus to another slave or to the master 
by pressing the appropriate hotkey(s).

The `Shortcut Key Count` field seems to represent the number of keys that make 
up the particular shortcut used. For instance, if the shortcut to go to the 
screen to the right of the current screen is `Ctrl + Alt + Right Arrow`, this 
number would be 3.

Unsure why the `Slave Number` field is sent by the master. See 
*0x02 - Input Event* for more information on this field.



### 0x0A - Slave Clipboard Status (S → M) ###

    | Size | Name / Description
    +------+-------------------
    |   64 | Common part
    |    4 | Clipboard format list len? (N)
    |    4 | IP/Mask?
    |    4 | Port?
    |   88 | Unknown
    |   sz | Clipboard format list (sz = 4 * N)
    |   ?? | Zeros / Garbage

Sent by a slave upon *Cursor Return* to its master (either by shortcut or 
through mouse movement) in case the slave's clipboard is non-empty and 
the *Share Clipboard* feature is enabled.

Its structure is identical to the one used in *0x03 - Cursor Enter*.



### 0x0B - Input Mirror (M → S) ###

Same as *0x02 - Input Event*, but input focus remains at the master and all
attached slaves receive / process the event.



### 0x0C to 0x0E - Unknown ###

Unknown. Never seen.



### 0x0F - Screensaver State (M → S) ###

    | Size | Name / Description
    +------+-------------------
    |   64 | Common part
    |    4 | State
    |   96 | Zeros / Garbage

Sent by master on activation and deactivation of the screensaver.

Observed values for `State`:

 * 0x00: deactivate screensaver
 * 0x01: activate screensaver



### 0x10 - Slave Lock (M → S) ###

    | Size | Name / Description
    +------+-------------------
    |   64 | Common part
    |  100 | Zeros / Garbage

Sent by master upon locking the master or when selecting the 
*Lock Slaves and Master* menu option.



### 0x11 - Slave Shutdown (M → S) ###

    | Size | Name / Description
    +------+-------------------
    |   64 | Common part
    |  100 | Zeros / Garbage

Sent by master when selecting the *Shutdown Slaves and Master* menu option, or
when pressing the *Shutdown Slave Workstations* button.



### 0x12 - Master Heartbeat (M → S) ###

    | Size | Name / Description
    +------+-------------------
    |   64 | Common part
    |  100 | Zeros / Garbage

Sent by master every 120 seconds of inactivity.



### 0x13 - Slave Ctrl+Alt+Delete (M → S) ###

    | Size | Name / Description
    +------+-------------------
    |   64 | Common part
    |  100 | Zeros / Garbage

Sent by a master upon pressing the *Ctrl+Alt+Delete equivalent for slave systems*
hotkey (*Master Preferences* tab).



### 0x14 - Config Update (M → S) ###

    | Size | Name / Description
    +------+-------------------
    |   64 | Common part
    |    4 | Config Valid
    |    4 | Focus Hostname strlen (N)
    |    4 | Focus IP (BE)
    |    2 | Focus Screen X
    |    2 | Focus Screen Y
    |   84 | Unknown0
    |   45 | Slave Screen Setup
    |   sz | Focus Hostname (sz = N)

Sent by a master to inform all slaves of configuration changes: 

 * adding to or editing a slave on the master configuration (screens, layout, 
   hotkeys, etc)
 * changes to the screen layout on the *Master Configuration* tab
 * changes to the master's number of screens and / or their layout

This packet is also sent directly after a *Cursor Enter ACK* is received by a
master, to inform all slaves about the slave that has input focus. This is
probably used for updating the *Information Window* (*Global Preferences* tab),
which displays the hostname and an arrow pointing towards the active screen.

The `Config Valid` field seems to indicate whether the last *Config Update* a 
slave received is still valid. Packets with a 0 here require a slave to reset
their state (and possibly wait for a new update).

The coordinate system used by ID has its x-axis pointing left, and its y-axis 
pointing up, with the master at its origin:

     1, 1 | 0, 1 | -1, 1
    ---------------------
     1, 0 |  M   | -1, 0
    ---------------------
     1,-1 | 0,-1 | -1,-1

The `Focus Screen X` and `Focus Screen Y` fields seem to be used when drawing
the arrow in the *Information Window*.

The `Slave Screen Setup` field is identical to that in the *Cursor Enter* 
packet. See *0x03 - Cursor Enter* for details. The two unknown fields in this
field appear to have some relation to the `Screen X/Y` fields in this packet,
but their exact purpose is not known.

The `Focus Hostname` contains the hostname of the slave that has input focus
as a null-terminated string, with length equal to `Focus Hostname strlen`. In
case the hostname is unknown, the corresponding IP address (as a string) is 
placed here.



### 0x15 - Unknown ###

Unknown. Never seen.



### 0x16 - Disable Edge Transitions (M → S) ###

    | Size | Name / Description
    +------+-------------------
    |   64 | Common part
    |    4 | State
    |   96 | Zeros / Garbage

Sent by master on pressing the *Enable / Disable screen edge transitions* 
hotkey (*Master Preferences* tab).

Observed values for `State`:

 * 0x00: transitions enabled
 * 0x01: transitions disabled



### 0x17 - Session Termination (S → M) ###

    | Size | Name / Description
    +------+-------------------
    |   64 | Common part
    |  100 | Zeros / Garbage

Sent when:

 * ID process is terminated (because of exit or shutdown / reboot)
 * machine goes into standby
 * ID is disabled (via *Disable Input Director* on the *Main* tab)



### 0x18 - Unknown ###

Unknown. Never seen.



### 0x19 - Encryption Config Mismatch (S → M) ###

    | Size | Name / Description
    +------+-------------------
    |   64 | Common part
    |    4 | Response to Sequence Number
    |    1 | Encryption Type
    |    1 | Encryption Password Test
    |    2 | Unknown
    |   92 | Zeros / Garbage

Sent by a slave upon receiving a *Session Setup* and *Slave Config Request*
with encryption enabled, but for which the configuration or the password test
byte do not equal those of the master ('None' is also considered an encryption
type).

The `Response to Sequence Number` contains the value of the `Sequence Number`
field in the received *Slave Config Request*.

The `Encryption Type` and `Encryption Password Test` fields represent the
configuration of the intended slave.



### 0x1A - Session Setup (M → S) ###

    | Size | Name / Description
    +------+-------------------
    |   64 | Common part
    |  100 | Zeros / Garbage

Sent by a master to start a new session with a slave, either because the slave
was just added to the configuration of the master, or for some other reason.

The *Session Key* in the *Common part* of this packet is the key that will be
used in all subsequent communication, in case the slave completes the session
setup.

TODO: encryption checking and rejection.



### 0x1B - Session Setup ACK (S → M) ###

    | Size | Name / Description
    +------+-------------------
    |   64 | Common part
    |  100 | Zeros / Garbage

Sent by a slave in response to a *Session Setup* packet.

This packet is needed for the master to continue to the *Slave Config Request*
state (see *0x05 - Slave Config Request*).



### 0x1C - Slave Announce (S → M) ###

    | Size | Name / Description
    +------+-------------------
    |   64 | Common part
    |  100 | Zeros / Garbage

Sent by ID in slave mode every 5 seconds, for as long as it doesn't have an 
active master.



### 0x1D - Slave Heartbeat (S → M) ###

    | Size | Name / Description
    +------+-------------------
    |   64 | Common part
    |  100 | Zeros / Garbage

Sent every 120 seconds of inactivity.




TODO
----

 * Figure out packet types 0x01, 0x08, 0x0C, 0x0D, 0x0E, 0x18 and 0x1E (get 
   captures)
 * Determine whether the 'header' is actually `Header Length` long, instead of
   only the first 64 bytes. That would explain why there are so many 'duplicate'
   fields. Perhaps the first few 4 byte fields are general purpose and are
   interpreted based on `Packet Type`, with the rest of the header being more
   strictly defined.
 * See if *Slave Config Reply* changes in situations with multiple monitors
   attached to slaves




[wireshark]:            http://www.wireshark.org
[uix5/packet-inpdir-udp]: http://www.github.com/uix5/packet-inpdir-udp
[INPUT structure]:      http://msdn.microsoft.com/en-us/library/windows/desktop/ms646270
[KEYBDINPUT structure]: http://msdn.microsoft.com/en-us/library/windows/desktop/ms646271
[MOUSEINPUT structure]: http://msdn.microsoft.com/en-us/library/windows/desktop/ms646273

[Language Identifier Constants and Strings]: http://msdn.microsoft.com/en-us/library/windows/desktop/dd318693
[Standard Clipboard Formats]: http://msdn.microsoft.com/en-us/library/windows/desktop/ff729168

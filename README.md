# web-km200
How to interact with Buderus Heating Systems through the Web KM200 device.

The Web KM200 device uses DHCP to acquire an IPv4 address in your home network.
It shows up with host name TK-850-JH3E-NET in the DHCP server. 
Search engines find a PDF manual for an embedded development board for this string at 
[https://www.tessera.co.jp/Download/TK-850JH3E+NET_UM_E.pdf](https://www.tessera.co.jp/Download/TK-850JH3E+NET_UM_E.pdf).
The manual describes an example program for the board which is a web server.
The Web KM200 provides an open TCP Port 80.  
Buderus is probably using this embedded board inside the web KM200 and the software running on it 
was probably developed by modifying the example web server program.

Some home automation systems like openHAB and fhem and some standalone skripts can communicate with the Web KM200, and at least read values from the heating system. Some reverse engineering has apparently already been done, or some information was transferred from Buderus to the implementors of existing software.
The necessary information how to talk to the Web KM200 is already present in the code out there,

Here, this existing information is merely summarized in English instead of in code.

# Preparation

It seems to be necessary to 
* allow the device to connect to the internet and download firmware,
* download the Buderus app to a smartphone, 
* tick several boxes to allow Buderus everything they want from your firstborn and more, 
* set up a private password for the web KM device with the smartphone app.

After that, device owners who want to interact with their Buderus heating system locally without using the Buderus app and without sending their system's data to Buderus' servers may block internet access  for the Web KM200 device, and delete the smartphone app.

# Local HTTP communication
When opening http://TK-850-JH3E-NET/ in a browser, then an empty web page displays. Any URL other than / shows

> Sorry, the requested file does not exist on this server.

This includes URLs that should exist like http://TK-850-JH3E-NET/gateway/DateTime.

The existing software solutions use a custom user-agent header in the http request, 
TeleHeater/x.y.z, 
where x.y.z is a version number.
When making a GET request after changing the browser's user agent to TeleHeater
(with or without slash version number), the response is different: 
The web server responds with a "Content-Type: application/json" header, 
but the body will not contain json data but contain some base64-encoded data, 
which causes browsers to either show a different error message like 
"unable to parse json" with a button to show the raw data, or show the raw data directly.

Easier than using the browser with a changed user agent is to retrieve the encoded data with
a tool made for this purpose like curl or wget:

    wget -U TeleHeater http://TK-850-JH3E-NET/gateway/DateTime
    curl -A TeleHeater http://TK-850-JH3E-NET/gateway/DateTime

The data is base64-encoded, encrypted JSON. To make use of the contained data, it needs to be decrypted.
To be able to decrypt, the decryption key must be known.

# Derive the decryption key
The decryption key is created from 3 pieces of information:
- The gateway password of the Web KM200 device as printed on a sticker on the outside of that device, without the dashes. 16 characters. Example: NeUCsyQMLVYqKJec
- The private password previously created in the smartphone app. Example: HnE75f+a%aXP
- A sequence of 32 magic bytes. These: 
    0x86, 0x78, 0x45, 0xe9, 0x7c, 0x4e, 0x29, 0xdc, 0xe5, 0x22, 0xb9, 0xa7, 0xd3, 0xa3, 0xe0, 0x7b, 0x15, 0x2b, 0xff, 0xad, 0xdd, 0xbe, 0xd7, 0xf5, 0xff, 0xd8, 0x42, 0xe9, 0x89, 0x5a, 0xd1, 0xe4

The decryption key is the concatenation of the MD5 sums of
1. The concatenation of the gateway password and the magic byte sequence.
2. The concatenation of magic byte sequence and the private password.

For example in bash, the decryption key from the above example passwords can be derived like this:

    GatewayPassword=NeUCsyQMLVYqKJec
    PrivatePassword='HnE75f+a%aXP' # Make sure to properly escape special characters for bash if necessary
    Magic=$(printf \\x86\\x78\\x45\\xe9\\x7c\\x4e\\x29\\xdc\\xe5\\x22\\xb9\\xa7\\xd3\\xa3\\xe0\\x7b\\x15\\x2b\\xff\\xad\\xdd\\xbe\\xd7\\xf5\\xff\\xd8\\x42\\xe9\\x89\\x5a\\xd1\\xe4)
    Part1=$(echo -n "$GatewayPassword$Magic" | md5sum | cut -c-32)
    Part2=$(echo -n "$Magic$PrivatePassword" | md5sum | cut -c-32)
    Key="$Part1$Part2"
    echo $Key

91df2cd7631c309f2027b89a5126a481bf39ade2565b0af0947faad456a5cc9c

This is the hex representation of the key, the key has 32 bytes, the first byte of the key is 0x91, the last one is 0x9c.

# Decrypt the data

The data received from the KM200 is encrypted with Rijndael in ECB mode with the 128 bit (32 byte) key derived above.
This should be enough information to decrypt the received data, but the various KM200 software modules mentioned above 
demonstrate how the decryption is done in various programming languages. 

For completeness, the encrypted data can be decrypted in bash with openSSL like this:

    grep .. DateTime | base64 --decode | openssl enc -aes-256-ecb -d -nopad -K $Key

This decrypts data previously received with wget and saved in file "DateTime".  `grep ..` part suppresses the empty first line of the web server response, the `base64 --decode` command extracts the binary ciphertext from the base64 encoded data. 
Finally, the openssl command decrypts the ciphertext. 
`enc` tells openssl to encrypt or decrypt, with the `-d` option later chooses decryption rather than encryption.
The -aes-256-ecb selects the decryption algorithm, this is the correct choice for decrypting rijndal-128 in ecb mode.
The -nopad argument tells openssl that no padding on the part of openssl is needed 
because the encrypted web server response is already padded to a multiple of the decryption block size: The Web KM 200 has appended bytes with value 0x00 to the cleartext for this purpose.
The -K $Key finally specify the encryption key as derived in the previous section, it can be given here in hex.

Although the padding bytes with value 0x00 will not show up when displaying the decrypted json in a terminal, they are present at the end of the output stream of the openssl decryption command. Depending on how the json data is processed after the encryption, they may or may not hamper the post-processing. It is possible to remove these 0x00 bytes from the decrypted output by piping the output through `tr -d '\0'`.

Retrieving the latest data and decryption can be combined like this:

    curl -A TeleHeater http://TK-850-JH3E-NET/gateway/DateTime -s | grep .. | base64 --decode | openssl enc -aes-256-ecb -d -nopad -K $Key | tr -d '\0'

# Interpret the decrypted data

The command above returns readable json data, e.g.

    {"id":"/gateway/DateTime","type":"stringValue","writeable":1,"recordable":0,"value":"2022-03-21T20:10:58"}

where "id" contains a copy of the path that we asked for while "value" contains
the result data. To extract only the value using the shell, the command line
tool `jq` can be used:

    curl -A TeleHeater -s http://TK-850-JH3E-NET/gateway/DateTime | grep .. | base64 --decode | openssl enc -aes-256-ecb -d -nopad -K $Key | tr -d '\0' | jq .value

## Interpret "switchProgram" data

Some URIs contain data of type "switchProgram". "Switch programs" control at
which times and on which weekdays hot water and rooms are to be heated and
to what temperatures. E.g., the switch program for generating hot water can
be fetched and decrypted from URL
http://TK-850-JH3E-NET/dhwCircuits/dhw1/switchPrograms/A may look like this:

    {
      "id": "/dhwCircuits/dhw1/switchPrograms/A",
      "type": "switchProgram",
      "setpointProperty": {
        "id": "/dhwCircuits/dhw1/temperatureLevels",
        "uri": "http://192.168.178.33/dhwCircuits/dhw1/temperatureLevels"
      },
      "maxNbOfSwitchPoints": 42,
      "maxNbOfSwitchPointsPerDay": 6,
      "switchPointTimeRaster": 15,
      "writeable": 1,
      "switchPoints": [
        {
          "dayOfWeek": "Mo",
          "setpoint": "high",
          "time": 390
        },
        {
          "dayOfWeek": "Mo",
          "setpoint": "off",
          "time": 600
        }
      ]
    }

A switch program contains "switch points".  In this example, there can be
up to 6 switch points per day and a total of up to 42 switch points per week.
The switch points are listed  in the array "switchPoints". Each switch point
is a JSON object with the fields "dayOfWeek", "setpoint", and "time".
"dayOfWeek" contains the first two letters of the English names for weekdays,
"time" contains the minutes past midnight when this switch point switches. The
"time" value must be a multiple of the value in "switchPointTimeRaster".
The string value contained in field "setpoint" can be appended to the URI
contained in "setPointProperty", separated by the path separator "/". This
combined URI can be used to read the desired temperature setting.

The shown example would switch the hot water heater to setpoint "high" every
Monday 390 minutes after midnight, which is at 6:30 AM, and to setpoint "off"
at 10:00 AM. A real-world switch program would also contain entries for the
other weekdays.

# Find valid paths recursively

With some constraints, valid paths where values can be read or written can be
found recursively:  Take the gateway/DateTime example: we can list the entries
of directory gateway by asking the web-km200 for the path gateway:

    curl -A TeleHeater -s http://TK-850-JH3E-NET/gateway | grep .. | base64 --decode | openssl enc -aes-256-ecb -d -nopad -K $Key | tr -d '\0'

which may respond with JSON similar to the following:

    {
      "id": "/gateway",
      "type": "refEnum",
      "references": [
        {
          "id": "/gateway/uuid",
          "uri": "http://192.168.178.33/gateway/uuid"
        },
        {
          "id": "/gateway/firmware",
          "uri": "http://192.168.178.33/gateway/firmware"
        },
        {
          "id": "/gateway/userpassword",
          "uri": "http://192.168.178.33/gateway/userpassword"
        },
        {
          "id": "/gateway/versionFirmware",
          "uri": "http://192.168.178.33/gateway/versionFirmware"
        },
        {
          "id": "/gateway/versionHardware",
          "uri": "http://192.168.178.33/gateway/versionHardware"
        },
        {
          "id": "/gateway/boschSHPassword",
          "uri": "http://192.168.178.33/gateway/boschSHPassword"
        },
        {
          "id": "/gateway/portalPassword",
          "uri": "http://192.168.178.33/gateway/portalPassword"
        },
        {
          "id": "/gateway/knxPassword",
          "uri": "http://192.168.178.33/gateway/knxPassword"
        },
        {
          "id": "/gateway/haiPassword",
          "uri": "http://192.168.178.33/gateway/haiPassword"
        },
        {
          "id": "/gateway/DateTime",
          "uri": "http://192.168.178.33/gateway/DateTime"
        },
        {
          "id": "/gateway/instPassword",
          "uri": "http://192.168.178.33/gateway/instPassword"
        },
        {
          "id": "/gateway/instAccess",
          "uri": "http://192.168.178.33/gateway/instAccess"
        },
        {
          "id": "/gateway/instWriteAccess",
          "uri": "http://192.168.178.33/gateway/instWriteAccess"
        },
        {
          "id": "/gateway/version",
          "uri": "http://192.168.178.33/gateway/version"
        },
        {
          "id": "/gateway/update",
          "uri": "http://192.168.178.33/gateway/update"
        },
        {
          "id": "/gateway/logging",
          "uri": "http://192.168.178.33/gateway/logging"
        }
      ]
    }

However, not all listed entries can actually be read, and it may also be that
not all paths that can be read are actually listed here. Some of the listed
entries are themselves directories, they can be recognized by their
`"type": "refnum"` property in their JSON response when querying them.

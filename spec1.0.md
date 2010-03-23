FBI's protocol spec v1.0
========================

Terminology
-----------
There are a few terms used by FBI's spec which may have unclear meanings:

* **router**: The server which serves FBI connections. Components connect to a router to communicate. Compare to IRC servers.
* **component**: A script which maintains a connection to an FBI router. Components may subscribe to channels or publish data. Compare to IRC clients.
* **channel**: FBI Channels are very much like IRC channels. Components may subscribe (join) or unsubscribe (part) channels. However, at this time a component doesn't need to subscribe to a channel to publish payloads to it.
* **packet**: A single JSON structure, sent between a router and a component. FBI packets may span multiple network packets, as they do not have a size limit. All packets have an action, which is commonly used to name the packet; i.e. a packet with "action" set to "publish" is referred to as a publish packet.


Basic protocol
--------------
FBI's protocol spec is both very basic and somewhat complex at the same time. At the simplest level, all connections to FBI routers must be in newline-terminated JSON form. That is, all packets sent and received by a component (or router) will be valid JSON structures, terminated with a UNIX-style "\n" newline. Because the structure is encoded as JSON, newlines in strings must be escaped, and will not interfere with the line terminations.

For now, the structure of the JSON hashes is pretty loosely-defined. The only requirement is that the root hash has an "action" key. The corresponding value is used to determine what the packet contains or wants to do.

Defined actions in this early spec are: welcome, auth, subscriptions, components, channels, subscribe, unsubscribe, publish, and disconnect. Welcome packets are special packet; they are only ones which are never sent by the client.

Many packets try to include an 'origin' or 'target'. Later on this will likely become a packet-wide requirement.

Here is an example conversation, where << denotes packets which the component received and >> denotes packets which the component sent.

    ** component connects to router
    << {"name":"Development FBI Router","action":"welcome","origin":"$home.danopia.net"}
    >> {"action":"auth","user":"irc","secret":"hil0l"}
    << {"action":"auth","origin":"irc","user":"www"}
    >> {"action":"subscribe","channels":["#commits","#test"]}
    << {"action":"subscribe","origin":"irc","channels":["#commits","#test"]}
    >> {"data":{"test":"hello world"},"action":"publish","target":"#test"}
    << {"data":{"test":"hello world"},"origin":"irc","action":"publish","target":"#irc"}
    >> {"data":{"test":"hello world"},"action":"publish","target":"#irc"}
    ** nothing is received to ACK the publish
    >> {"action":"disconnect"}
    << {"action":"disconnect","origin":"irc"}
    ** router closes connection

Defined actions
---------------
### welcome
Sent by the server when it receives a connection. Contains the server name as the 'origin' and a short description of the server as 'name'. Can be safely ignored.


### Changing connection state

#### auth user, secret
Sent by the component to attempt logging in with a username and password. When successful, the 'secret' entry is removed and the packet is echoed back.

#### subscribe channels
Sent by the component to request a subscription to the channels listed under 'channels'. Echoed back with the successful subscriptions from the packet listed under 'channels' (in case you were already in some).

NOTE: This packet will also be sent to other subscribers of the channels if you are logged in when you subscribe. Use the packet 'origin' to determine who subscribed.

#### unsubscribe channels
Sent by the component to request removal from the channels listed under 'channels'. Echoed back with the successful unsubscriptions from the packet listed under 'channels' (in case you weren't already in some).

NOTE: This packet will also be sent to other subscribers of the channels if you are logged in when you unsubscribe. Use the packet 'origin' to determine who left.

#### disconnect
Sent by the component to request that the server connection be closed cleanly. A response is sent but the current implementation fails to send it.


### Interaction

#### publish data, target
Publish a data payload (which should be a JSON hash structure) to the specified target. Prefixes are used to determine if the target is a channel or component, allowing you to publish data directly to certain components. The publishing component's name is stored under 'origin'. The packet is then relayed to all recipients (via a channel subscription list or direct target).

If a component publishes to a channel to which it also subscribes, then the router will also relay the packet back to the origin to serve as a type of acknowledgement.

### Introspection

#### subscriptions
Sent by the component to request a list of channels which it has subscribed to. Returned with the array of names under 'channels'.

#### components
Sent by the component to request a list of components which are connected to the server. Returned with the array of names under 'components'.

#### channels
Sent by the component to request a list of channels which exist on the server. Returned with the array of names under 'channels'.


Naming scheme
-------------
Similarly to IRC, the FBI router reserves certain namespaces for certain objects. The symbols ~!#$%^&* are all reserved prefixes and may NOT be used to start a component's name. Channels are prefixed with # and therefore can start with anything (even Unicode!). Server hostnames are prefixed with $ (so the server may advertise itself as $fbi.danopia.net). The other prefixes are not used but may be in the future. Publishing data to server targets is not defined.

To allow for future expansion of the FBI network schema, NO name may include the "at" sign (@). (In a future protocol, routers may link and use their hostname to prevent inter-router conflicts, so the "www" component on one server will be globally labeled www@server1.danopia.net and will not conflict with the www component on another server.) Channels will probably share a global namespace but still have this restriction in place just in case.

As mentioned above, all names may include Unicode in the UTF-8 encoding. They may also include spaces, although such usage is frowned upon for the sake of CLI simplicity. The line is placed at newlines; there is no practical reason why a component name would require a newline, and newlines would break some functionality (i.e. typing in a component name at the commandline or even a singleline textbox in a GUI app or website).

Channels are prefixed by the # symbol. Please note that the current implementation will allow you to subscribe to channels without the prefix but you cannot publish to them. This behavior is caused by a lack of validation and is considered a bug.


Defined channels
----------------
At this point in the spec, any channel may contain any data structure. At a later time, individual channels may be restricted to only carrying packets with a certain structure. Even without the restriction in place it is *highly suggested* that a component only sends relevant data to common channels and in the same format as the existing structures.

Below are some examples of common channels and the structures which should be used. Note that the current router implementation detects 'url' entries in publish payloads and adds a 'shorturl' entry with a shortened URL (using public services such as is.gd and tinyurl.com) but this behavior should not be relied upon.

### #commits
Like some other channels, #commits payloads are an array of hashes (for now). The structures in the array resemble this:

    "project" => "fbi",
    "owner" => "danopia",
      # fork is GitHub specific, other sources should use false
    "fork" => false,
      # use nil for ["author"]["email"} if your source has none
    "author" => {"email" => "me@danopia.net", "name" => "Daniel Danopia"},
    "branch" => "master", # or nil
      # send commit as a number (not string) if your source has linear history
    "commit" => "786266a76e09f8c5dad4637534db81e40249c7b7",
      # abbreviated for clarity, but the whole message should be sent
    "message" => "Added a new feature.\n\nNow it is possible for (...)", 
    "url" => "http://github.com/danopia/fbi/commit/786266a76e09f8c5dad4637534db81e40249c7b7"

Please remember that the router may add a 'shorturl' entry next to every 'url' entry.

### #mailinglist
Mailinglist payloads are also an array of hashes, although this will probably change soon, as messages don't tend to be processed in batches, unlike distributed SCM hosts.

    "list" => "ooc-dev.lists.launchpad.net",
    "author" => "Amos Wenger",
    "subject" => "Re: [Ooc-dev] COS",
    "url" => "http://lists.launchpad.net/ooc-dev",
    "project" => "ooc"

The only official mailinglist source doesn't send a unique URL directly pointing to the entry for limitations in the emails which are sent out. Other sources won't do this; and later on the current source may not, either.

### #irc
This channel is the outlier. It will soon be replaced by #commands.

Payloads are JSON structures (not arrays). Example payload from "luser" sending "FBI-1: hey there, partner!" to "#commits":

    "server" => 2, # used to send replies
    "channel" => "#commits", # also used to send replies
    "sender" => "luser", # also used to send replies
    "command" => "hey",
    "args" => "there, partner!",
    "admin" => false, # true if the origin is a bot admin
    "default_project" => "danopia/fbi" # or nil

Note that the payload will not be sent unless the command 1) addressed or PM'ed an FBI IRC bot and 2) didn't have a handler set up in the bot process itself.

Error handling
--------------
The easiest way to describe the protocol-level error handling is that there isn't one. There will be soon. I have to decide whether there will be an "error" action which includes the invalid request or whether the bad packet will be relayed back with "error":true or some other thing. I am, however, leaning towards the former.

# Chapter 1 - Asterisk

https://aosabook.org/en/v1/asterisk.html

A server application for making, receiving, and performing custom processing of phone calls.\
Supports many technologies for making and receiving phone calls
- many VoIP (Voice over IP) protocols
- analog and digital connectivity 
-- traditional telephone network
-- PSTN (Public Switched Telephone Network


## 1.1. Critical Architectural Concepts

### 1.1.1. Channels
A *channel* represents a connection between the Asterisk system and a telephony endpoint.\
E.g. when a phone makes a call into an Asterisk system.\
Instance of the *ast_channel* data structure.

### 1.1.2. Channel Bridging
When person A calls Person B through Asterisk, two channels are created. One is connected to Phone A and the other to Phone B. The two are then bridged inside Asterisk.

Channel bridging is the act of connecting channels together for the purpose of passing media between them.\
Most commonly an audio stream, but there may also be a video or a text stream in the call.\
When there is more than one media stream (such as both audio and video), it is still handled by a single channel for each end of the call.

All media streams are negotiated through Asterisk. Anything that Asterisk does not understand and have full control over is not allowed. This means that Asterisk can do recording, audio manipulation, and translation between different technologies.

When two channels are bridged together, there are two methods that may be used to accomplish this: 
- generic bridging
- native bridging

*Generic bridge*
- works regardless of what channel technologies are in use
- passes all audio and signalling through the Asterisk abstract channel interfaces 
- most flexible bridging method
- least efficient due to the levels of abstraction necessary to accomplish the task

*Native bridge*
- technology-specific method of connecting channels
- If two channels are connected to Asterisk using the same media transport technology, it may be possible to connect them more efficiently than going through the abstraction layers in Asterisk 
-- when specialized hardware is being used for connecting to the telephone network, it may be possible to bridge the channels on the hardware only - the media does not need to flow up through the application at all 
--- In some VoIP protocols, it is possible to have endpoints send their media streams to each other directly, such that only the call signalling information continues to flow through the server

When it's time to bridge two channels, the decision between generic and native bridging is made by comparing them. 
- If both support the same native bridging method, then that will be used 
- Otherwise, generic bridging method will be used 

To determine whether or not two channels support the same native bridging method, a simple C function pointer comparison is used. 
- not the most elegant method
- works sufficiently for their needs 

### 1.1.3. Frames

Communication within Asterisk during a call is done by using frames - instances of the *ast_frame* data structure. 
Frame types
- media frames
- signalling frames

During a basic phone call, a stream of media frames containing audio would be passing through the system.\
Signalling frames are used to send messages about call signalling events, such as a digit being pressed, a call being put on hold, or a call being hung up.

The list of available frame types is statically defined.\
Frames are marked with a numerically encoded type and subtype. 

- VOICE: carry a portion of an audio stream.
- VIDEO: carry a portion of a video stream.
- MODEM: 
-- The encoding used for the data in this frame, such as T.38 for sending a FAX over IP. 
-- primary usage is for handling a FAX. 
-- frames of data be left completely undisturbed so that the signal can be successfully decoded at the other end 
--- different than AUDIO frames where it is acceptable to transcode into other audio codecs to save bandwidth at the cost of audio quality
- CONTROL: The call signalling message that this frame indicates - used to indicate call signalling events 
-- phone being answered, hung up, put on hold, etc.
- DTMF_BEGIN: Which digit just started. This frame is sent when a caller presses a DTMF key on their phone
- DTMF_END: Which digit just ended. This frame is sent when a caller stops pressing a DTMF key on their phone

## 1.2. Asterisk Component Abstractions
Asterisk is highly modularized.\
Core application is built from the source in the main/ directory of the source tree.\
Acts primarily as a module registry.\
Has code that knows how to connect all of the abstract interfaces together to make phone calls work.\
The concrete implementations of these interfaces are registered by loadable modules at runtime.

By default, *all* modules found in a predefined /modules directory on the filesystem will be loaded when the main application is started.\
This approach was chosen for its simplicity.\
However, a configuration file can be updated to specify exactly which modules to load and in what order to load them.\
Makes the configuration a bit more complex, but provides the ability to specify that modules that should not be loaded.
- reduces the memory footprint  
- security benefits, as well - best not to load a module that accepts connections over a network if it is not actually needed

When a module loads, it registers all of its implementations of component abstractions with the core application.\
There are many types of interfaces that modules can implement and register.\
Generally, related functionality is grouped into a single module.

### 1.2.1. Channel Drivers
The channel driver interface is the most most important interface.\
The channel API provides the abstraction that allows all other Asterisk features to work independently of the telephony protocol in use.\
Responsible for translating between the Asterisk channel abstraction and the details of the telephony technology that it implements, e.g. an abstract *ast_channel* is associated with a concrete implementation through ast_channel_tech.

ast_channel_tech defines a set of methods that must be implemented by a channel driver. 
- *ast_channel* factory method - the requester method in ast_channel_tech 
-- When an Asterisk channel is created, for an incoming/outgoing phone call, the implementation of ast_channel_tech associated with the type of channel needed is responsible for instantiation and initialization of the ast_channel for that call

Once an ast_channel has been created, it has a reference to the ast_channel_tech that created it.\
Many other operations must similarly be handled in a technology-specific way.\
When those operations must be performed on an ast_channel, the handling of the operation is deferred to the appropriate method from ast_channel_tech.


The most important methods in ast_channel_tech:\
requester: callback used to request a channel driver to instantiate an *ast_channel* object and initialize it for a given channel type.
- call: callback used to initiate an outbound call to the endpoint represented by an *ast_channel*
- answer: called when answers the inbound call associated with this *ast_channel*.
- hangup: called when the system has determined that the call should be hung up. The channel driver will then communicate to the endpoint that the call is over in a protocol specific manner
- indicate: Once a call is up, there are a number of other events that may occur that need to be signalled to an endpoint.
- E.g. if the device is put on hold, this callback is called to indicate that condition.
-- There may be a protocol specific method of indicating that the call has been on hold
-- the channel driver may simply initiate the playback of music on hold to the device
- send_digit_begin: This function is called to indicate the beginning of a digit (DTMF) being sent to this device.
- send_digit_end: This function is called to indicate the end of a digit (DTMF) being sent to this device
- read: This function is called by the core to read back an ast_frame from this endpoint
- write: This function is used to send an ast_frame to this device. The channel driver will take the data and packetize it as appropriate for the telephony protocol that it implements and pass it along to the endpoint
- bridge: This is the native bridge callback for this channel type. Important for performance reasons

Once a call is over, the abstract channel handling code in the core will invoke the *ast_channel_tech* hangup callback and then destroy the *ast_channel* object

### 1.2.2. Dialplan Applications
Call routing is configured in the *dialplan* in the /etc/asterisk/extensions.conf file.\
Consists of a series of call rules ("extensions").

When a phone call comes in, the dialed number is used to determine the extension that should be used for processing the call.\
The extension includes a list of dialplan *applications* which will be executed on the channel.\
The applications available for execution in the dialplan are maintained in an application registry, populated at runtime as modules are loaded.

Asterisk has nearly two hundred included applications.\
Applications can use any of the internal APIs to interact with the channel.\
Some applications do a single task, e.g. Playback, which plays back a sound file to the caller.\
Others perform a large number of operations, such as Voicemail.

Multiple applications can be used together to customize call handling using dialplans.\
If more extensive customization is needed, there are scripting interfaces available that allow call handling to be customized using any programming language.\
When using these, dialplan applications are still invoked to interact with the channel.

Example of dialplan omitted

The function prototype of a registered application is simply:
```
int (*execute)(struct ast_channel *chan, const char *args);
```
However, the application implementations use virtually all of the APIs found in include/asterisk/.

### 1.2.3. Dialplan Functions
Most dialplan applications take a string of arguments.\
Some values may be hard coded, variables are used in places where behavior needs to be more dynamic. 

Example omitted

Modules can register dialplan functions that can retrieve some information and return it to the dialplan.\
Dialplan functions can also receive data from the dialplan and act on it. 

As a general rule, while dialplan functions may set or retrieve channel meta data, they do not do any signalling or media processing.\
That is the job of dialplan applications.

Example omitted


### 1.2.4. Codec Translators

In VOIP, many different codecs are used for encoding media to be sent across networks.\
The variety of choices offers tradeoffs in media quality, CPU consumption, and bandwidth requirements.\
Asterisk supports many different codecs and can translate between them.

When a call is set up, Asterisk will attempt to get two endpoints to use a common media codec to avoid transcoding.\
Not always possible. Even if a common codec is being used, transcoding may still be required. 

E.g. if Asterisk is configured to do some signal processing on the audio as it passes through the system (e.g. increase or decrease the volume level), it will need to transcode the audio back to an uncompressed form before it can process it. 

Asterisk can also be configured to do call recording.\
If the configured format for the recording is different than that of the call, transcoding will be required.

#### Codec Negotiation

The method used to negotiate which codec will be used for a media stream is specific to the technology used to connect the call to Asterisk.\
In some cases (e.g. the traditional telephone network (the PSTN)) there may not be any negotiation to do.\
In other cases, especially using IP protocols, there is a negotiation mechanism 
- capabilities and preferences are expressed
- a common codec is agreed upon

Example omitted

High level view of codec negotiation in SIP (the most commonly used VOIP protocol):\
An endpoint sends a new call request to Asterisk with a list of codecs it is willing to use.\
Asterisk consults its configuration which includes a list of allowed codecs in preferred order.\
Responds by choosing the most preferred codec (based on its own configured preferences) that is listed in both.

Asterisk does not handle complex codecs well, especially video. Codec negotiation demands have gotten more complicated over the last ten years. 

Codec translator modules provide implementations of the *ast_translator* interface. 
- has source and destination format attributes
- provides a callback used to convert a chunk of media from the source to the destination format
- knows nothing about the concept of a phone call - only knows how to convert media from one format to another

See include/asterisk/translate.h and main/translate.c.\
Implementations of the translator abstraction can be found in the codecs directory.

## 1.3. Threads
Asterisk makes heavy use of threads.\
Uses the POSIX threads API to manage threads and related services such as locking.\
Code that interacts with threads does so by going through a set of wrappers used for debugging purposes.\
Most threads in Asterisk can be classified as either
- a Network Monitor Thread
- a Channel Thread ("PBX thread", because its primary purpose is to run the PBX for a channel)

### 1.3.1. Network Monitor Threads
Network monitor threads exist in every major channel driver.\
Responsible for monitoring whatever network they are connected to (IP network, the PSTN, etc.) and monitor for incoming calls or other types of incoming requests.\
Handle initial connection setup steps such as authentication and dialed number validation.\
Once the call setup has been completed, the monitor threads will create an *ast_channel*, and start a channel thread to handle the call for the rest of its lifetime.

### 1.3.2. Channel Threads
Channels are inbound or outbound.\
An inbound channel is created when a call comes in - executes the Asterisk dialplan.\
A channel thread is created for every inbound channel that executes the dialplan.

Dialplan applications always execute in the context of a channel thread.\
Dialplan functions almost always do, as well. 

It is possible to read and write dialplan functions from an asynchronous interface such as the Asterisk CLI.\
However, it is still always the channel thread that is the owner of the ast_channel data structure and controls the object lifetime.

## 1.4. Call Scenarios

### 1.4.1. Checking Voicemail
The first major component involved in this scenario is the channel driver.\
Responsible for handling the incoming call request from the phone, which will occur in the channel driver's monitor thread. 

Depending on the telephony technology being used to deliver the call to the system, there may be some sort of negotiation required to set up the call. 

Another step of setting up the call is determining the intended destination for the call.\
Usually specified by the number that was dialed by the caller.\
In some cases there is no specific number available since the technology used to deliver the call does not support specifying the dialed number (e.g. incoming call on an analog phone line).

After the channel driver verifies that the configuration has extensions defined in the dialplan for the dialed number, it allocates a ast_channel object and creates a channel thread.\
The channel thread is responsible for handling the rest of the call.

The main loop of the channel thread handles dialplan execution.\
Uses the rules defined for the dialed extension and executes the steps that have been defined. 

Example omitted

When the channel thread executes the Answer application, Asterisk will answer the incoming call.\
Answering a call requires technology specific processing, so in addition to some generic answer handling, the answer callback in the associated ast_channel_tech structure is called to handle answering the call.\
This may involve sending a special packet over an IP network, taking an analog line off hook, etc.

Next the the channel thread executes VoicemailMain application, provided by the app_voicemail module.\
While the Voicemail code handles a lot of call interaction, it knows nothing about the technology used to deliver the call into the system.\
The channel abstraction hides these details from the Voicemail implementation.

There are many features involved in providing a caller access to their Voicemail.\
All are primarily implemented as reading and writing sound files in response to input from the caller (e.g. digit presses).\
DTMF digits can be delivered to Asterisk in many different ways - these details are handled by the channel drivers.\
Once a key press has arrived , it is converted into a generic key press event and passed along to the Voicemail code.

One of the important interfaces in Asterisk that has been discussed is that of a codec translator.\
These codec implementations are very important to this call scenario.\
 When the Voicemail code would like to play back a sound file to the caller, the format of the audio in the sound file may not be the same format as the audio being used in the communication between the Asterisk system and the caller.\
 If it must transcode the audio, it will build a translation path of one or more codec translators to get from the source to the destination format.

At some point, the caller is done and hangs up.\
The channel driver detects this and converts it into a generic channel signalling event.\
The Voicemail code receives the event and exits, since there is nothing left to do once the caller hangs up.\
Control will return back to the main loop in the channel thread to continue dialplan execution.\
In this example there is no more dialplan processing to be done. The channel driver will be given an opportunity to handle technology specific hangup processing and then the ast_channel object will be destroyed.

### 1.4.2. Bridged Call
Scenario when one phone calls another through the system.\
The initial call setup process is identical to the previous example.\
The difference begins when the call has been set up and the channel thread begins executing the dialplan.

The following dialplan is a simple example that results in a bridged call.\
Using this extension, when a phone dials 1234, the dialplan will execute the Dial application, which is the main application used to initiate an outbound call.

Example omitted 

The argument specified to the application says that the system should make an outbound call to the device referred to as SIP/bob.\
The SIP portion of this argument specifies that the SIP protocol should be used to deliver the call.\
bob will be interpreted by the channel driver that implements the SIP protocol, *chan_sip*.\
Assuming the channel driver has been configured with an account called bob, it will know how to reach Bob's phone.

The Dial application will ask the core to allocate a new channel using the SIP/bob identifier.\
The core will request that the SIP channel driver perform technology specific initialization.\
The channel driver will also initiate the process of making a call out to the phone.\
As the request proceeds, it will pass events back into the core, which will be received by the Dial application. 

Events may include a response that
- the call has been answered
- the destination is busy
- the network is congested
- the call was rejected for some reason
- etc 

Ideally the call is answered - this information is then propagated back to the inbound channel.\
Asterisk will not answer the part of the call that came into the system until the outbound call was answered.\
Once both channels are answered, the bridging of the channels begins.

During a channel bridge, audio and signalling events from one channel are passed to the other until some event occurs that causes the bridge to end, such as one side of the call hanging up.\
Once the call is done, the hangup process is similar to voicemail example.\
The major difference here is that there's two channels involved - the technology specific hangup processing is executed for both before the channel thread stops running.

### 1.5. Final Comments

Channels and flexible call handling using the Asterisk dialplan has enabled the the development of complex telephony systems in an industry that is continuously evolving over many years.\
The architecture of does however not enable scaling the system across multiple servers.\
Asterisk SCF (Scalable Communications Framework) is under development to address these scalability concerns.
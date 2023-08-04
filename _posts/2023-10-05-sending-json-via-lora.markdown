---
title:  "Sending JSON payload with LoRa is trickier than I thought"
date:   2023-10-05 10:15:27 +0700
type: post
tags: [rpi, python, electrical]
---

Recently I'm helping a friend's project about device intercommunication on a two Raspberry Pi 
with embedded [SX1262 868M LoRa HAT](https://www.waveshare.com/wiki/SX1262_868M_LoRa_HAT){:target="_blank"}.

## Logic flow

Outline of the logical flow is described like so:
- There are two RPis, let's say **Node T** (Transmitter) and **Node R** (Receiver)
- Node T will gather several data from a physical sensor, format it as a structured payload, 
and sent the payload to Node R wirelessly with LoRa's help
- Node R will receive the payload, decode it back into usable format, and test these data
with [RandomForestRegressor](https://scikit-learn.org/stable/modules/generated/sklearn.ensemble.RandomForestRegressor.html){:target="_blank"}
models that are already supplied inside the node in a form of pickle dump
- The test result will affect certain state of another physical device that directly connected with Node R

| ![](/assets/2023-10-05/node_t.jpeg) |
|:-----------------------------------:|
|       *Node T (transmitting)*       |

| ![](/assets/2023-10-05/node_r.jpeg) |
|:-----------------------------------:|
|        *Node R (receiving)*         |

## Early state of the code

For what it's worth, my friend already supplied several piece of the code that are already worked 
on each their own scope.
What I did was to only complete the puzzle with some fixes to couple of real world problem that arise in the middle of
the development.

I have no real experiences in RPi, nor in LoRa before.
Most related skills that I had were python and some bytes manipulation in C/C++ that 
I did few years back when I was pursuing my bachelor.

Reading a lot of new & unfamiliar technical terms (such as `GPIO`, `SSR`, `Serial`, `Baudrate`)
felt frightening at first, but also challenging!

Let's start by defining the payload: a single payload on a single transmission cycle is formatted like so:

```json
{
  "timestamp": "2023-10-04 16:25:24.484442",
  "wind_speed": 2,
  "wind_direction": 280,
  "flight_speed": 3.2,
  "flight_altitude": 4.7,
  "interval": 1.5,
  "flight_mode": 0,
  "nozzle": 2
}
```

As you can see, the payload consist of a timestamp (when the payload is created) and several variable that are
obtained from several sensor by Node T.

Node T will generate such payload and transmit it every 1 second, with this code below.
Note that the payload is dumped and passed as `message` when calling the function.

```
def send_message(self, address: int, frequency: int, message: str):
    offset_frequency = frequency - (850 if frequency > 850 else 410)

    payload_high_address = bytes([address >> 8])                        # receiving node's high 8bit address
    payload_low_address = bytes([address & 0xff])                       # receiving node's low 8bit address
    payload_target_frequency = bytes([offset_frequency])                # receiving node's frequency
    payload_self_high_address = bytes([self.get_address() >> 8])        # self high 8bit address
    payload_self_low_address = bytes([self.get_address() & 0xff])       # self low 8bit address
    payload_self_low_frequency = bytes([self.get_offset_frequency()])   # own low frequency
    payload_encoded_message = message.encode()                          # message payload

    bytes_data = (
            payload_high_address
            + payload_low_address
            + payload_target_frequency
            + payload_self_high_address
            + payload_self_low_address
            + payload_self_low_frequency
            + payload_encoded_message
    )
    self.node.send(bytes_data)
```

Similarly, Node R will receive the payload every 1 second with this function:

```
def receive_message(self) -> str:
    if self.node.serial.inWaiting() > 0:
        receive_buffer = self.node.serial.read(self.node.serial.inWaiting())
        sender_address = (receive_buffer[0] << 8) + receive_buffer[1]
        sender_frequency = receive_buffer[2] + self.node.start_freq
        byte_message = receive_buffer[3:-1]
        string_message = byte_message.decode()
        return string_message
```

And here is the first ever log from Node R:

```
Raw message: {"timestamp": "2023-10-04 14:50:34.090499", "wind-speed": 0, "wind-direction": 0, "flight_speed": 0.0, "flight_altitude": 0.0, "interval": 0.0, "flight_mode": 0, "nozzle": 0}
Raw message: {"timestamp": "2023-10-04 14:50:35.386182", "wind-speed": 0, "wind-direction": 0, "flight_speed": 0.0, "flight_altitude": 0.0, "interval": 0.0, "flight_mode": 0, "nozzle": 0}
Raw message: {"timestamp": "2023-10-04 14:50:36.682111", "wind-speed": 0, "wind-direction": 0, "flight_speed": 0.0, "flight_altitude": 0.0, "interval": 0.0, "flight
Raw message: e": 0, "nozzle": 0}
Raw message: {"timestamp": "2023-10-04 14:50:37.977775", "wind-speed": 0, "wind-direction": 0, "flight_speed": 0.0, "flight_altitude": 0.0, "interval": 0.0, "flight_mode": 0, "nozzle": 0}
Raw message: {"timestamp": "2023-10-04 14:50:39.273601", "wind-speed": 0, "wind-direction": 0, "flight_speed": 0.0, "flight_altitude": 0.0, "interval": 0.0, "flight_mode": 0, "nozzle": 0}
Raw message: {"timestamp": "2023-10-04 14:50:40.569374", "wind-speed": 0, "wind-direction": 0, "flight_speed": 0.0, "flight_altitude": 0.0, "interval": 0.0, "flight_mode": 0, "nozzle": 0}
Raw message: {"timestamp": "2023-10-04 14:50:41.865055", "wind-speed": 0, "wind-direction": 0, "flight_speed": 0.0, "flight_altitude": 0.0, "interval": 0.0, "flight_mode": 0, "nozzle": 0}
Raw message: {"timestamp": "2023-10-04 14:50:43.160865", "wind-speed": 0, "wind-direction": 0, "flight_speed": 0.0, "flight_altitude": 0.0, "interval": 0.0, "flight_mode": 0, "nozzle": 0}
Raw message: {"timestamp": "2023-10-04 14:50:44.456780", "wind-speed": 0, "wind-direction": 0, "flight_speed": 0.0, "flight_altitude": 0.0, "interval": 0.0, "flight_mode": 0, "nozzle": 0}
Raw message: {"timestamp": "2023-10-04 14:50:45.752481", "wind-speed": 0, "wind-direction": 0, "flight_speed": 
Raw message:  "flight_altitude": 0.0, "interval": 0.0, "flight_mode": 0, "nozzle": 0}
Raw message: {"timestamp": "2023-10-04 14:50:47.048234", "wind-speed": 0, "wind-direction": 0, "flight_speed": 0.0, "flight_altitude": 0.0, "interval": 0.0, "flight_mode": 0, "nozzle": 0}
Raw message: {"timestamp": "2023-10-04 14:50:48.343934", "wind-speed": 0, "wind-direction": 0, "flight_speed": 0.0, "flight_altitude": 0.0, "interval": 0.0, "flight_mode": 0, "nozzle": 0}
Raw message: {"timestamp": "2023-10-04 14:50:49.639774", "wind-speed": 0, "wind-direction": 0, "flight_speed": 0.0, "flight_altitude": 0.0, "interval": 0.0, "flight_mode": 0, "nozzle": 0}
Raw message: {"timestamp": "2023-10-04 14:50:50.935499", "wind-speed": 0, "wind-direction": 0, "flight_speed": 0.0, "flight_altitude": 0.0, "interval": 0.0, "flight_mode": 0, "nozzle": 0}
Raw message: {"timestamp": "2023-10-04 14:50:52.231330", "wind-speed": 0, "wind-direction": 0, "flight_speed": 0.0, "flight_altitude": 0.0, "interval": 0.0, "flight_mode": 0, "nozzle": 0}
Raw message: {"timestamp": "2023-10-04 14:50:53.527106", "wind-speed": 0, "wind-direction": 0, "flight_speed": 0.0, "flight_altitude": 0.0, "interval": 0.0, "flight_mode": 0, "nozzle": 0}
Raw message: {"timestamp": "2023-10-04 14:50:54.822941", "
Raw message: -speed": 0, "wind-direction": 0, "flight_speed": 0.0, "flight_altitude": 0.0, "interval": 0.0, "flight_mode": 0, "nozzle": 0}
Raw message: {"timestamp": "2023-10-04 14:50:56.118695", "wind-speed": 0, "wind-direction": 0, "flight_speed": 0.0, "flight_altitude": 0.0, "interval": 0.0, "flight_mode": 0, "nozzle": 0}
Raw message: {"timestamp": "2023-10-04 14:50:57.414415", "wind-speed": 0, "wind-direction": 0, "flight_speed": 0.0, "flight_altitude": 0.0, "interval": 0.0, "flight_mode": 0, "nozzle": 0}
Raw message: {"timestamp": "2023-10-04 14:50:58.710177", "wind-speed": 0, "wind-direction": 0, "flight_speed": 0.0, "flight_altitude": 0.0, "interval": 0.0, "flight_mode": 
Raw message: nozzle": 0}
Raw message: {"timestamp": "2023-10-04 14:51:00.005798", "wind-speed": 0, "wind-direction": 0, "flight_speed": 0.0, "flight_altitude": 0.0, "interval": 0.0, "flight_mode": 0, "nozzle": 0}
Raw message: {"timestamp": "2023-10-04 14:51:01.302678", "wind-speed": 0, "wind-direction": 0, "flight_speed": 0.0, "flight_altitude": 0.0, "interval": 0.0, "flight_mode": 0, "nozzle": 0}
Raw message: {"timestamp": "2023-10-04 14:51:02.598585", "wind-speed": 0, "wind-direction": 0, "flight_speed": 0.0, "flight_altitude": 0.0, "interval": 0.0, "flight_mode": 0, "nozzle": 0}
Raw message: {"timestamp": "2023-10-04 14:51:03.894304", "wind-speed": 0, "wind-direction": 0, "flight_speed": 0.0, "flight_altitude": 0.0, "interval": 0.0, "flight_mode": 0, "nozzle": 0}
Raw message: {"timestamp": "2023-10-04 14:51:05.190057", "wind-speed": 0, "wind-direction": 0, "flight_speed": 0.0, "flight_altitude": 0.0, "interval": 0.0, "flight_mode": 0, "nozzle": 0}
Raw message: {"timestamp": "2023-10-04 14:51:06.485748", "wind-speed": 0, "wind-direction": 0, "flight_speed": 0.0, "flight_altitude": 0.0, "interval": 0.0, "flight_mode": 0, "nozzle": 0}
```

Each line represent a complete transmission & reception cycle. 
Node R did receive the data, but you can notice that the data doesn't seem to be consistent.
We got a full payload for the majority of the case, 
but sometimes either the payload is missing its front or its end part.

We failed at preserving the integrity of the data.

## Debugging the payload integrity issue

That moment, I didn't really understand how this reception logic works. 
I thought the bug is due to some packet loss, 
or that maybe we need to send & receive the data in a perfectly synchronized manner, 
which is impossible as far as my head goes.

Maybe when Node T start transmitting, Node R wasn't ready to receive the data so some of the payload get ignored?
Or maybe the payload was forcefully cut in the middle because Node R receive another payload in the middle of the reception?

So much confusion - so much curiosity 😊

Our first approach was to try delaying the Node R interval to 10 second. 
Basically this means that Node R will only process the reception every 10 second, 
while Node T will keep sending the payload every 1 second.

The resulted log is as follows:

```
Raw message: {"timestamp": "2023-10-04 15:10:20.546010", "wind-speed": 0, "wind-direction": 0, "flight_speed": 0.0, "flight_altitude": 0.0, "interval": 0.0, "flight_mode": 0, "nozzle": 0}{"timestamp": "2023-10-04 15:10:21.841714", "wind-speed": 0, "wind-direction": 0, "flight_speed": 0.0, "flight_altitude": 0.0, "interval": 0.0, "flight_mode": 0, "nozzle": 0}{"timestamp": "2023-10-04 15:10:23.137467", "wind-speed": 0, "wind-direction": 0, "flight_speed": 0.0, "flight_altitude": 0.0, "interval": 0.0, "flight_mode": 0, "nozzle": 0}{"timestamp": "2023-10-04 15:10:24.432756", "wind-speed": 0, "wind-direction": 0, "flight_speed": 0.0, "flight_altitude": 0.0, "interval": 0.0, "flight_mode": 0, "nozzle": 0}{"timestamp": "2023-10-04 15:10:25.729036", "wind-speed": 0, "wind-direction": 0, "flight_speed": 0.0, "flight_altitude": 0.0, "interval": 0.0, "flight_mode": 0, "nozzle": 0}{"timestamp": "2023-10-04 15:10:27.024815", "wind-speed": 0, "wind-direction": 0, "flight_speed": 0.0, "flight_altitude": 0.0, "interval": 0.0, "flight_mode": 0, "nozzle": 0}{"timestamp": "2023-10-04 15:10:28.320604", "wind-speed": 0, "wind-direction": 0, "flight_speed": 0.0, "flight_altitude": 0.0, "interval": 0.0, "flight_mode": 0, "nozzle": 0}
Raw message: {"timestamp": "2023-10-04 15:10:29.616470", "wind-speed": 0, "wind-direction": 0, "flight_speed": 0.0, "flight_altitude": 0.0, "interval": 0.0, "flight_mode": 0, "nozzle": 0}{"timestamp": "2023-10-04 15:10:30.912124", "wind-speed": 0, "wind-direction": 0, "flight_speed": 0.0, "flight_altitude": 0.0, "interval": 0.0, "flight_mode": 0, "nozzle": 0}{"timestamp": "2023-10-04 15:10:32.207922", "wind-speed": 0, "wind-direction": 0, "flight_speed": 0.0, "flight_altitude": 0.0, "interval": 0.0, "flight_mode": 0, "nozzle": 0}{"timestamp": "2023-10-04 15:10:33.503702", "wind-speed": 0, "wind-direction": 0, "flight_speed": 0.0, "flight_altitude": 0.0, "interval": 0.0, "flight_mode": 0, "nozzle": 0}{"timestamp": "2023-10-04 15:10:34.799492", "wind-speed": 0, "wind-direction": 0, "flight_speed": 0.0, "flight_altitude": 0.0, "interval": 0.0, "flight_mode": 0, "nozzle": 0}{"timestamp": "2023-10-04 15:10:36.095204", "wind-speed": 0, "wind-direction": 0, "flight_speed": 0.0, "flight_altitude": 0.0, "interval": 0.0, "flight_mode": 0, "nozzle": 0}{"timestamp": "2023-10-04 15:10:37.391074", "wind-speed": 0, "wind-direction": 0, "flight_speed": 0.0, "flight_altitude": 0.0, "interval": 0.0, "flight_mode": 0, "nozzle": 0}{"timestamp": "2023-10-04 15:10:38.686835", "wind-speed": 0, "wind-direction": 0, "flight_speed": 0.0, "flight_altitude": 0.0, "interval": 0.0, "flight_mode": 0, "nozzle": 0}
Raw message: {"timestamp": "2023-10-04 15:10:39.982603", "wind-speed": 0, "wind-direction": 0, "flight_speed": 0.0, "flight_altitude": 0.0, "interval": 0.0, "flight_mode": 0, "nozzle": 0}{"timestamp": "2023-10-04 15:10:41.278406", "wind-speed": 0, "wind-direction": 0, "flight_speed": 0.0, "flight_altitude": 0.0, "interval": 0.0, "flight_mode": 0, "nozzle": 0}{"timestamp": "2023-10-04 15:10:42.574083", "wind-speed": 0, "wind-direction": 0, "flight_speed": 0.0, "flight_altitude": 0.0, "interval": 0.0, "flight_mode": 0, "nozzle": 0}{"timestamp": "2023-10-04 15:10:43.869945", "wind-speed": 0, "wind-direction": 0, "flight_speed": 0.0, "flight_altitude": 0.0, "interval": 0.0, "flight_mode": 0, "nozzle": 0}{"timestamp": "2023-10-04 15:10:45.165715", "wind-speed": 0, "wind-direction": 0, "flight_speed": 0.0, "flight_altitude": 0.0, "interval": 0.0, "flight_mode": 0, "nozzle": 0}{"timestamp": "2023-10-04 15:10:46.462418", "wind-speed": 0, "wind-direction": 0, "flight_speed": 0.0, "flight_altitude": 0.0, "interval": 0.0, "flight_mode": 0, "nozzle": 0}{"timestamp": "2023-10-04 15:10:47.758269", "wind-speed": 0, "wind-direction": 0, "flight_speed": 0.0, "flight_altitude": 0.0, "interval": 0.0, "flight_mode": 0, "nozzle": 0}
```

The message did not break anymore! But now we have 7 or 8 payloads on a single reception process :")

We realized the problem was probably not about how the LoRa fails to reliably send data, 
but rather how we process the reception.

My analytical guess is that Node R will keep appending any received payload to `self.node.serial.in_waiting`
variable/register when any valid payload arrived.
The variable will keep being filled with those data either until it was full or it was manipulated by us
(being read or flushed, for example).
How we want the retrieval process behave is maybe also up to us.

We don't seem to know when a single payload end, as we don't have any separator/delimiter on the payload.
An idea came to my head - to process each JSON with regex as we may be able to use the `}{` pattern.
But after careful consideration, 
I lean towards not using it as the effort was too big compared to how the fragile the result will be,
at least per my experiences doing data cleaning with regex.

We spent quite some time googling and reading the doc, in the edge of giving up myself,
until randomly I found this function inside `serial/serialutils.py`:

| ![](/assets/2023-10-05/read_until.png) |
|:--------------------------------------:|
|           *Will this work?*            |

The function seem to behave like what we wanted: to wait/block the program flow until certain character appear
(in this case, the default `\n`).

So in Node R we changed `read()` with `read_until()`:

```python3
def receive_message(self) -> str:
    if self.node.serial.inWaiting() > 0:
        # receive_buffer = self.node.serial.read(self.node.serial.inWaiting())
        receive_buffer = self.node.ser.read_until()  # < -----------
        sender_address = (receive_buffer[0] << 8) + receive_buffer[1]
        sender_frequency = receive_buffer[2] + self.node.start_freq
        byte_message = receive_buffer[3:-1]
        string_message = byte_message.decode()
        return string_message
```

Consequently, we will also need to add `\n` at the end of our payload inside Node T's `send_message()`, 
so Node R would understand where each payload ends:

```
# append \n to indicate end of the message
message = message + '\n'
```

And voilà! The received payload seems to be whole and consistence even after some time of running!

```
Raw message: {"timestamp": "2023-10-04 15:38:12.391509", "wind-speed": 0, "wind-direction": 0, "flight_speed": 0.0, "flight_altitude": 0.0, "interval": 0.0, "flight_mode": 0, "nozzle": 0}
Raw message: {"timestamp": "2023-10-04 15:38:13.674248", "wind-speed": 0, "wind-direction": 0, "flight_speed": 0.0, "flight_altitude": 0.0, "interval": 0.0, "flight_mode": 0, "nozzle": 0}
Raw message: {"timestamp": "2023-10-04 15:38:14.970536", "wind-speed": 0, "wind-direction": 0, "flight_speed": 0.0, "flight_altitude": 0.0, "interval": 0.0, "flight_mode": 0, "nozzle": 0}
Raw message: {"timestamp": "2023-10-04 15:38:16.266160", "wind-speed": 0, "wind-direction": 0, "flight_speed": 0.0, "flight_altitude": 0.0, "interval": 0.0, "flight_mode": 0, "nozzle": 0}
Raw message: {"timestamp": "2023-10-04 15:38:17.562026", "wind-speed": 0, "wind-direction": 0, "flight_speed": 0.0, "flight_altitude": 0.0, "interval": 0.0, "flight_mode": 0, "nozzle": 0}
Raw message: {"timestamp": "2023-10-04 15:38:18.857779", "wind-speed": 0, "wind-direction": 0, "flight_speed": 0.0, "flight_altitude": 0.0, "interval": 0.0, "flight_mode": 0, "nozzle": 0}
Raw message: {"timestamp": "2023-10-04 15:38:20.153427", "wind-speed": 0, "wind-direction": 0, "flight_speed": 0.0, "flight_altitude": 0.0, "interval": 0.0, "flight_mode": 0, "nozzle": 0}
Raw message: {"timestamp": "2023-10-04 15:38:21.449225", "wind-speed": 0, "wind-direction": 0, "flight_speed": 0.0, "flight_altitude": 0.0, "interval": 0.0, "flight_mode": 0, "nozzle": 0}
Raw message: {"timestamp": "2023-10-04 15:38:22.744997", "wind-speed": 0, "wind-direction": 0, "flight_speed": 0.0, "flight_altitude": 0.0, "interval": 0.0, "flight_mode": 0, "nozzle": 0}
Raw message: {"timestamp": "2023-10-04 15:38:24.040912", "wind-speed": 0, "wind-direction": 0, "flight_speed": 0.0, "flight_altitude": 0.0, "interval": 0.0, "flight_mode": 0, "nozzle": 0}
Raw message: {"timestamp": "2023-10-04 15:38:25.336557", "wind-speed": 0, "wind-direction": 0, "flight_speed": 0.0, "flight_altitude": 0.0, "interval": 0.0, "flight_mode": 0, "nozzle": 0}
```

I'm super happy with the result! 

If you doesn't get how significant this is, 
imagine you are tasked to "send 10kgs of oranges from city A to B with a truck",
up until now this article only talks about the truck, because how can we send the oranges if we didn't have 
a reliable truck in the first place?

The truck - or the payload transmission system - is the backbone of this entire project.

By having this settled, we have fixed more than 80% of the problems!

## Decoding the raw message

Remember that what we've got is a raw string message. Naturally, we want to deserialize this JSON-formatted string
back to python object so data retrieval can be done with ease in the following steps later.

But with `json.loads(string_message)`, the decode process failed!

```
Raw message: {"timestamp": "2023-10-04 16:25:24.484442", "wind-speed": 0, "wind-direction": 85, "flight_speed": 0.0, "flight_altitude": 0.0, "interval": 0.0, "flight_mode": 0, "nozzle": 0}
Attempting to do json.loads() on: "{"timestamp": "2023-10-04 16:25:24.484442", "wind-speed": 0, "wind-direction": 85, "flight_speed": 0.0, "flight_altitude": 0.0, "interval": 0.0, "flight_mode": 0, "nozzle": 0}"
Failed: Expecting value: line 1 column 1 (char 0)
Traceback (most recent call last):
  File "/home/pi/Sys/rover_upgrade/library.py", line 316, in _init_
    self.json_message = json.loads(message)
  File "/usr/lib/python3.7/json/_init_.py", line 348, in loads
    return _default_decoder.decode(s)
  File "/usr/lib/python3.7/json/decoder.py", line 337, in decode
    obj, end = self.raw_decode(s, idx=_w(s, 0).end())
  File "/usr/lib/python3.7/json/decoder.py", line 355, in raw_decode
    raise JSONDecodeError("Expecting value", s, err.value) from None
json.decoder.JSONDecodeError: Expecting value: line 1 column 1 (char 0)

Raw message: {"timestamp": "2023-10-04 16:25:25.768733", "wind-speed": 0, "wind-direction": 85, "flight_speed": 0.0, "flight_altitude": 0.0, "interval": 0.0, "flight_mode": 0, "nozzle": 0}
Attempting to do json.loads() on: "{"timestamp": "2023-10-04 16:25:25.768733", "wind-speed": 0, "wind-direction": 85, "flight_speed": 0.0, "flight_altitude": 0.0, "interval": 0.0, "flight_mode": 0, "nozzle": 0}"
Failed: Expecting value: line 1 column 1 (char 0)
Traceback (most recent call last):
  File "/home/pi/Sys/rover_upgrade/library.py", line 316, in _init_
    self.json_message = json.loads(message)
  File "/usr/lib/python3.7/json/_init_.py", line 348, in loads
    return _default_decoder.decode(s)
  File "/usr/lib/python3.7/json/decoder.py", line 337, in decode
    obj, end = self.raw_decode(s, idx=_w(s, 0).end())
  File "/usr/lib/python3.7/json/decoder.py", line 355, in raw_decode
    raise JSONDecodeError("Expecting value", s, err.value) from None
json.decoder.JSONDecodeError: Expecting value: line 1 column 1 (char 0)
```

Wait, what?!

I've carefully inspected each character inside those "supposed to be JSON formatted" string, but found no issue.

I've copied the entire json string that printed in the log, 
and reproducing it on my local PC leads to no error.

From my entire web development career and so many REST API tasks, I've never experienced such bug before.

I can't reproduce the error!

<br>
<br>

After so many times of confusion and shrugging, 
it came to my mind to print the `string_message` back to the encoded form.

Then I got this:

```
>>> string_message_1.encode()
b'\x00\x00\x12{"timestamp": "2023-10-04 16:25:24.484442", "wind-speed": 0, "wind-direction": 85, "flight_speed": 0.0, "flight_altitude": 0.0, "interval": 0.0, "flight_mode": 0, "nozzle": 0}'

>>> string_message_2.encode()
b'\x12\x00\x00\x12{"timestamp": "2023-10-04 16:25:25.768733", "wind-speed": 0, "wind-direction": 85, "flight_speed": 0.0, "flight_altitude": 0.0, "interval": 0.0, "flight_mode": 0, "nozzle": 0}'
```

Looks like the `string_message` is (I would say) dirty? Is it prepended by some kind of byte?

Evaluating the transmission code, we are aware that beside the message itself, the entire payload also consist of:
- Node T's address & frequency (source)
- Node R's address & frequency (destination)

Both of them are merged with the actual message with bitwise operation. 
So maybe somehow the bitwise calculation isn't precise?

If we can reliably clean this, will we get a valid JSON?

Coming from a web scrapping experiences during my college time, removing "strange characters" is as easy as:

```
cleaned_message = ''.join([character for character in presumably_dirty_message if character.isprintable()])
```

The JSON deserialization succeed! Wohooo! 😁

|      ![](/assets/2023-10-05/is_json.png)      |
|:---------------------------------------------:|
| *Super happy by noticing the `Is JSON: True`* |

## Conclusion

Those nodes can send & receive JSON-based string reliably! 
The data will be used for future steps - being tested with supplied models. 

The whole project then continued as it should, but are outside the scope of this article.

I've spent several days on this and I can say I'm not regretting any of these experiences.

In this modern day, you can find an army-set of Django & JS developer. 
But low-level programmer? I don't think so.
This project doesn't even count as a low-level programming anyway, but you guys get the point.

I imagine that someday I would sit comfortably on a low-level programming career, 
tinkering with C/C++/Rust on an opensource project and getting paid by sponsorship 🙂

Imagine working every day for something that you really like and enjoy and still getting paid enough to fullfill
all of your commitments 🙂

Someday, Rahmat. Someday. Insya Allah.

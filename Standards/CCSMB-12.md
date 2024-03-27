# CCSMB-12: Coordinate Protocol

*Author: Sammy L. Koch <@1Turtle>*  
*Version: v1.0.0*  
*Last updated: YYYY-MM-DD*  

| Type        | Value                     |
| ----------- | ------------------------- |
| Version     | 1.0.0                     |
| MIME        | `message/cp`              |

## Introduction

**CP** stands for _Coordinate Protocol_ and allows the communication inside a decentralized system where each instance can send packages with the only limitation of telling how far it has been travaled from the sender to the receiver.

### Goal and Scope

This protocol shines when it comes to ensuring a standard where it is possible to address specific instances or ensuring from where a received package comes from. 

It heavily leans on the usage of so called **CP Addresses** in order to determine the ownership of packages and therefore should be ideally only used whenever at least one instance is expected to be on a static location. See the following section for further details on what defines a CP Address and why it is important having a static instance.

Having a way of using [trilateration](https://en.wikipedia.org/wiki/Trilateration) through a setup of four wireless modems on the instance allows to check the signature of every received package. See [this example section](#filtering-packages-by-their-signature) for detailed instructions.

## CP Addresses

A **CP Address** is the equivalent to an _IP_ or _MAC Address_. Every instance exists somewhere and therefore can be described using coordinates. _(This standard will expect 3D Coordinates in the format of X, Y and Z from here on. The protocol could be easily adjusted to more or less dimensions however.)_

### Formatation

A CP Address is made upon all the values of the coordinates within one string without gaps but the numbers prefixes as separation. Coordinates do not have to be absolute or rounded, as long as they are static, they are valid.

In case the coordinates may not be static but dynamicly by for instance having a Pocket Computer or Turtle moving, a so called [handshake](https://en.wikipedia.org/wiki/Handshake_(computing)) should be made first.

For the Coordinates `-123`, `+456` and `+789`, the corresponding CP Address would be **`-123+456+789`**. Another example would be `+47472.13`, `-56` and `-1748.967`, where the CP Address then would be **`+47472.13-56-1748.967`**.

## Package

A package following the CP comes as a string formatted as an [JSON](https://ecma-international.org/publications-and-standards/standards/ecma-404/) object and always contains the key `header` which contains another object. The header always needs the key `"type"` within the MIME value `"message/cp"` and also the key `"receiver"` with it's desired destination. Besides the header, there also is the key `"body"`, which can be anything. It contains the actual message of the pacakge.

A package could look like this:

```json
{
    "header": {
        "type": "message/cp", // Always needs to be this string in order to be recognizable.
        "receiver": "-123+456+789" // Can be either a CP Address or the string "broadcast".
    },
    "body": null // Can be anything.
}
```

### Signatures

The package does not contain a signature of where it came from. Since this protocol is expecting that every package automatically comes with how far it has been traveled, it does not contain a key for this metadata. Instead, the receiver can use [trilateration](https://en.wikipedia.org/wiki/Trilateration) to determine the exact position and this way have a _signed package_.

## Example

The following are some potential solutions for various scenarios in CC: Tweaked, written for Lua 5.2. The following examples can be used within a Computer that comes with an _(Ender)_ Modem inside an environment where using GPS also works.

### Broadcasting a package

This code broadcasts the message `"Hello World"` to everyone in reach via the port `80` and takes respones over the port `81`:

```lua
local modem = peripheral.find("modem")
if not modem or not modem.isWireless() then
    error("Wireless modem requried")
end

local port = 80

local package = {
    header = {
        type = "message/cp",
        receiver = "broadcast"
    },
    body = "Hello World"
}

-- 
package = textutils.serialiseJSON(package)

modem.transmit(80, 81, package)
```

### Receiving anonymous packages

This code waits for packages on port `80` that are either meant for this Computer _(through our CP Address)_ or that have been broadcasted to everyone:

```lua
local modem = peripheral.find("modem")
if not modem or not modem.isWireless() then
    error("Wireless modem requried")
end

local port = 80

local x, y, z = gps.locate(3)
if not x then
    error("No GPS setup")
end

-- Convert to CP Address
local cp =  ("%s%d%s%d%s%d"):format(
    (x >= 0 and "+" or "-"), x,
    (y >= 0 and "+" or "-"), y,
    (z >= 0 and "+" or "-"), z,)

-- Start listening
modem.open(port)
while true do
    local _, _, channel, replyChannel, message, distance = os.pullEvent("modem_message")

    local package = textutils.unserializeJSON(message)
    if package then
        -- Package is JSON object.

        -- Does it contain a header key?
        local header = package.header
        if not header then
            goto continue
        end

        -- Is the protocol known?
        if header.type ~= "message/cp" then
            goto continue
        end

        -- Is package for us?
        if header.receiver == cp or header.receiver == "broadcast" then
            -- Yes it is!
            print(("Package received on channel %d (reply %d) with the message %s"):format(
                channel, replyChannel, "'"..tostring(package.body).."'" or "nil"
            ))
        end
    end

    ::continue::
end
```

> Notice how the example only uses one modem and how the `distance` variable is not being used to determine from where a package came from. This code example cannot guarantee that a package came from the same instance without further setups outside of our scope. In order to check for that, look at the next example.

### Filtering packages by their signature

This example requires a more complex setup. The previous ones only required one modem and so far did not ensure from where a received package comes from. Here, a packages signature can be checked to proof from where it originally came from.

First, ensure that the reiceiving instance has four Wireless Modems, placed in a way where [trilateration]() can be used. The [GPS constellation from the tweaked.cc wiki](https://tweaked.cc/guide/gps_setup.html) will be used in our case. Those modems will be connected to a single Computer using Wired Modems, instead of four like shown in the image of the wiki.

With our setup being ready, the modems roles need to be defined first in our code:

```lua
local m = {
    ["modem_0"] = {
        pos = vector.new(123, 456, 789)
        p = peripheral.wrap("modem_0")
    },
    ["modem_1"] = {
        pos = vector.new(789, 123, 456)
        p = peripheral.wrap("modem_1")
    },
    ["modem_2"] = {
        pos = vector.new(456, 789, 123)
        p = peripheral.wrap("modem_2")
    },
    ["modem_3"] = {
        pos = vector.new(987, 654, 321)
        p = peripheral.wrap("modem_3")
    },
}

-- ...
```

> Note: the strings `"modem_0"` to `"modem_3"` are going to be used here with some example coordinates. The strigs and coordinates of course are different for each setup. The strings for the modems also should be put into constants first, as those are going to be used multiple times.

The ports now need to be open up for all the modems and start listen for the packages again:

```lua
local port = 80

m["modem_0"].p.open(port)
m["modem_1"].p.open(port)
m["modem_2"].p.open(port)
m["modem_3"].p.open(port)

local packages = {}

while true do
    local _, side, channel, replyChannel, message, distance = os.pullEvent("modem_message")

    -- ...

    ::continue::
end
```

The Computer will now receive one package four times, because each modem will receive it. Because of this, we can determine a packages exact position. Before this can be calculated however, it first needs to be stored after the corresponding modem. The packages are going to be put in the corresponding variable `packages` with the needed information as its value:

```lua
if channel ~= port then
    goto continue
end

-- Initialize package if not already done
if type(packages[message]) ~= "table" then
    packages[message] = {
        pos1 = nil,
        pos2 = nil,
        distances = {}
    }
end

-- Store the distance from the sender to the modem of the receiver
table.insert(packages[message].distances, {modem = side, distance = distance })

-- ...
```

With this out of the way, the calculation can now be done for each package in the table that has been received four times:

```lua
local distances = packages[message].distances
if #distances>= 3 then
    -- Use trilateration to determine CP Address
    -- This may run multiple times until it got the exact position

    -- Convert data to something we can pass to trilaterate
    local data = {}
    for i=1, 3 do
        local offset = (#distances > 3) and 1 or 0

        local distance = distances[i+offset]
        table.insert(data,
            { vPosition = m[distance.modem].pos, nDistance = distance.distance }
        )
    end

    -- Actual calculation
    if not pos1 then
        pos1, pos2 = trilaterate(distances[1], distances[2], distances[3])
    else
        pos1, pos2 = narrow(pos1, pos2, distances[3])
    end

    -- Store results
    packages[message].pos1 = pos1
    packages[message].pos2 = pos2

    if pos1 and not pos2 then
        -- We now can get the CP Address
        local cp =  ("%s%d%s%d%s%d"):format(
            (x >= 0 and "+" or "-"), x,
            (y >= 0 and "+" or "-"), y,
            (z >= 0 and "+" or "-"), z,)

        -- Handle package however we need it
        print(("Received package from %s with the message: %s"):format(
            cp, message
        ))

        packages[message] = nil -- Remove from memory, we are done
    end
end
```
> Note: The used implementations for the functions `trilaterate` and `narrow` are from the [GPS program](https://github.com/cc-tweaked/CC-Tweaked/blob/mc-1.20.x/projects/core/src/main/resources/data/computercraft/lua/rom/apis/gps.lua) in CC: Tweaked.

Finally, we now could determine the CP Address of an received package. This example may not be efficient with how it handles the memory for the packages and may also do a bad job in cleaning up its packages. It should however show how to determine a CP Address in it's core element.

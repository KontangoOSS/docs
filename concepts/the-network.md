# The Network

## What the network is for

The Kontango network exists so that the equipment you own — wherever it is — can act like one cloud. A Raspberry Pi in your living room, a desktop in your home office, a VPS in a data center on another continent, a laptop in a hotel — all of them can be on the same private network, reach the same internal services, and trust each other, without any of those services being exposed to the public internet.

In other words: the network is the part that lets your hardware live anywhere and still work together as if it were in one room.

## A picture you can hold in your head

Imagine a building. Most people walk in through the front lobby — that's the public internet. Anyone can stand outside, knock on the door, peek through the windows, try the handles.

Kontango is a **private hallway** running through the same building. It has its own door, the door is locked, and only people with a specific kind of key can get in. The hallway connects to several rooms — the git room, the secrets room, the admin room — and each room has its own lock too.

When your device enrolls, it gets the key to the hallway. Whether each individual room opens for you depends on what your key is allowed to do.

The public internet still works exactly the same way. You can browse Wikipedia, watch YouTube, send email — all of that goes out the front door of the building. The Kontango hallway is *additional*, not a replacement.

## Hardware freedom is the point

The network doesn't care where your devices are. It doesn't care what they are. A $35 Pi, a $5,000 server, a Mac mini under your TV, a free-tier cloud VM — once they're enrolled, they're on the network the same way. This is on purpose. The whole point is that **you choose your hardware**, and the network adapts to it instead of the other way around.

That has practical consequences:

- You can move a service from one machine to another without changing how anyone reaches it
- You can host things across cheap hardware you already own, instead of paying for cloud compute
- You can mix on-premises and cloud freely — there's no separate "VPN to home" or "VPN to office"
- If a piece of hardware dies, you replace it with whatever you have lying around

None of this is a stunt. It's the boring practical answer to "I have some hardware, I have some workloads, I'd like the workloads to run on the hardware without an ops team."

## What it is NOT

**It is not a VPN.** A VPN takes all your traffic and routes it through one tunnel — typically slow, all-or-nothing, and it makes your public IP look like it's somewhere else. Kontango leaves your normal internet alone. Only traffic to internal `.tango` addresses goes through Kontango. Everything else takes its usual path.

**It is not just a wifi network.** Wifi connects your device to a router; that's a physical link. Kontango is a logical thing — your laptop can be on hotel wifi, on your phone's tethered connection, or on the office network, and Kontango works the same way in all three. It doesn't care where your device is.

**It is not security through obscurity.** People sometimes hide things on weird ports and hope no one finds them. That's not Kontango. Internal services on Kontango have *no* public address at all — there's nothing to find, nothing to scan, nothing to attack. The only way in is through enrollment, and enrollment requires a certificate that's tied to a specific device.

## What changes for you when you're on Kontango

| | Without Kontango | With Kontango |
|---|---|---|
| Public internet | Works | Works the same |
| `git.tango` (internal git) | Doesn't resolve, no such address | Loads, and you're already logged in by certificate |
| SSH to a server | Need its IP, a VPN, or a jump host | `ssh your-host` just works |
| Speed | — | Same as the underlying network — no detour |
| What others can see | The public internet, your IP | Same; Kontango doesn't broadcast anything |

## The `.tango` thing

Internal services have addresses that end in `.tango` — like `git.tango`, `bao.tango`, `proxmox.tango`. That's not a real internet domain. You can't type `git.tango` from a coffee shop laptop and reach anything; it just won't resolve.

`.tango` is a private naming scheme that only works on enrolled devices. When your device sees `.tango`, it knows: "this isn't a public address — route this through the Kontango hallway." When the agent on your device isn't running, those addresses go nowhere. That's the point.

It's like an internal extension number at a company. Dial `7423` from inside the office and you reach the right desk; dial it from outside and the call goes nowhere.

## Architecture in one paragraph

A Kontango deployment is typically a 3-node high-availability cluster on hardware you control. Even if one node goes down, the network keeps working. The cluster is the part that knows who's enrolled, what services exist, and which devices are allowed to reach which services. Everything is encrypted end-to-end — even the cluster itself can't see the contents of your traffic.

## Where to next

- [Enrolling a device](enrollment.md) — how a device gets the key
- [Services](services.md) — what's in the rooms
- [Policy](policy.md) — who's allowed in which room

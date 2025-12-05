
(New drawing)

There are a few fundamental components to this project:
### Jitsi Meet

Jitsi itself is an open-source project containing several repositories. The only real important one for the sake of this project is Jitsi Meet. This is the main video conferencing application that has been extremely well-maintained and well-documented. It comes in a few forms via their docs, including downloading as a complete docker file that Just Works™. However, you probably don't want that if, like me, you don't really understand Docker and its limitations.

My advice: before even setting up a Jitsi server, have a purchased domain ready from a major provider like Cloudflare. Also, don't try to run peer-to-peer or locally. I managed to run my own Jitsi server from my own machine, but it requires access to your  router so that you can enable port-forwarding ports. I probably spent two weekends trying to use a relay to access my server via STUN/TURN servers. Just skip all that, get AWS free tier, and install it on an EC2.

Holy crap, the handbook actually has [[that exact advice]]. Too bad I didn't listen. But you can, and you should.

### NICE DCV

Amazon DCV (previously NICE DCV) is a remote desktop protocol (RDP). RDP lets you take remote control of someone else's computer, while having their screen streamed directly to you. The main app for doing this is literally just called Windows RDP. 

So why did Amazon acquire DCV for a bunch of money? Here are a few things that are special about DCV:

1) Several users can remote in at the same time, Windows RDP only allows one session
2) Supported on the three main operating systems (MacOS, Linux, Windows)
3) DCV supports 4K and 60 FPS, and will compress the image quality in order to maintain low latency
4) DCV can resize the remote session based on the size of the end user's client

All these things make DCV an interesting choice for companies that have expensive, highly customized workstations because they can simply remote into that machine. It also is good for gaming and is the remote desktop protocol that supports PLAICraft.

**Does it Just Work™ ?**

Like, almost. DCV itself is incredibly stable, but just be warned that if you don't have a valid SSL, you're SOL. Both Jitsi Meet and DCV use TCP/UDP for video streaming (rather than HTTP), and anything Websocket, WebRTC or UDP-based will basically reject a connection unless both sides have valid SSL certificates. SSL certificates are very easily generated from a domain provider like Cloudflare when you own a domain. Also, keep in mind that DCV is closed-source, and early on I tried to figure out how to separate inputs between multiple users in DCV, but I barely made a scratch.

### Open World Agents

PLAICraft, to capture multimodal training data, has a bespoke setup consisting of [[OBS]], a Minecraft mod package called Spigot, and then whatever middleware they used to pack it all in together. Open World Agents is trying to generalize that process for all desktop recording, which makes a ton of sense. It doesn't matter if the user interface is Minecraft, Excel or trading Memecoins; the inputs of screen, audio, mouse and keyboard ought to be the same. Actually, another win for DCV: it allows I/O devices, so bring your USB steering wheel.

The key component for this project from Open World Agents is called [OCAP](https://github.com/open-world-agents/ocap). OCAP captures full desktop interactions, and then writes it to two files: MCAP and MKV. [MCAP](https://mcap.dev/) is an open-source standard for time-aligned multimodal input files.  MKV is just a standard video file type.

OCAP mainly relies on an impossibly large library called Gstreamer, which is basically just a ton of multimedia encoders. You can generally install from Gstreamer the same way that people install from a package manager like brew or pip. Open World Agents and OCAP use Anaconda, which as far as I know is a virtual environment and path manager, not unlike a python virtual environment. The main way you call it is via "conda" and you'll know that you have conda when you see (base) in your terminal.

**Does OCAP Just Work™?**

Well... I give them 7.5 / 10. I must accept responsibility as a developer who reads docs too fast (or not at all). But their docs are quite scattered, and they recently broke off OCAP into its own repository which confused me. The biggest issues are critical limitations with OCAP at the time of writing: it's Windows-only, and requires an NVIDIA GPU. This means that you have to request from your cloud provider access to NVIDIA GPUs (I requested 8 vCPUs from AWS, which corresponded to two g4dn.xlarge instances at a time), spend about 5x more on compute, AND you need to install the correct NVIDIA drivers on your virtual machine, and probably increase the storage while you're at it.

I plan on making an open-source contribution to their project, because near the end of my directed study I was able to replace the NVIDIA encoder with a more standard x264enc software encoder. I hope that I can save you some grief. In the event that I was abducted or something, it's not hard. The encoder is hardcoded within the project and OCAP is open-source.

The last thing on OCAP for this section: I found that the commands in the docs straight up did not work. Here's what you should do: download the project onto your VM via GitHub, then in Powershell, run this command. This basically installs all the dependencies for the project as far as I know.

```python
pip install -e .
```

### Windows

This project will basically turn you into a system administrator. I used Windows Server 2022, which you should understand is not a full-feature Windows. There is no Microsoft Store. There are no AI features. Think of it this way: as an end user who has no computer science background, it's Windows Lite. But for someone looking to experiment with the OS, it should be everything you need.

The most important part of my workflow on Windows was having a user called Administrator, and then a user called KioskUser. Administrator was where I wrote code, set up Task Scheduler, messed with registries etc. while KioskUser was the end user.

"Kiosk" refers to an umbrella term for single-app sessions in Windows. The burger menu screen at your local Wendy's? That's a kiosk. Normally, when a user logs into their desktop on Windows, it runs an executable called explorer.exe, which gives them the desktop GUI. If you replace that with your own executable, like burger.exe, congratulations, you just made a kiosk. The only thing the user has access to (not entirely true, see AppLocker below) is your app. If they Alt + F4 and kill that process, they will only get a black screen.

The main way that you configure a kiosk (other than creating a brand new user, which is straightforward) is via regedit, or Registry Edit. These are low-level instructions that the OS reads when booting up and when loading specific user profiles. Do not go poking around here without a purpose because you could brick your whole virtual machine.

You'll also get familiar with Task Scheduler, which is a pretty awesome automation application that allows you to basically run any command on a certain trigger. There are pre-written triggers (e.g. at log on), but you can create custom triggers too (I never needed to).

The last thing I'll mention now is AppLocker. AppLocker basically denies access to certain paths on a desktop, even ones that a user has access to by default. For example, you probably don't want your KioskUser accessing Task Manager, and you can lock that. It is NOT a keyboard logger, so users can still press Ctrl + Alt + Delete and get to the session menu that contains Task Manager.
### Amazon Web Services

AWS was the cloud provider that I used, and from this point forward, I will not be using brand-agnostic terms. With that in mind, here a few of the services and their jargon that you should know about:

EC2: These are amazon's virtual machines. You pick the OS, the hardware, the storage, the security groups, the IAM roles. It's a computer in the cloud.

AMI: If you take a snapshot of an EC2, it's like Amazon takes every byte exactly where it is at a given time and freezes it into an AMI. It is super easy to make one and then create new EC2's from it. Think of it as a saved file that you can clone infinitely.

IAM: These are permissions in AWS. Wanna write to a database? You need IAM permissions for that. Wanna read a database? There's a SEPARATE IAM permission for that. If any services will interact with each other (and they will a whole damn lot), they need IAM permissions. Permissions are inside a Policy which is inside a Role.

S3: Basically Google Drive. You can write to it via a library called boto3, which is basically AWS tools. that lets you interact with AWS outside of AWS. The basic unit of S3 is a bucket.

DynamoDB: A database service. It's NoSQL, meaning that you don't interact with it via a query language like SQL. In my opinion, this just makes it easier to view / interact with than some of AWS's SQL solutions like RDS.

Lambda: Lambdas are sick. They're basically serverless and stateless chunks of code that you can trigger in various ways (e.g. EventBridge). If you want to upload a new hot dog recipe to your S3 and your DynamoDB, one lambda can do both. They're like spaghetti strands that make sure information flows through your application.

API Gateway: This is how I called Lambdas from the frontend. You set up an endpoint that your app can hit like "https://aws.gateway123/join/schedule" and when you hit it, you decide what it does. I had it invoke lambdas with whatever payload my frontend sent.

Note that if I was hosting my domain on AWS, I would probably use something like Route 53, but I used Cloudflare. I'll get to that later.








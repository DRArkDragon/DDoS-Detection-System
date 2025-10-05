# DDoS-Detection-System
This project represents my Major Project as part of the Master of Computer Applications (MCA) program at Jagan Institute of Management Studies, affiliated with Guru Gobind Singh Indraprastha University (GGSIPU). The project was collaboratively developed by a team of three students, including myself.

## Abstract 
This project focuses on detecting Distributed Denial of Service (DDoS) attacks within Software Defined Network (SDN) environments. Using network emulation tools such as Mininet and monitoring solutions including Telegraf and InfluxDB, the project simulates network traffic to identify abnormal patterns indicative of DDoS attacks. The system aims to enhance network security by providing real-time detection and visualization capabilities, aiding in timely mitigation and network stability.


<br>

---

## Installation methods :wrench:

We have created a **Vagrantfile** through which we provide each machine with the necessary scripts to install and configure the scenario. By working in a virtualized environment we make sure we all have the exact same configuration so that tracing and fixing erros becomes much easier. If you do not want to use Vagrant as a provider you can follow the native installation method we present below.

### Vagrant
First of all, clone the repository from GitHub :octocat: and navigate into the new directory with:
```bash
git clone https://github.com/DRArkDragon/DDoS-Detection-System
cd DDoS-Detection-System
```
We power up the virtual machine through **Vagrant**:
```bash
vagrant up
```

And we have to connect to both machines. **Vagrant** provides a wrapper for the *SSH* utility that makes it a breeze to get into each virtual machine. The syntax is just `vagrant ssh <machine_name>` where the `<machine_name>` is given in the **Vagrantfile** :
```bash
vagrant ssh test
vagrant ssh controller
```

We should already have all the machines configured with all the necessary tools to bring our network up with Mininet on the **test** VM, and Ryu on the **controller** VM. This includes every `python3` dependency as well as any needed packages.

#### Troubleshooting problems regarding SSH
If you have problems connecting via SSH to the machine, check that the keys in the path `.vagrant/machines/test/virtualbox/` are owned by the user, and have read-only permissions for the owner of the key. 

``` bash
cd .vagrant/machines/test/virtualbox/
chmod 400 private_key

# We could also use this instead of "chmod 400" (u,g,o -> user, group, others)
# chmod u=r,go= private_key
```
Instead of using vagrant's manager to make the SSH connection, we can opt for manually doing it ourselves by passing the path to the private key to SSH. For example:

```bash
ssh -i .vagrant/machines/test/virtualbox/private_key vagrant@10.0.123.2
```


## OR we can use XTERM if you get any error using Vagrant.
### System Installation and Usage Procedure Using Xterm Terminal
The system runs on Ubuntu VMs created with VMware VirtualBox. Each VM requires manual access via terminal (Xterm) to start components, rather than automated tools like Vagrant. This ensures consistent configuration for easier debugging and control.


---

### Native
This method assumes you already have any VMs up and running with the correct configuration and dependencies installed. Ideally you should have 2 VMs. We will be running **Ryu** (the *SDN* controller) in one of them and we will have **mininet**'s emulated network with running in the other one. Try to use Ubuntu 16.04 (a.k.a **Xenial**) as the VM's distribution to avoid any mistakes we may have not encountered.

First of all clone the repository, and then navigate into it:
```bash
git clone https://github.com/DRArkDragon/DDoS-Detection-System
cd DDoS-Detection-System
```

Manually launch the provisioning scripts in each machine:
```bash
# To install Mininet, Mininet's dependencies and telegraf. Run it on the "mininet" VM
sudo ./util/install_mininet.sh
sudo ./util/install_telegraf.sh

# To install Ryu and Monitoring system (Grafana + InfluxDB). Run it on the "controller" VM
sudo ./util/install_ryu.sh
sudo ./util/install_grafana_influxdb.sh
```

---

## Our scenario
Our network scenario is described in the following script: [`src/scenario_basic.py`](https://github.com/GAR-Project/project/blob/master/src/scenario_basic.py). Mininet makes use of a Python API to give users the ability to automate processes easily, or to develop certain modules at their convenience. For this and many other reasons, Mininet is a highly flexible and powerful tool for network emulation which is widely used by the scientific community. 

* For more information about the API, see its [manual](http://mininet.org/api/annotated.html).


<!--![Escenario](https://i.imgur.com/kH7kAqB.png)-->
<!-- Using HTML let's us center images! It's kind of dirty though... -->
<p align="center">
    <img src="https://i.imgur.com/kH7kAqB.png">
</p>

The image above presents us with the *logic* scenario we will be working with. As with many other areas in networking this logic picture doesn't correspond with the real implementation we are using. We have seen throughout the installation procedure how we are always talking about 2 VMs. If you read carefully you'll see that one VM's "names" are **controller** and **mininet**. So it should come as no surprise that the controller and the network itself are living in different machines!

The first question that may arise is how on Earth can we logically join these 2 together. When working with virtualized enviroments we will generate a virtual LAN where each VM is able to communicate with one another. Once we stop thinking about programs and abstract the idea of "*process*" we find that we can easily identify the **controller** which is just a **ryu** app, which is nothing more than a **python3** app with the **controller**'s VM **IP** address and the port number where the **ryu** is listening. We shouldn't forget that **any** process running within **any** host in the entire **Internet** can be identified with the host's **IP** address and the processes **port** number. Isn't it amazing?

Ok, the above sounds great but... Why should we let the controller live in a machine when we could have everything in a single machine and call it a day? We have our reasons:

* Facilitate teamwork, since the **AI's logic** will go directly into the controller's VM. This let's us increase both working group's independence. One may work on the mininet core and the data collection with **telegraf** whilst the other can look into the DDoS attack detection logic and visualization using **Grafana** and **InfluxDB**. 

* Facilitate the storage of data into **InfluxDB** from **telegraf**, as due to the internal workings of Mininet there may be conflicts in the communication of said data. Mininet's basic operation at a low level is be detailed below.

* Having two different environments relying on distinct tools and implementing different functionalities let's us identify and debug problems way faster. We can know what piece of software is causing problems right away!

### Running the scenario
Running the scenario requires having logged into both VMs manually or using vagrant's SSH wrapper. First of all we're going to power up the controller, to do so we run the following from the `controller` VM. It's an application that does a basic forwarding, which is just what we need:

```
ryu-manager ryu.app.simple_switch_13
```

You might prefer to run the controller in the background as it doesn't provide really meaningful information. In order to do so we'll run:

```
ryu-manager ryu.app.simple_switch_13 > /dev/null 2>&1 &
```

Let's break this big boy down:

* `> /dev/null` redirects the `stdout` file descriptor to a file located in `/dev/null`. This is a "special" file in linux systems that behaves pretty much like a black hole. Anything you write to it just "disappears" :open_mouth:. This way we get rid of all the bloat caused by the network startup.

* `2>&1` will make the `stderr` file descriptor point where the `stdout` file descriptor is currently pointing (`/dev/null`). Terminal emulators usually have both `stdout` and `stderr`"going into" the terminal itself so we need to redirect these two to be sure we won't see any output.

* `&` makes the process run in the background so that you'll be given a new prompt as soon as you run the command.

If you want to move the controller app back into the foreground so that you can kill it with `CTRL + C` you can run `fg` which will bring the last process sent to the background back to the foreground.




<p align="center">
    <img src="img/Image 1.png">
</p>

Once the controller is up we are going to execute the network itself, to do so launch the aforementioned script from the `test` machine:

```
sudo python3 scenario_basic.py
```

<!-- ![mininet_up](https://i.imgur.com/DSPsPDL.png) -->

<p align="center">
    <img src="img/Image 2.png">
</p>

Notice how we have opened **Mininet CLI** from the `test` machine. We can perform many actions from this command line interface. The most useful ones are detailed below.

### Is working properly?
We should have our scenario working as intended by now. We can check our network connectivity by pinging the hosts, for example:

``` bash
mininet> h1 ping h3

# We can also ping each other with the pingall command
mininet> pingall
```

<!-- ![ping_basic](https://i.imgur.com/NhglFK5.png) -->

<p align="center">
    <img src="img/Image 3.png">
</p>

As you can see in the image above, there is full connectivity in our scenario. You may have noticed how the first **ping** takes way longer than the other to get back to use. That is, its **RTT** (**R**ound **T**rip **T**ime) is abnormally high. This is due to the empty **ARP** tables we currently have *AND* to the fact that we don't yet have a flow defined to handle **ICMP** traffic:

* An **ARP** resolution between sender and receiver of the ping takes place so that the sender learns the next hop's **MAC** address.

* In addition, the **ICMP** message (ping-request) will be redirected to the driver (a.k.a controller) to decide what to do with the packet as the switches don't yet have a **flow** to handle this traffic type. This way the controller will, when it receives the packet, instantiate a set of rules on the switches so that the **ICMP** messages are routed from one host to the other.

<!-- ![ryu_packet_in](https://i.imgur.com/lSGDeTN.png) -->

<p align="center">
    <img src="img/Image 4.png">
</p>

<br>

As you can see, the controller's **stdout** indicates the commands it has been instantiating according to the packets it has processed. In the end, for the first packet we will have to tolerate a delay due to **ARP** resolution and **flow** lookup and instantiation within the controller. The good thing is the rest of the packets will already have the destination **MAC** and the rules will already instantiated in the intermediate switches, so the new delay will be minimal.

---

## Attack time! :boom:

<div style="text-align: justify">

We have already talked about how to set up our scenario but we haven't got into breaking things (i.e the fun stuff :smiling_imp:). Our goal is to simulate a **DoS** (**D**enial **o**f **Service**) attack. Note that we usually refer to this kind of threats as **DDoS** attacks where the first **D** stands for **D**istributed. This second "name" implies that we have multiple machines trying to flood our own. We are going to launch the needed amounts of traffic from a single host so we would be making a mistake if we were talking about a distributed attack. All in all this is just a minor nitpick, the concept behind both attacks is exactly the same.

We need to flood the network with traffic, great but... How should we do it? We already introduced the tool we are going to be using: **hping3**. This program was born as a TCP/IP packet assembler and analyzer capable of generating ICMP traffic. Its biggest asset is being able to generate these ICMP messages as fast as the machine running it can: just what we need :japanese_goblin:.

The main objective is being able to classify the traffic in the network as a normal or an abnormal situation with the help of AI algorithms. For these algorithms to be effective we need some training samples so that they can "learn" how to regard and classify said traffic. That's why we need a second tool capable of generating "normal" ICMP traffic so that we have something to compare against. Good ol' **ping** is our pal here.

</div>

#### Time to limit the links

<div style="text-align: justify">

We should no mention our scenario again. We had a **Ryu** controller, three **OVS** switches and several hosts "hanging" from these switches. The question is: **what's the capacity of the network links?**

<!-- ![escenario](https://i.imgur.com/aeteCr9.png) -->

According to Mininet's [wiki](https://github.com/mininet/mininet/wiki/Introduction-to-Mininet) that capacity is not limited in the sense that the network will be able to handle as much traffic as the hardware emulating it can. This implies that the more powerful the machine, the larger the link capacity will be. This poses a problem to our experiment as we want it to be reproducible in any host. That's why we have decided to limit each link's bandwidth during the network setup.

This behaviour is a consequence of Mininet's implementation. We'll discuss it [here](#mininet_internals) later down the road but the key aspect is that we cannot neglect Mininet's implementation when making design choices!

<br>

</div>

##### How to limit them

<div style="text-align: center">

<br>

In order to limit the available **BW** (**B**and **W**idth) we'll use Mininet's API. This API is just a wrapper for a **TC** (**T**raffic **C**ontroller) who is in charge of modifying the kernel's **planner** (i.e *Network Scheduler*). The code where we leverage the above is:

```python
net = Mininet(topo = None,
              build = False,
              host = CPULimitedHost,
              link = TCLink,
              ipBase = '10.0.0.0/8')
```

Note how we need to limit each host's capacity by means of the CPU which is what we do through the `host` parameter in Mininet's contructor. We'll also need links with a `TCLink` type. We can achieve this thanks to the `link` parameter. This will let us impose the limits to the network capacity ourselves instead of depending on the host's machines capabilities.

<br>

After fiddling with the overall constructor we also need to take care when defining the network links. We can find the following lines over at **src/scenario_basic.py**:

```python
net.addLink(s1, h1, bw = 10)
net.addLink(s1, h2, bw = 10)
net.addLink(s1, s2, bw = 5, max_queue_size = 500)
net.addLink(s3, s2, bw = 5, max_queue_size = 500)
net.addLink(s2, h3, bw = 10)
net.addLink(s2, h4, bw = 10)
net.addLink(s3, h5, bw = 10)
net.addLink(s3, h6, bw = 10)
```

We are fixing a **BW** for the links with the `bw` parameter. We have also chosen to assign a finite buffer size to the middle switches in an effor to get as close to reality as we possibly can. If the `max_queue_size` parameter hadn't been defined we would be working with "infinite" buffers at each switch's exit ports. Having these finite buffers will in fact introduce a damping effect in our tests as onece you fill them up you can't push any more data through: the output queues are absolutely full... In a real-life scenario we would suffer huge packet losses at the switches and that could be used as a symptom as well but we haven't taken it into accoun for the sake of simplicity.

We fixed the queue lengths so that they were coherent with standard values. We decided to use a **500 packet** size because *Cisco*'s (:satisfied:) queue lengths range from 64 packets to about 1000 as found [here](https://www.cisco.com/c/en/us/support/docs/routers/7200-series-routers/110850-queue-limit-output-drops-ios.html). We felt like 500 was an appropriate value in the middle ground. With all these restrictions our scenario would look like this:

<!-- ![limits](https://i.imgur.com/pzCf5GJ.png) -->

<p align="center">
    <img src="https://i.imgur.com/pzCf5GJ.png">
</p>

By inspecting the network dimensions we can see how we have a clear bottleneck... This "flaw" has been introduced on purpose as we want to clearly differentiate regurlar traffic from the one we experience when under attack.

</div>

#### Getting used to hping3

<div style="text-align: justify">

This versatile tool can be configured so that it can explore a given network, perform traceroutes, send pings or carry out out flood attacks on different network layers. All in all, it lets us craft our own packets and send them to different destinations at some given rates. You can even forge the source **IP** address to go full stealth mode :ghost:. We'll just send regular pings: **ICMP --> Echo request (Type = 8, Code = 0)** whilst increasing the rate at which we send them. This will in turn make the network core collapse making our attack successful.

Check out this [site](https://tools.kali.org/information-gathering/hping3) for more info on this awesome tool.

</div>

#### Installing things... again! :weary:

<div style="text-align: justify">

The tool will be already present on the test machine as it was included in the **Vagrantfile** as part of the VM's provisioning script. In case you want to manually install it you can just run the command below as **hping3** is usually within the default software sources:

```
sudo apt install hping3
```

</div>

#### Usage

<div style="text-align: jsutify">

As we have previously discussed this is quite a complete tool so we will only use one of the many functionalities to keep things simple. The command we'll be using is:

```
hping3 -V -1 -d 1400 --faster <Dest_IP>
```

We are going to break down each of the options:

* `-V`: Show verbose output (i.e show more information)
* `-1`: Generate ICMP packets. They'll be ping requests by default
* `-d 1400`: Add a bogus payload. This is not strictly needed but it'll help us use up the link's BW faster. We have chosen a 1400 B payload so as not to suffer fragmentation at the network layer.
* `--faster`:

<br>

We would like to point out that `hping3` could have been invoked with the `--flood` option instead of `--faster`. When using `--flood` the machine will generate as many packets as it possibly can. This would be great in a world of rainbows but... The virtual network was quickly overwhelmed by the ICMP messages and packets began to be discarded everywhere. Event though this is technically a **DoS** attack gone right too it obscures the phenomena we are faster so we decided to use `--faster` as the rate it provides suffices for our needs.

</div>

---

#### Demo time! :tada:

<div style="text-align: justify">

The attack we are going to carry out comprises hosts **1**, **2** and **4**. We'll launch `hping3` from **Host1** targeting **Host4** and we'll try to ping **Host4** from **Host2**. We will in fact see how this "regular" ping doesn't get through as a consequence of a successful **DoS** attack. The image below depicts the situation:

<!-- ![ataque](https://i.imgur.com/awt7e5v.png) -->

<p align="center">
    <img src="https://i.imgur.com/awt7e5v.png">
</p>

Let's begin by setting up the scenario like we usually do:

```bash
sudo python3 scenario_basic.py
```

Time to open terminals to both ICMP sources. We'll also fire up `Wireshark` on **Host4** to have a closer look at what's going on. Note the ampersand (`&`) at the end of the second command. It'll detach the `wireshark` process from the terminal so that we can continue running commands as we normally would. To do this we need to run:

```bash
mininet> xterm h1 h2
mininet> h4 wireshark &
```

Please note the above might fail. We're running this demo on macOS and even though we'd started [XQuartz](https://www.xquartz.org) we failed to get the *XTerm* window. We encountered a message related to some bad permissions and whatnot. This all has to do with the fact that `mininet` is ran as `root` (by means of `sudo`). Long story short, you should run this before starting `mininet`:

    sudo cp .Xauthority /root

We also struggled to get *XQuartz* to show the *XTerm* windows on macOS when connecting to the machine with `vagrant ssh test`. In order to try and circumvent that we instead issued:

    # Note the -Y has to do with trusted X11 Forwarding
    ssh -Y -i .vagrant/machines/test/virtualbox/private_key vagrant@10.0.123.2

Note the above has been 'explained' at the beginning of the document.

<br>

Time to launch `hping3` from **Host1** with the parameters we discussed:

<p align="center">
    <img src="img/Image 5.png">
</p>

<br>

If we now try to ping **Host4** from **Host2** we'll fail horribly:

<p align="center">
    <img src="img/Image 6.png">
</p>

<br>

If we halt the **DoS** attack we will see the regular traffic resume its normal operation after a short period of time.


<br>

We then see how the **DoS** attack against **Host4** has been successful. In order to facilitate issuing the needed commands we have prepared a couple of `python` scripts containing all the needed information so that we only need to run them and be happy. You can find them at:

* Attack: [`src/ddos.py`](https://github.com/GAR-Project/project/blob/master/src/ddos.py)
* Regular traffic: [`src/normal.py`](https://github.com/GAR-Project/project/blob/master/src/normal.py)

With all this ready to rock we now need to focus on detecting these attacks and seeing how to possibly mitigate them.

</div>

---

## Traffic classification with a SVM (**S**upport **V**ector **M**achine)
We have our scenario working properly and the attack is having the desired effect on our network. In other words, it's blowing things up. If we are to detect the attack we need to gather representative data and process it somehow so that we can predict whether we are under attack or not. As Jack the Ripper once said, let's break this into parts. We'll begin by gathering the necessary data and sending it to a database we can easily query. We'll then prepare training datasets for our SVM and get it ready for making guesses. Let's begin!


### First step: Getting the data collection to work :dizzy_face:

#### What tools are we going to use?
For a previous project belonging to the same subject we were introduced to both **telegraf** and **influxdb**. The first one is a metrics agent in charge of collecting data about the host it's running on. It's entirely plugin driven so configuring it is quite a breeze! The latter is a **DBMS** (**D**ata**B**ase **M**anagement **S**ystem) whose architecture is specifically geared towards time series, just what we need! The interconnection between the two is straightforward as one of **telegraf**'s plugins provides native support for **influxdb**. We'll have to configure both appropriately and we'll see it wasn't as easy as we once thought due to mininet getting in the way. We have come up both with a "hacky" solution and an alternative any Telecommunications Engineer would be prod of. Just kidding, but it uses networking concepts and not workarounds though.

#### Leveraging the Mininet's shared filesystem
Have you ever felt like throwing yourself into `/dev/null` to never come back? That was pretty much our mood when trying to get a host within mininet's network to communicate with the outside world. In order to understand how we ended up "fixing" (it just works :grimacing:) everything we need to go back and take a look at our initial ideas and implementations.

We should not forget that we are looking at `ICMP` traffic in order to make predictions about the state of the network. We first thought about running **telegraf** on a network switch that was directly connected to the controller where our **InfluxDB** instance is running. The good thing about this scheme is that the telegraf process within the switches can communicate with the DB running in the controller through `HTTP`. This is due to the fact that we are invoking the `start()` method of the switches during the network configuration so even though there's no "real" link between them (we didn't create it by calling `addLink()`) they can still communicate.

The above sounds wonderfully well but... switches can only work with information up to the **link layer**, they know nothing about **IP** packets or **ICMP** messages. We should note that **ICMP** is a layer 3-ish (more like layer 3.5) protocol. As it relies on IP for the network services but doesn't have a port number we cannot assign a particular layer to it... All in all the switches knew nothing about ICMP messages crossing them so we find that we need to run telegraf on one of the hosts if we want to get our metrics. In a real case scenario we could devote a router (which can process ICMP data) instead of a switch for this purpose and reconfigure the network accordingly. Anyway we need to get the telegraf instance running in one of the mininet created hosts to communicate with the influx database found in the controller VM. Let's see how we can go about it...

When discussing the internal mechanisms used by mininet later on we'll find out that it relies solely on network namespaces. This implies that the filesystem is shared across the network elements we create with mininet **AND** the host machine itself. This host machine has direct connectivity with the VM hosting the controller so we can take advantage of what others consider to be a flaw in mininet's architecture. We are going to run a telegraf instance on mininet's `Host 4` whose input plugin will gather ICMP data and whose output will be a file in the VM's home directory. We'll be running a second telegraf instance in the host VM whose input will be the file containing `Host 4`'s output and whose output will be the Influx DB hosted in the controller VM. This architecture leverages the shared filesystem and uses a second telegraf instance as a mere proxy between one of mininet's internal hosts and the controller VM, both living in entirely different networks.

In order to implemnent this idea we have created all the necessary configuration files under `conf` to then copy them to the appropriate places during Vagrant's provisioning stage.

#### Implementing a NAT (**N**etwork **A**ddress **T**ranslator) in Mininet for external communication
Once we implemented the solution above we were able to continue developing the **SVM** as we already had a way of retrieving data. That's why we decided to devote some time to looking for a more elegant solution. Just like we usually do in home LANs we decided to instantiate a NAT process to get interconnection to the network created for the VM's from within the emulated one. Due to problems with the internal functioning of this NAT process provided by Mininet, extra configuration had to be added to achieve the desired connectivity. To solve the problem a series of predefined rules (flows) were installed in each switch to "route" the traffic from our data collector to the NAT process and from there to the outside to InfluxDB.  This could be considered a "fix", but in fairness we are only using the logic of an SDN network to route our traffic in the desired way.  You can take a closer look at this implementation [in this branch](https://github.com/GAR-Project/project/tree/full-connectivity).

#### What data are we going to use?
We are trying to overwhelm `Host 4` with a bunch (a **VERY BIG** bunch) of `ICMP Echo Requests` (that is fancy for `pings`). By reading through telegraf's input plugin list we came across the **net** plugin capable of providing `ICMP` data out of the box.

#### Getting the data to InfluxDB
Instead of directly sending the output to an influxdb instance we are going to send it to a regular file thanks to the **file** output plugin. This leads us directly to the configuration of the second telegraf instance.

In this second process we'll be using the **tail** input plugin. Just like Linux's `tail`, this command will continuously read a file so that it can use it as an input data stream. Instead of polling the file continuously we chose to instead notify telegraf to read it when changes took place. This leads to a more efficient use of system resources overall. The output plugin we'll be using is now good ol' **influxdb**. We'll point it to the influxdb instance running on the `controller` VM so that everything is correctly connected.

The structure of the system we are dealing with is then:

<p align="center">
    <img src="https://i.imgur.com/RO2w885.png">
</p>

We are now ready to start querying our database and begin working with the acquired information.

#### A note on the sampling period
When configuring the interconnection between both telegraf instances we initially left the default `10 s` refresh interval in both. When we read the data we were getting in the DB we noticed some "satrange" results in between correct readings so we decided to fiddle with these sampling times in case they were interfering with each other. As we are communicating both processes by means of a file the timing for reading and writing can be critical... We fixed a `2 s` sampling interval in "mininet's" telegraf process and a `4 s` refresh rate in the VM's instance. This means that we are going to get 2 entries in the DB with each update!

After running some tests we found everything was working flawlessly now :ok_woman: so we just left it as is.

### Second step: Generating the training datasets

#### Weren't we using the received ICMP messages as the input?
Well... yes and no. The cornerstone for the SVM's input is indeed the number of received ICMP messages **BUT** we decided to use the *derivative* of the incoming packets with respect to time instead of the absoulute value. This approach will let the network admin apply the exact same SVM for attack detection even if the traffic increases due to a network enlargement. As we are looking for sudden changes in incoming messages rather than for large numbers this approach is more versatile.

After debating it for a while we settled on including the average of the derivative of the incoming packets as a parameter too. As the mean will vary slowly due to the disparity of the data generated by both situations we'll be more likely to consider the aftermath as an attack too. Even though we may not be subject to very high incoming packet variations any more we'll take a while to resume a normal operation and we decided to let this "recovery time" play a role in the SVM's prediction.

#### Writing a script: `src/data_gathering.py`
Once we have the desired data stored in the DB using the SVM becomes a matter of reading it and formatting it so that the SVM "likes it". In order to make the process faster we decided to write a simple python script that uses influxdb's python API to read the data and prepare a **CSV** (**C**omma **S**eparated **V**alues) to later be read by the script implementing the SVM.

The defining quality of training data is meaningfulness. The SVM's predictions will only be as good as the training it received so we need to provide insightful data if we are to get any consistent results.

In order to get appropriate data samples we went ahead an simulated regular traffic by pinging the target host at a rate of roughly 1 ICMP message per second. We then attacked the target until we got around 100 samples into de DB.

Generating the DB is just a matter of reading the DB and outputting the read data to a text file with a `.csv` extension.

### Third step: Putting it all together: `src/traffic_classifier.py`
Apart from the scenario's setup the most important program we wrote is the `traffic classifier` without a doubt. The file defines the `gar_py()` class which includes a SVM instance, the query used for getting data and many other configuration parameters as its attributes. This let's us use this same technique in other scenarios, in other words, we are increasing this solution's portability.

The class' constructor will limit itself to initializing its attributes and training the SVM by reading the training files we have already prepared. The main thing to note here is how we need to conform to the input format accepted by the SVM itself.

Once it's trained we just need to call the class's `work_time()` method which will enter an infinite loop whose operation can be summarised into these points:

1. Read the last 3 entries in the DB.
2. Verify these entries are indeed new.
3. Update the parameters we're going to use for the prediction.
4. Order the SVM to predict wheteher the new data represents an attack or not.
5. Write an entry to the appropriate DB signaling whether or not we're under attack.
6. Wait 5 seconds to read new data. New data is sent to the DB every 4 seconds so reading insaley fast is just throwing resources out the window.

Additionally we used matplotlib to draw the classification we were carrying out. As you can see, the red dots are those data that have been classified as an anomalous traffic, DDoS traffic, and although it seems that there is only one blue dot belonging to "normal" traffic, it is not the case, there are several but their deviation between them is minimal :relaxed: .

<p align="center">
    <img src="https://i.imgur.com/8nfNKYn.png" width="50%">
</p>

We've also written a signal handler to allow for a graceful exit when pressing `CTRL + C`.

And with that we are finished! :tada: We hope to have been clear enough but if you still have any questions don't hesitate to contact. 

---


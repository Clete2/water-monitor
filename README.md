# Introduction
This project has been created to assist you in building your own solution to monitor your sump pump or any other water level and receive alerts based on a threshold that you control.

I have personally chosen the following stack for this project, but you can choose whatever you are most comfortable with. Note that none of the links below are affiliate links nor am I affiliated with any product mentioned below.

The guide assumes you have a moderate understanding of Linux and are comfortable with the command line.

## Hardware
- [Raspberry Pi](https://www.raspberrypi.org)
  - Breadboard, GPIO breakout board, power supply, wires, and microSD card
- [Milone Technologies eTape](https://milonetech.com/products/standard-etape-assembly) - water level sensor with standard assembly and voltage divider (24")
- [Adafruit ADS1015](https://www.adafruit.com/product/1083) - analog to digital signal converter to read the water level sensor
- [Waterproof DS18B20](https://www.adafruit.com/product/381) - temperature sensor (optional)
- 10k ohm resistor?? TODO

## Software
Don't worry about installing the software just yet. We will install Docker and Docker Compose and it'll take care of the rest of the stack!
- [Raspbian](http://raspbian.org) - a Debian based operating system for your Raspberry Pi
- [Docker](https://docs.docker.com/install/) - a lightweight container system
- [Docker Compose](https://docs.docker.com/compose/install/) - an easy way to orchestrate building multiple docker containers and networking them together
- [Grafana](https://grafana.com) - a beautiful metric graphing frontend
- [InfluxDB](https://www.influxdata.com/time-series-platform/influxdb/) - a time series database for storing metrics
- [Telegraf](https://www.influxdata.com/time-series-platform/telegraf/) - collects metrics and sends them to InfluxDB
- [Telegram Messenger](https://telegram.org) - a chat client with apps for Android and iOS that can send push notifications to your phone (optional, you can use any alterting mechanism you choose)

# Regular server or computer (not Raspberry Pi)
## Software
I don't have an easy way to run Grafana on the Raspberry Pi because the official docker image doesn't have a Raspberry Pi variant. Therefore I recommend you do this part on a normal Linux server or on your own computer.

We will install [Docker](https://docs.docker.com/install/) and [Docker Compose](https://docs.docker.com/compose/install/). Follow the directions for each using the inline links.

Next, clone this project using `git` to get all the files you need to get started running Grafana, InfluxDB, and Telegraf in containers.

`$ git clone https://github.com/Clete2/water-monitor.git`

Now we will build the docker images and use `docker-compose` to start up our our containers.

`$ cd water-monitor`

Build the Docker containers

`$ sudo docker-compose build`

Run the Docker containers. `-d` daemonizes them so that they run in the background.

`$ sudo docker-compose up -d`

If you execute `sudo docker ps` you should see something similar to the below text:

    CONTAINER ID        IMAGE                   COMMAND                  CREATED             STATUS              PORTS                          NAMES
    c550e4d8e9ba        watermonitor_grafana    "/run.sh"                About an hour ago   Up About an hour    0.0.0.0:3000->3000/tcp         watermonitor_grafana_1
    99ef98096e03        watermonitor_telegraf   "/entrypoint.sh tele…"   About an hour ago   Up About an hour    8092/udp, 8125/udp, 8094/tcp   watermonitor_telegraf_1
    5ba9311117d1        influxdb                "/entrypoint.sh infl…"   About an hour ago   Up About an hour    0.0.0.0:8086->8086/tcp         watermonitor_influxdb_1

Congratulations! You're now running Grafana, InfluxDB, and Telegraf on your machine! Time to explore it and get familiar.

## Exploring Grafana
The setup that I have provided in the Docker Compose file is a very simple setup.

### Telegraf
For metric collection using Telegraf, I enabled all the default metrics like `cpu` and `memory`. As a bonus I added a `ping` metric that routinely pings websites to check if they are up or down and logs latency to each site.

### InfluxDB
I didn't do anything other than the base installation!

### Grafana
I configured InfluxDB as a data source and I created a dashboard to show the results of the `ping` metric.

# Raspberry Pi Setup
## Operating System
First things first, you'll need to set up your Raspberry Pi. I'm not going to detail this out because you can do it a variety of ways. I recommend that you install [Raspbian](http://raspbian.org). To enable SSH, add a file called `ssh` to the SD card.

If you're uncomfortable with that, check out the [official guide](https://www.raspberrypi.org/learning/software-guide/) and plug in a mouse, keyboard, and monitor. I highly recommend you spend time using the Raspberry Pi and becoming familiar with Linux before continuing.

## WiFi
If you have a Raspberry Pi 3 or newer, be sure to change the file `/etc/wpa_supplicant/wpa_supplicant.conf` and provide your WiFi information:

    country=US
    ctrl_interface=DIR=/var/run/wpa_supplicant GROUP=netdev
    update_config=1
    network={
            ssid="My Network Name"
            psk="My Network Password"
    }

## Docker
Open up a terminal or SSH into your Raspberry Pi. At this point I'm going to assume you are running as `root` and are in `root`'s home directory. 

`$ sudo su -`

`$ apt install docker && systemctl enable docker && systemctl start docker`

Save off a sample Telegraf config file:

`$ docker run --rm telegraf telegraf config > telegraf.conf`

Configure the `[[outputs.influxdb]]` section to use a URL that points to the IP address of the computer you are currently using or the server you plan to install Grafana on.

Edit `telegraf.conf` and add the following lines. Ignore the first `[[inputs.logparser]]` section if you chose to not use a temperature probe:

    [[inputs.logparser]]
        files = ["/logs/temp.log"]
        from_beginning = true

        [inputs.logparser.grok]
            patterns = ['%{TIMESTAMP_ISO8601:timestamp:ts-"2006-01-02 15:04:05"} ambient_downstairs=%{NUMBER:ambient_downstairs:float}, water=%{NUMBER:water:float}']

    [[inputs.logparser]]
        files = ["/logs/water.log"]
        from_beginning = true

        [inputs.logparser.grok]
            patterns = ['%{TIMESTAMP_ISO8601:timestamp:ts-"2006-01-02 15:04:05"} %{NUMBER:water_level:float}']

The `logparser` input will read a log and turn it into a metric. Here we are capturing two metrics using two instances of the `logparser` input.
<p align="center">
  <br/>
  <img src=resources/cluster-logo.png alt="zeroSwarm">
  <br/>
</p>

# Raspberry Pi (Zero) cluster with Docker Swarm

## Stuff you'll need
In this project I'll use:

| Material      | Units          | Price |
| ------------- |:-------------:| -----:|
| [Raspberry Pi Zero W](https://www.kubii.es/pi-zero-w/1851-raspberry-pi-zero-w-kubii-3272496006997.html)      | x2 | ~10€/each |
| [Samsung EVO (class 10) 32GB](https://www.amazon.es/Samsung-EVO-Tarjeta-memoria-adaptador/dp/B06XW91D41/) microSD card      | x2 | ~13€/each |
| [5V 2.4A Dual USB Charger](https://www.kubii.es/cargadores-fuentes-raspberry-pi/1753-alimentacion-5v-24a-dual-usb-kubii-3272496006201.html)      | 1x | ~10€ |
| (_optional_) [TP-LINK TL-WR802N](https://www.amazon.es/TP-LINK-TL-WR802N-inal%C3%A1mbrico-portable-garant%C3%ADa/dp/B00TQEX8BO/) Nano router     | 1x | ~25€ |

## Steps
1. **Install [Raspbian Stretch Lite](http://downloads.raspberrypi.org/raspbian_lite/images/) on each microSD card**. I've used the version from 2018-03-13.
2. **Enable SSH and set up Wi-Fi**. In order to do this, insert the microSD into your pc, navigate to `/media/YOUR_USER_NAME/boot` and create an empty `shh` file:
    ```bash
    $ sudo touch ssh
    ```
    Then, open the other volume and use `sudo nano /etc/wpa_supplicant/wpa_supplicant.conf` to include the following lines:
    ```
    network={
        ssid="YOUR_NETWORK_NAME"
        psk="YOUR_PASSWORD"
        key_mgmt=WPA-PSK
    }
    ```
3. **Reproduce step 2 on each microSD** and turn on all the Raspberry's.
4. After 20 seconds, **scan your network to find the IP of each Raspberry Pi Zero**. You can either access your Wi-Fi router's settings or use:
    ```bash
    $ sudo nmap -sn 192.168.1.0/24
    ```
    Note: This step is optional if you have set up static IP addresses for all your nodes.
5. **SSH into your nodes (`ssh pi@192.168.1.XYZ`) and install Docker**. By default the password is _raspberry_. Then (for each node) run:
    ```bash
    pi@raspberrypi:~ $ curl -sSL https://get.docker.com | sh
    ```
    Note: This will take a while!
6. **Only on the first (master) node** initialize the swarm by using:
    ```bash
    pi@raspberrypi:~ $ sudo docker swarm init --advertise-addr 192.168.1.XYZ
    ```
    Where `192.168.1.XYZ` is the IP of the current node. This will generate a command that will be used to add the rest of the nodes to the swarm.
7. **SSH into the rest of the nodes** and run the command produced on the previous step to connect the swarm.
8. On the master node (the one from step 6), run:
    ```bash
    pi@raspberrypi:~ $ sudo docker node ls
    ```
    to check all the nodes on the swarm. [You can create a swarm of one manager node, but you cannot have a worker node without at least one manager node](https://docs.docker.com/engine/swarm/how-swarm-mode-works/nodes/). This means that we'll have to promote our second node to a manager:
    ```bash
    pi@raspberrypi:~ $ sudo docker promote NODE_ID
    Node NODE_ID promoted to a manager in the swarm.
    ```

Now, let's create a service in our swarm with `docker service create`:
```bash
pi@raspberrypi:~ $ sudo docker service create \
    --name viz \
    --publish 8080:8080/tcp \
    --constraint node.role==manager \
    --mount type=bind,src=/var/run/docker.sock,dst=/var/run/docker.sock \
    alexellis2/visualizer-arm:latest
```

This will build and start an image with just one replica:
```
pi@raspberrypi:~ $ sudo docker service ls
ID   NAME MODE       REPLICAS IMAGE                            PORTS
xxxx viz  replicated 1/1      alexellis2/visualizer-arm:latest *:8080->8080/tcp
```

To scale that image, run:
```
pi@raspberrypi:~ $ sudo docker service scale viz=4
```

At this point you should be able to visit `192.168.1.XYZ:8080` to see a nice swarm [visualizer](https://github.com/ManoMarks/docker-swarm-visualizer):

<p align="center">
  <br/>
  <img src=resources/vis.png alt="Visualizer">
  <br/>
</p>

Congrats!

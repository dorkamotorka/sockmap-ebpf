# eBPF SockMap

First we need to build and run the eBPF program:
```
go generate
go build
sudo ./sockmap
```

Then we spawn a server and a client process in different network namespaces as well as setting up the cgroup(v2) and the veth pairs using:
```
mkdir -p /sys/fs/cgroup/unified/test.slice
ip netns add A
ip netns add B
ip -n A link add name veth0 type veth peer name veth0 netns B
ip -n A link set dev lo up
ip -n B link set dev lo up
ip -n A link set dev veth0 up
ip -n B link set dev veth0 up
ip -n A addr add 10.0.0.1/24 dev veth0
ip -n B addr add 10.0.0.2/24 dev veth0
```

Network namespace A is where we run our server using:
```
echo $$ > /sys/fs/cgroup/unified/test.slice/cgroup.procs # Put current shell process into the cgroup to which our BPF program is attached to 
ip netns exec A sockperf server -i 10.0.0.1 --tcp --daemonize # Run the server inside the network namespace A
```

And then run the client (new shell) in the network namespace B using:
```
echo $$ > /sys/fs/cgroup/unified/test.slice/cgroup.procs # Put current shell process into the cgroup to which our BPF program is attached to
ip netns exec B sockperf ping-pong -i 10.0.0.1 --tcp --time 30 # Run the client inside the network namespace B
```


The latency results I get on my machine is `18.282 usec` without the Sockmap, and with the latency reduces to `12.997 usec`, which is a aprox. `30%` improvement!

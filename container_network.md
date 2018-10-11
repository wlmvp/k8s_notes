#自己创建一个网桥
brctl addbr testbr
ip addr add 172.18.0.1/16 dev testbr

#创建网卡对
ip link add name veth1 mtu 1500 type veth peer name veth2 mtu 1500

#一端绑定网桥
#ip link set veth1 master docker0 
ip link set veth1 master testbr    
ip link set veth1 up

#另一端放进容器网络的netns中
docker inspect '--format={{ .State.Pid }}' bb1fc3609735

rm -f /var/run/netns/6027    
ln -s /proc/6027/ns/net /var/run/netns/6027
ip link set veth2 netns 6027
ip netns exec 6027 ip link set veth2 name eth0
ip netns exec 6027 ip addr add 172.18.0.9/24 dev eth0    
ip netns exec 6027 ip link set eth0 up

#配置容器网络的网关
ip netns exec 6027 route add default gw 172.18.0.1

#访问外网，通过nat
iptables -t nat -S
iptables -t nat -A POSTROUTING -s 172.18.0.0/16 ! -o testbr -j MASQUERADE





#ovs vxlan--------------------------------------------------------------------------------

# 创建一个openvswitch bridge
ovs-vsctl add-br ovs-br0
# 添加一个到192.168.119.32的接口
ovs-vsctl add-port ovs-br0 vxlan-port-to-192.168.119.141 -- set  interface vxlan-port-to-192.168.119.141 type=vxlan option:remote_ip="192.168.119.141"

# 创建一对虚拟网卡veth
ip link add vethx type veth peer name vethContainer

# sleep 3 seconds to wait for the completion of previous work.
sleep 3

# 将vethx接入到ovs-br0中
ovs-vsctl add-port ovs-br0 vethx
ifconfig vethx up

# 如果net namespace目录没有创建则新建一个
if [ ! -d "/var/run/netns" ]; then
  mkdir -p /var/run/netns
fi
# 将docker容器使用的net namespace 打回原形
ln -s /proc/${pid}/ns/net /var/run/netns/${pid}
ip netns list

# 将vethContainer加入到容器的net namespace中
ip link set vethContainer netns ${pid}

# 配置vethContainer接口
ip netns exec ${pid} ifconfig vethContainer 192.168.100.101/24 up
ip netns exec ${pid} ifconfig -a
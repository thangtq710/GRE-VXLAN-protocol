# Tìm hiểu về GRE và VXLAN 

**Mục lục:**

	1.Giới thiệu về Overlay network
 	
	2.Cơ chế hoạt động/vm-to-vm communication.
   	
	3.Giới thiệu về VXLAN và GRE.
   
   	4.Lab
   
   	5.Tổng kết   

########

**1. Giới thiệu về Overlay network**

- Trong những năm trở lại đây, các kỹ thuật ảo hóa và đám mây đã trở nên phổ biến hơn bao giờ hết. Trong trung tâm dữ liệu ảo hóa, mỗi máy chủ vật lý có thể chạy một hoặc nhiều máy chủ ảo dựa trên nền hypervisor.

- Kiến trúc mạng bây giờ phải đối mặt với nhu cầu kết nối giữa các máy chủ ảo với nhau, mỗi Virtual Machine (VM) yêu cầu 1 địa chỉ MAC duy nhất và 1 địa chỉ IP. 

- Trong trung tâm dữ liệu ảo hóa, các mạng L2 này cần được cô lập thành các mạng riêng biệt, nếu như với môi trường với số lượng endpoint nhỏ, VLAN là giải pháp hoàn hảo. Tuy nhiên, khi mà số lượng máy ảo trên các máy chủ vật lý càng ngày càng tăng với số lượng lớn theo thời gian ( ví dụ trong môi trường cloud computing), kéo theo yêu cầu cô lập các mạng ảo này thì nảy sinh vấn đề vì số VLAN ID tối đa chỉ là 4096. Mặt khác việc di chuyển các máy chủ ảo từ máy chủ vật lý này sang máy chủ vật lý khác phải nhanh chóng, thuận lợi. Để đáp ứng các yêu cầu này, các kỹ thuật chồng lấn khác nhau (network overlay) được ra đời bao gồm: VXLAN, GRE, SPBV, TRILL, Fabric Path… Cụ thể trong khuôn khổ của bài viết này mình sẽ trình bày về VXLAN và GRE.

- Đầu tiên ta tìm hiểu Overlay network là gì?

![alt](images/overlaynetwork.png)

- Overlay networking là công nghệ cho phép tạo ra các mạng ảo (logic) trên hệ thống mạng vật lý bên dưới (underlay network) mà không ảnh hưởng hoặc ảnh hưởng không đáng kể tới hạ tầng mạng bên dưới. Cụ thể hơn, với overlay network, ta có thể tạo ra các mạng ảo L2 trên nền hạ tầng mạng L3 network.

**2. Cơ chế hoạt động **

![alt](images/vm-to-vm.png)

- Khi cấu hình overlay network cho mạng ảo, GRE/VXLAN tạo nên các VTEP -(Virtual Tunnel Endpoints). Chúng ta có thể hình dung VTEP có hai giao diện – Một kết nối với mạng IP ở giữa và giao diện còn lại kết nối tới mạng nội bộ của các VM.

- Chức năng của Tunnel Endpoint là đóng gói VM trafic trong một IP header để gửi qua mạng IP bên dưới.

- Tư tưởng chung của các công nghệ như GRE hay VXLAN là khi một mạng overlay tạo nên thì sẽ có một ID duy nhất để định danh cho mạng đó đồng thời được sử dụng để đóng gói lưu lượng (traffic encapsulation). Mỗi gói tin giữa các VM trên các host vật lý khác nhau được đóng gói trên một host và gửi tới các host khác thông qua point-to-point GRE hoặc VXLAN tunnel. Khi gói tin tới host đích, các tunnel header sẽ bị loại bỏ (tại tunnel endpoint) và gói tin được chuyển tiếp tới bridge kết nối với VM.

- Sơ đồ sau đây thể hiện gói tin trước và sau khi được đóng gói bởi host.

![alt](images/image1.png)

- Theo hình mô tả ở trên, địa chỉ nguồn và đích trong outer IP header sẽ định danh cho endpoint của tunnel. Tất cả các host tham gia vào VXLAN/GRE thì hoạt động như một tunnel end points.

- Các địa chỉ nguồn và đích trong Inner IP header định danh cho các VM gửi và nhận payload.

**3. Một số công nghệ overlay network**

**3.1. VXLAN**

- Virtual Extensible LAN (VXLAN) là giao thức tunneling, thuộc giữa lớp 2 và lớp 3.

- VXLAN là giao thức sử dụng UDP (cổng 4789) để truyền thông và một segment ID độ dài 24 bit còn gọi là VXLAN network identifier (VNID). Chỉ các máy ảo trong cùng VXLAN segment mới có thể giao tiếp với nhau..

- VXLAN ID (VXLAN Network Identifier hoặc VNI) là 1 chuỗi 24-bits so với 12 bits của của VLAN ID. Do đó cung cấp hơn 16 triệu ID duy nhất ( giá trị này của VLAN: 4096 ).

- VXLAN frame format:

![alt](images/vxlanframe.png)

- Frame Ethernet thông thường bao gồm địa chỉ MAC nguồn, MAC đích, Ethernet type và thêm phần VLAN_ID (802.1q) nếu có. Đây là frame được đóng gói sử dụng VXLAN, thêm các header sau:
	
	- VXLAN header: 8 byte bao gồm các trường quan trọng sau:
		
		* Flags: 8 but, trong đó bit thứ 5 (I flag) được thiết lập là 1 để chỉ ra rằng đó là một frame có VNI có giá trị. 7 bit còn lại dùng dữ trữ được thiết lập là 0 hết.
		
		* VNI: 24 bit cung cấp định danh duy nhất cho VXLAN segment. Các VM trong các VXLAN khác nhau không thể giao tiếp với nhau. 24 bit VNI cung cấp lên tới hơn 16 triệu VXLAN segment trong một vùng quản trị mạng.

    - Outer UDP Header: port nguồn của Outer UDP được gán tự động và sinh ra bởi VTEP và port đích thông thường được sử dụng là port 4789 hay được sử dụng (có thể chọn port khác).

    - Outer IP Header: Cung cấp địa chỉ IP nguồn của VTEP nguồn kết nối với VM bên trong. Địa chỉ IP outer đích là địa chỉ IP của VTEP nhận frame.

    - Outer Ethernet Header: cung cấp địa chỉ MAC nguồn của VTEP có khung frame ban đầu. Địa chỉ MAC đích là địa chỉ của hop tiếp theo được định tuyến bởi VTEP
	
**3.2. GRE**

- GRE-Generic Routing Encapsulation là giao thức được phát triển đầu tiên bởi Cisco, Giao thức này sẽ đóng gói một số kiểu gói tin vào bên trong các IP tunnels để tạo thành các kết nối điểm-điểm (point-to-point) ảo. Các IP tunnel chạy trên hạ tầng mạng công cộng.

- Đây là một phương pháp đơn giản và hiệu quả để chuyển dữ liệu thông qua mạng public network, như Internet. GRE cho phép hai bên chia sẻ dữ liệu mà họ không thể chia sẻ với nhau thông qua mạng public network.

- GRE đóng gói dữ liệu và chuyển trực tiếp tới thiết bị mà de-encapsulate gói tin và định tuyến chúng tới đích cuối cùng. Gói tin và định tuyến chúng tới đích cuối cùng. GRE cho phép các switch nguồn và đích hoạt động như một kết nối ảo point-to-point với các thiết bị khác (bởi vì outer header được áp dụng với GRE thì trong suốt với payload được đóng gói bên trong).

**4. Lab**

- **4.1. Lab VXLAN với Open vSwitch

- Topology:
	- Host 1: 192.168.30.130
		* vm1: 10.0.0.101/24
		* vswitch br0: 10.0.0.1
		* vswitch br1: 192.168.30.130
	
	- Host 2: 192.168.30.184
		* vm2: 10.0.0.102/24
		* vswitch br0: 10.0.0.2
		* vswitch br1: 192.168.30.184
		
![alt](images/lab.png)

- Mô tả:

	- Dưới đây mình sẽ thực hiện 1 bài lab sử dụng vmware để chạy 2 máy ảo host1, host2 đóng vai trò như các node vật lí trong thực tế.

	- Trên 2 host này, sẽ được cài hệ điều hành Ubuntu Server 16.04, cài sẵn các phần mềm Open vSwitch, KVM với QEMU, libvirt-bin để tạo các vm. 2 host này đều sủ dụng card mạng ens33 ( coi như là card mạng vật lý).

	- Dùng wireshark để bắt gói tin VXLAN
	
- Cấu hình:

	* Tạo 2 vSwitch br0 và br1 trên cả 2 host.

	* Cấu hình chế độ mạng bridge cho vSwitch br1 và card mạng ens33 trên cả 2 host.
	
	* Trên HOST 1, tạo VM1(cirros1) kết nối với vSwitch br0. Trên HOST 2 tạo VM2(cirros2) kết nối với vSwitch br0.

- Mục tiêu bài lab:

	* Kiểm tra kết nối giữa VM1 và VM2.
    * Kiếm tra gói tin VXLAN và so sánh với lý thuyết.
	
- Thực hiện: 
	* Đầu tiên cài đặt Open vSwitch và KVM trên cả 2 Host:
	```
	apt-get install qemu-kvm qemu virt-manager virt-viewer libvirt-bin libvirt-dev
	apt-get install openvswitch-switch
	```
	
	* Tạo 2 vswitch br0 và br1 trên cả 2 host:
	
	```
	ovs-vsctl add-br br0
	ovs-vsctl add-br br1
	```
	
	* Bật 2 vswitch trên cả 2 host 
	
	```
	ip link set dev br0 up
	ip link set dev br1 up
	```
	
	* Trên host1 tạo chế độ mạng bridge cho vswitch br1 và card mạng ens33:
	
	```
	ovs-vsctl add-port br1 ens33 
	ifconfig ens33 0 && ifconfig br1 192.168.30.184/24
	```
	* Trên host2 tạo chế độ mạng bridge cho vswitch br1 và card mạng ens33:
	
	```
	ovs-vsctl add-port br1 ens33 
	ifconfig ens33 0 && ifconfig br1 192.168.30.130/24
	```
	
	* Add lại route trên cả 2 host:
	
	```
	route add default gw 192.168.30.1 br1
	```
	
	* Cấu hình IP cho br0 trên host 1:
	
	```
	ip a add 10.0.0.1/24 dev br0
	```
	* Cấu hình IP cho br0 trên host 2:
	
	```
	ip a add 10.0.0.2/24 dev br0
	```
	
	* Cấu hình VXLAN tunnel cho vswitch br0 trên host 1:
	
	```
	ovs-vsctl add-port br0 vxlan1 -- set interface vxlan1 type=vxlan option:remote_ip=192.168.30.130
	```
	* Cấu hình VXLAN tunnel cho vswitch br0 trên host 2:
	
	```
	ovs-vsctl add-port br0 vxlan1 -- set interface vxlan1 type=vxlan option:remote_ip=192.168.30.184
	```
	
	* Show config vừa cấu hình:
	
	```
	root@ubuntu:~# ovs-vsctl show
	a5886467-efe3-425a-acf9-ae949e0b2207
    Bridge "br0"
        Port "vxlan1"
            Interface "vxlan1"
                type: vxlan
                options: {remote_ip="192.168.30.130"}
        Port "vnet0"
            Interface "vnet0"
        Port "br0"
            Interface "br0"
                type: internal
    Bridge "br1"
        Port "ens33"
            Interface "ens33"
        Port "br1"
            Interface "br1"
                type: internal
    ovs_version: "2.5.5"
	```
	
	* Tạo mạng ovs0 với vswitch ovs1 trên cả 2 host: 
	
	```
	<network>
  		<name>ovs0</name>
  		<forward mode='bridge'/>
  		<bridge name='br0'/>
  		<virtualport type='openvswitch'/>
	</network>
	```
	* Thực hiện các command sau để tạo mạng và bắt đầu mạng:

	```
	virsh net-define ovs0.xml
	virsh net-start ovs0
	virsh net-autostart ovs0
	```
	* Check mạng vừa tạo:
	
	```
	root@ubuntu:~# virsh net-list
 	Name                 State      Autostart     Persistent
	----------------------------------------------------------
 	default              active     yes           yes
 	ovs0                 active     yes           yes
	```
	
	* Create Virtual machines testvm and attach network ovs0 vừa tạo ở trên ( làm tương tự với host còn lại):
	
	```
	virt-install --name=testvm --ram=512 --vcpus=1 --cdrom=/root/ubuntu-16.04.2-server-amd64.iso --os-type=linux --os-variant=ubuntu16.04 --network network:ovs0 --disk path=/var/lib/libvirt/images/testvm2.dsk,size=8
	```
	
	* *Note: Fix qemu-kvm: could not open disk image ' ': Permission denied các bạn có thể tham khảo ở đây:*
	
	```
	https://github.com/jedi4ever/veewee/issues/996
	```
	
	* Host testvm2 được tạo với IP: 10.0.0.102
	
	![alt](images/testvm.png)
	
	* Tương tự ta tạo VM (testvm1) trên host 1 với IP: 10.0.0.101
	
	-> Kết quả ta test thử kết nối giữa 2 vm trên 2 host:
	
	![alt](images/testping.png)
	
	* Dùng wireshark bắt gói tin và phân tích: 
	
	![alt](images/wireshark.png)
	
	
	
	
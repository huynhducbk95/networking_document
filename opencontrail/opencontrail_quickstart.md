# OpenContrail quick start
## 1. Overview
Chương này sẽ cung cấp cho chúng ta một cái nhìn tổng quát về hệ thống OpenContrail - một nền tảng mở rộng cho Software Defined Networking (SDN).

Tất cả các khái niệm chính được giới thiệu một cách ngắn gọn trong chương này và sẽ được tìm hiểu một cách chi tiết trong phần còn lại của tài liệu.

## 1.1. Use cases
OpenContrail là một hệ thống mở rộng có thể được sử dụng cho nhiều use case về networking, tuy nhiên có hai driver chính trong kiến trúc này:
-  Cloud Networking: Các Private Cloud của các doanh nghiệp hoặc các nhà cung cấp dịch vụ (Service Provider), Infrastructure as a Service (IaaS) và Virtual Private Cloud (VPCs) đối với các nhà cung cấp dịch vụ cloud (Cloud Service Provider).
- Network Function Virtualization (NFV) trong mạng của các nhà cung cấp dịch vụ (Service Provider Network): Điều này cung cấp các dịch vụ giá trị gia tăng (Value Added Service) cho các mạng biên của các nhà cung cấp dịch vụ mạng chẳng hạn như business edge networks, broadband subscriber management network, và mobile egde network.

Đối với các trường hợp sử dụng là Private cloud, virtual private cloud (VPC), và IaaS, thì tất cả chúng đều liên quan đến một hạ tầng vật lý (data center) được ảo hóa và được sử dụng bởi nhiều project. Tức là, trong từng use case cụ thể, nhiều project trong data center chia sẻ cùng tài nguyên vật lý (physical server, physical storage, physical network). Mỗi project được gán cho một số tài nguyên chẳng hạn như virtual machine, virtual storage, virtual network. Các tài nguyên này tách biệt với các project khác, trừ khi được cho phép bởi một chính sách bảo mật. Các virtual network trong các data center cũng có thể được kết nối đến một mạng vật lý layer-3 hoặc layer-2.

Đối với use case Network Function Virtualization (NFV) thì liên quan đến việc điều phối và quản lý các chức năng của mạng chẳng hạn như firewall, các hệ thống phát hiện xâm nhập hoặc dự phòng - Intrusion Detection or Prevention Systems (IDS/IPS), Deep Packet Inspection (DPI), caching, Wide Area Network (WAN) optimization, etc. và tất cả các tính năng này đều được thực hiện trong các virtual machine thay vì các trong các thiết bị phần cứng vật lý.
## 1.2. OpenContrail Controller và vRouter
OpenContrail gồm hai thành phần chính: OpenContrail Controller và OpenContrail vRouter.

OpenContrail Controller là trung tâm điều khiển logic tập trung nhưng phân tán về mặt các thành phần vật lý. Nó chịu trách nhiệm cung cấp việc quản lý, điều khiển và phân tích các chức năng trong virtual network.

OpenContrail vRouter là một forwarding plane (được hiểu như là các thiết bị mạng ảo như virtual router, virtual switch) chạy trên nền tảng ảo hóa của các server ảo hóa. Thành phần này mở rộng mạng từ các router và các switch vật lý trong data center vào một virtual overlay network nằm trong các server ảo hóa (khái niệm về một overlay network được giải thích trong phần sau). OpenContrail vRouter về cơ bản có khái niệm tương tự như các virtual switch thương mại hoặc opensource witch hiện nay chẳng hạn như OVS, nhưng nó cũng hỗ trợ việc cung cấp routing và các dịch vụ tầng cao (tức là nâng cao lên thành virtual Router, thay vì virtual switch như OVS).

OpenContrail controller cung cấp việc tập trung control plane và managemet plane trong kiến trúc SDN của các hệ thống và thực hiện điều phối các vRouter.
## 1.3. Virtual network
Virtual network (VN) là một khái niệm chính trong hệ thống OpenContrail. Virtual network là một cấu trúc logic được xây dựng trên nền của mạng vật lý (physical network). Các virtual network này được sử dụng để tạo ra các mạng VLAN tách biệt và được sử dụng bởi các project trong cùng một data center. Do vậy, các project sẽ sự dụng các mạng khác nhau và tăng tính bảo mật giữa các project. Mỗi project hoặc một ứng dụng có thể có một hoặc nhiều virtual network. Mỗi virtual network được tách biệt với tất cả các mạng khác, trừ khi nó được cho phép kết nối với các mạng khác.

Virtual network có thể được kết nối đến và được mở rộng thông qua các mạng vật lý sử dụng một router trong hạ tầng vật lý chẳng hạn như Multi-Protocol Label Switching (MPLS - mạng sử dụng giao thức định tuyến thông qua các label), Layer 3 Virtual Prviate Network(L3VPN) và Ethernet Virtual Private Network(EVPN).

Các virtual network cũng được để implement Network Function Virtualization (NFV) và chuỗi các dịch vụ. Vậy làm thế nào được được các mục đích này khi sử dụng virtual network, chúng ta sẽ tìm hiểu chi tiết trong phần sau.

## 1.4. Overlay Networking
Virtual network có thể được implement theo nhiều cơ chế khác nhau. Ví dụ, mỗi virtual cloud có thể được implement như là một Virtual Local Area Network (VLAN), Virtual Private Network (VPN),...

Virutal Network cũng có thể được implement sử dụng hai mạng - một mạng vật lý cơ bản bên dưới (a physical underlay network) và một virtual overlay network. Kỹ thuật overlay network này thực ra đã được triển khai rộng rãi trong ngành công nghiệp mạng LAN không dây (wireless) hơn một nửa thập kỷ qua, tuy nhiên ứng dụng của kỹ thuật này trong các mạng của data center thì vẫn là một khái niệm khá mới mẻ. Kỹ thuật này cũng đang dần được tiêu chuẩn hóa trong các diễn đàn khác nhau, và được implement trong các project opensource và các sản phẩm network virtualization của nhiều nhà cung cấp.

**Quyền** của physical underlay network là cung cấp một "IP fabric" - Trách nhiệm của IP này là cung cấp một kết nối điểm điểm (unicast IP) từ bất kỳ một thiết bị vật lý nào (server, storage device, router, or switch) đến một thiết bị vật lý khác. Một underlay network lý tưởng là sẽ cung cấp một kết nối có độ trễ thấp, non-blocking, high-bandwidth giữa hai điểm bất kỳ trong mạng.

Các virtual Router đang chạy trong các hypersivor của các server ảo hóa tạo ra một virtual overlay network trên nền tảng của physical underlay network sử dụng một lưới các "tunnel" giữa các router này. Trong trường hợp của OpenContrail, những tunnel này có thể là MPLS qua GRE/UDP tunnel, hoặc là VXLAN tunnel.

Các Router và các Switch vật lý ở tầng physical underlay network sẽ không chứa bất kỳ một trạng thái nào của mỗi project trong data center: Tức là chúng sẽ không chứa bất kỳ một địa chỉ MAC, địa chỉ IP, hoặc là các chính sách của các virtual machine. Bảng forwarding trong các router và switch của physical underlay network chỉ chứa địa chỉ IP và địa chỉ MAC của các server vật lý. Các router gateway hoặc các switch gateway được dùng để kết nối một virtual network đến một physical network là một ngoại lệ - **[can't understand ....]**

Các virtual router, nói cách khác, được sử dụng để lưu trữ trạng thái của mỗi project. Mỗi vRouter chứa một bảng forwarding riêng biệt (một routing instance) cho một virtual network. Bảng forwarding này sẽ chứa địa chỉ IP (được sử dụng cho layer-3 overlay) hoặc địa chỉ MAC (trong trường hợp layer 2 overlay) của các virtual machine trong mỗi virtual network. Không có một vRouter nào chứa tất cả địa chỉ IP và địa chỉ MAC của tất cả virtual machine trong toàn bộ data center. Một vRouter chỉ cần chứa các routing instance **[can't understand....]**

## 1.5. Overlay dựa trên MPLS L3VPN và EVPN
Các giao thức control plane và các giao thức data plane khác nhau cho các overlay network được sử dụng tùy thuộc vào yêu cầu của các vendor và các tổ chức.

Cho ví dụ, đối với chuẩn IETF VXLAN draft đưa ra một cơ chế đóng gói data plane mới và đưa ra một control plane có hành động "flood and learn source address" tương tự như chuẩn Ethernet trong việc gán vào bảng forwarding và yêu cầu một hoặc nhiều multicast group trong underlay network để implement hành động flooding.

Hệ thống OpenContrail rất giống với chuẩn MPLS L3VPN (đối với layer 3 overlay) và MPLS EVPN (đối với layer 2 overlay).

Trong data plane, OpenContrail hỗ trợ MPLS thông qua GRE, một cơ chế đóng gói data plane được hỗ trợ rộng rãi bởi các router hiện nay. OpenContrail cũng hỗ trợ một tiêu chuẩn đóng gói khác data plane khác chẳng hạn như MPLS thông qua UDP và VXLAN. Các tiêu chuẩn đóng gói chẳng hạn NVGRE có thể dễ dàng được thêm vào trong các phiên bản tiếp theo.

Control plane protocol giữa các node control plane của hệ thống OpenContrail hoặc một physical gateway router (hoặc switch) được sử dụng là BGP (và netconf cho management). Đây là một control plane protocol được sử dụng cho MPLS L3VPN và MPLS EVPN.

Giao thức giữa OpenContrail controller và OpenContrail vRouter dựa trên XMPP.

Thực tế hệ thống OpenContrail sử dụng các giao thức control plane và data plane tương tự như các giao thức được sử dụng MPLS L3VPN và MPLS EVPN mà có nhiều tính năng nâng cao - Những công nghệ này đã đạt được kiểm chứng và có thể scale, chúng được triển khai rộng rãi trong các mạng của các vendor.

## 1.6. OpenContrail và Open Source
OpenContrail được thiết kế để hoạt động trong một môi trường opensource cloud. Cung cấp một giải pháp end-to-end được tích hợp một cách đầy đủ:
- Hệ thống OpenContrail được tích hợp với opensource hypervisor chẳng hạn như KVM hoặc XEN.
- Hệ thống OpenContrail được tích hợp với opensource virtualization orchestration system chẳng hạn như OpenStack và CloudStack.
- Hệ thống OpenContrail được tích hợp với các hệ thống opensource quản lý server vật lý chẳng hạn như chef, puppet, cobbler và ganglia.

OpenContrail được cấp phép theo giấy phép của Apache 2.0. Juniper Network đã hỗ trợ một phiên bản thương mại của OpenContrail System.
## 1.7. Scale-out Architecture và High Availability
Như chúng ta đã nhắc tới ở trên, OpenContrail Controller là được tập trung hóa nhưng các thành phần vật lý được phân tán (Physically distributed).

Physically distributed có nghĩa là hệ thống OpenContrail bao gồm nhiều kiểu node khác nhau, mỗi node có thể có nhiều instance phục vụ cho khả năng high availability và horizontal scaling. Những node instance này có thể là các server vật lý hoặc là virtual machine. Đối với một triển khai đơn giản nhất, nhiều kiểu node có thể được implement trong cùng một server. Có 3 kiểu node:
- Các node cấu hình (configuration node) chịu trách nhiệm cho management layer. Những node này cung cấp một REST API có thể được sử dụng để cấu hình hệ thống hoặc trích xuất trạng thái họat động của hệ thống. Các dịch vụ được khởi tạo được đại diện bởi các đối tượng trong một database có khả năng mở rộng theo chiều ngang mà được mô tả bởi một mô hình dữ liệu (chi tiết về các mô hình dữ liệu sẽ được nói đến chi tiết ở phần sau). Configuration node này cũng chứa một transformation engine (đôi khi được nói đến như là một complier), được sử dụng để vận chuyển các đối tượng trong mô hình dữ liệu ở dịch vụ high-level vào các đối tượng lower-level tương ứng trong mô hình dữ liệu. Trong khi mô hình dữ liệu của dịch vụ high-level (high-level service data model) mô tả **những gì** dịch vụ cần để được implement, thì mô hình dữ liệu của dịch vụ low-level lại mô tả **cách thức** mà các service này cần để implement. Configuration node này publish nội dung của mô hình dữ liệu low-level đến node control sử dụng interface cho giao thức Metadata Access Points (IF-MAP).
- Control Node được sử dụng để implement phần logic được tập trung hóa (logically centralized) của control plane. Không phải tất cả tính năng của control plane là logically centrailzed - một số tính năng control plane vẫn được implement theo kiểu phân tán trong các router và switch và/hoặc physical và virtual trong mạng. Các node control sử dụng giao thức IF-MAP để monitor nội dung của low-level technology data model mà được tính toán bởi các node configuration (để mô tả trạng thái mong muốn của mạng). Các node control sử dụng kết hợp các giao thức south-bound để thực hiện các chức năng của nó, ví dụ như tạo ra trạng thái thực tế của mạng giống với trạng thái mong muốn của mạng (như trong mô tả của low-level technology data model). Trong version đầu tiên của OpenContrail System các giao thức south-bound này bao gồm Extensible Messaging and Presence Protocol (XMPP) để điều khiển các OpenContrail vRouter, cũng như kết hợp giữa Border Gateway Protocol (BGP) và Network Configuration (Netconf) Protocol để điều khiển các router vật lý. Các node control cũng sử dụng BGP cho việc đồng bộ trạng thái giữa các node control với nhau trong trường hợp cho nhiều node control cho lý do scale-out và high availability.
- Các node analytic chịu trách nhiệm cho việc tập hợp, đối sánh và biểu diễn các thông tin phân tích đối với các vấn đề xử lý lỗi và đối với việc hiểu biết về cách sử dụng mạng. Mỗi thành phần của OpenContrail System tạo ra các record chi tiết cho mỗi sự kiện xảy ra trong hệ thống. Các event record này được gửi đến một trong nhiều instance (trong trường hợp scale-out) của node analytic để đối chiếu và lưu trữ thông tin trong database có khả năng scale theo chiều ngang sử dụng một format được tối ưu cho việc xử lý và truy vấn time-serie. Các node analytic có cơ chế để tự động trigger tập hợp nhiều record khi sự kiện cụ thể xuất hiện; mục đích của việc trigger các record này là để có thể lấy được nguyên nhân bắt đầu của bất kỳ một vấn đề nào mà không cần phải xử lý nó. Các node analytic cung cấp một nouth-bound analytic query REST API.

Physically distributed của OpenContrail Controller là một tính năng riêng biệt. Bởi vì trong trường hợp có nhiều instance dư thừa của bất kỳ loại node nào, hoạt động trong chế độ active-active (trái ngược với chế độ active-standby), hệ thống vẫn có thể họat động mà không có bất kỳ gián đoạn khi một node nào đó bị lỗi. Khi một node bị quá tải, các instance bổ sung có loại node đó có thể được khởi tạo ngay sau đó để chia sẻ công việc với node bị quá tải. Điều này ngăn chặn việc bất kỳ một node độc lập nào trở thành nút thắt cổ chai và cho phép hệ thống có thể quản lý một hệ thống rộng lớn với hàng ngàn server.

Trên đây chúng ta nhắc đến từ logically contralized, nó có nghĩa là OpenContrail Controller hoạt động nhưng một đơn vị logic, mặc dù thực tế OpenContrail được implement trong một cluster của nhiều node.

## 1.8. The Central Role of Data Models: SDN as a Compiler
Các data model thực hiện một quyền trung tâm trong hệ thống OpenContrail. Một data model bao gồm một tập các đối tượng, khả năng của các đối tượng này và mối quan hệ giữa chúng.

Data model cho phép các ứng dụng thể hiện ý nghĩa của chúng trong một cách khai báo chứ không phải là một cách bắt buộc, điều này rất quan trọng trong việc đạt được hiệu quả lập trình cao. Một khía cạnh cơ bản của kiến trúc OpenContrail System là dữ liệu được tính toán bởi platform cũng nhưng bởi các ứng dụng được lưu trữ bởi platform. Do đó, các ứng dụng có thể được xem như là vô trạng thái. Mục đích quan trọng nhất của thiết kế này là các application cá nhân được giải phóng khỏi việc phải lo lắng về sự phức tạp của high availability, scale và peering, tức là đối với các tính năng high availability,... thì platform sẽ chịu trách nhiệm cho việc thực hiện chức năng này, còn các application sẽ chỉ thực hiện chức năng cơ bản của nó.

Có hai kiểu data model: high-level service data model và low-level technology data model. Cả hai kiểu data model này đều được mô tả bằng cách sử dụng ngôn ngữ mô hình hóa dữ liệu thông dụng mà hiện tại dựa trên một cơ chế IF-MAP XML. YANG đang được xem xét cho version tiếp theo.

High-level service data model mô tả trạng thái mong muốn của mạng tại mỗi high level của mạng abstraction, việc sử dụng các đối tượng để ánh xạ trực tiếp đến các dịch vụ được cung cấp đến người dùng cuối cùng (end-user) - ví dụ như một virtual network, hoặc một chính sách kết nối, hoặc một chính sách bảo mật.

Low-level technology data model mô tả trạng thái mong muốn của mạng tại mỗi low level của mạng abstraction, việc sử dụng các đối tượng đề ánh xạ đến các giao thức mạng xác định chẳng hạn như BGP, hoặc là một VXLAN network.

Các configuration node chịu trách nhiệm cho việc vận chuyển mọi thay đổi ở high-level service data model đến một tập tương ứng các thay đổi trong lower-level technology data model. Điều này tương tự với khái niệm của một Just In Time (JIT) complier - cao hơn nữa là thuật ngữ 'SDN as a complier'.

Các control node chịu trách nhiệm cho việc tạo ra trạng thái thực tế giống với trạng thái mong muốn, như được mô tả bởi low-level technology data model sử dụng kết hợp giữa các south-bound protocol ví dụ như XMPP, BGP và NetConf.

## 1.9  North-Bound Application Programming Interfaces
Các node configuration trong OpenContrail Controller cung cấp một nouth-bound REST API đến các hệ thống điều phối và cung cấp dịch vụ. Các nouthbound REST API này mặc định được tạo ra từ high-level data model. Điều này chỉ ra rằng nouthbound REST API sẽ là công cụ để các dịch vụ được cung cấp. Các REST API này được bảo mật thông qua Https cho việc xác thực và mã hóa.
## 1.10. Graphical User Interface
OpenContrail System cũng hỗ trợ một giao diện người dùng (Graphic User Interface - GUI). GUI này được xây dựng hoàn toàn sử dụng các REST APT được nói đến ở phần trước.
## 1.11. An Extensible Platform
Version đầu tiên của OpenContrail System bao gồm một high-level service data model, một low-level technology data model, và một transformation engine để chuyển đổi nhưng thay đổi tương ứng từ high-level service data model đến low-level technology data model. Ngoài ra, phiên bản đầu tiên này cũng chứa một tập các giao thức southbound.

High-level service data model trong phiên bản đầu tiên của OpenContral System là các cấu trúc dịch vụ chẳng hạn như các project, các virtual network, các chính sách kết nối và các chính sách bảo mật. Các đối tượng mô hình này được chọn để hỗ trợ cho các mục đích của các use case trong cloud networking và NFV.

Low-level technology data model được thiết kế đặc biệt để thực hiện các dịch vụ sử dụng overlay network.

Transformation engine trong các node configuration chứa "compiler" để chuyển đổi high-level service data model đến low-level technology data model.

Tập hợp các nouthbound protocol trong các node control là BGP, netconf, XMPP.

OpenContrail System là một nền tảng có khả năng mở rộng, có nghĩa là bất cứ thành phần nào được nói đến ở trên đều có thể được mở rộng để hỗ trợ các use case mới hoặc/và các công nghệ mạng mới:
- High-level service data model có thể được mở rộng bằng cách thêm các đối tượng dịch vụ để đại diện cho các dịch vụ mới chẳng hạn như traffic engineering và bandwidth calendaring các core network của các nhà cung cấp dịch vụ.
- Low-level service data model cũng được mở rộng theo một trong hai lý do. Thứ nhất là high-level service tương ứng được implement sử dụng một công nghệ khác; ví dụ multi-tenancy được implement sử dụng VLAN thay vì overlay. Thứ hai là một high-level service mới được giới thiệu yêu cầu một low-level service mới.
- Transformation engine cũng được mở rộng hoặc để ánh xạ các đối tượng high-level service đến một đối tượng low-level technology service mới (vd: một cách mới để implement một dịch vụ hiện tại) hoặc là để ánh xạ một đối tượng high-level service mới đến các đối tượng low-level service đang có (vd: implement một service mới).

Các giao thức southboud cũng có thể được thêm vào trong node control. Việc thêm giao thức mới này là cần thiết để hỗ trợ một loại thiết bị mạng mới nào đó mà yêu cầu một giao thức mới.
## OpenContrail Architecture Details
Như thể hiện trong **hình 1**, OpenContrail System bao gồm hai phần: một controller và một tập các vRouter thực hiện chức năng của một thành phần forwarding được implement trong các hypervisor (vd: KVM,XEN,...) trong các server ảo hóa.

![](http://www.opencontrail.org/wp-content/uploads/2014/10/Figure01.png)

Như trong hình trên ta có thể thấy, thành phần controller cung cấp các nouthbound REST API được sử dụng bởi các application. Các API được sử dụng để tích hợp với các hệ thống điều phối cloud chẳng hạn như OpenStack thông qua Neutron plugin. Các REST API này cũng được sử dụng bởi các ứng dụng khác. Nói chúng, các REST API này được sử dụng để implement các web-base GUI, tương tự như thành phần GUI trong hệ thống OpenContrail.

OpenContrail cung cấp ba interface: một tập các nouthbound REST API để nói chuyện với các hệ thống điều phối cloud và các ứng dụng, các southbound interface để nói chuyện với các thành phần mạng ảo (vRouter) hoặc các thành phần vật lý (physical router/switch), và các east-west interface để giao tiếp giữa các controller (trong tính năng high availability và scale-out). Cụ thể, hệ thống điều phối cloud sử dụng nouthbound REST API là OpenStack, XMPP là southbound inteface để giao tiếp với các vRouter, BGP là east-west bound interface để giao tiếp với các controller khác,  BGP và Netconf là các southbound interface để giao tiếp với các thành phần vật lý.

Bên trong controller cũng bao gồm bao thành phần sau:
- Configuration node: chịu trách nhiệm cho việc transtale các đối tượng high-level service data model vào các đối tượng low-level service phù hợp để tương tác với các thành phần mạng.
- Control node: chịu trách nhiệm cho việc truyền trạng thái low level này đến và nhận từ các thành phần mạng và các hệ thống ngang hàng trong một cách đồng nhất.
- Analytic node: chịu trách nhiệm cho việc bắt các gói dữ liệu real-time từ các thiết bị mạng, trừu tượng dữ liệu đó và trình diễn dữ liệu trong các mô hình phù hợp để các application sử dụng.

Các vRouter phục vụ chức năng như một thành phần mạng và được implement hoàn toàn trong phần mềm. Chúng chịu trách nhiệm cho việc forwarding các gói tin từ VM này đến VM khác thông qua một tập hợp các tunnel (tunnel - sử dụng trong overlay network, được hiểu như là các đường hầm logic nối giữa các server chứa các VM). Các tunnel này hình thành một mạng overlay được triển khai trên top của một mạng vật lý (physical IP-over-Ethernet network). Mỗi vRouter gồm có 2 phần: một user space agent để implement control plane và một kennel module để implement forwarding engine.

OpenContrail System được implement với ba khối cơ bản sau:
- multi-tenancy, được biết đến như là Network Virtualization, là tính năng cho phép tạo ra các virtual network cung cấp các Closed User Group (CUGs) đến tập hợp các VM.
- Gateway function, khối này có chức năng kết nối các virtual network đến physical network thông qua một gateway router, có khả năng gắn các non-virtualized server (nhưng server không có ảo hóa) hoặc các networking service đến virtual network thông qua một gateway router.
- Service chaning, hay còn gọi là NFV, đây là khối có tính năng điều khiển các luồng traffic thông qua một chuỗi các virtual hoặc physical networking service chẳng hạn như Firewall, load balance,...

## 2.1.1. Node
Bây giờ, chúng ta sẽ chuyển qua cấu trúc bên trong của hệ thống. Như trong hình 2, hệ thống được implement như là một tập các node kết hợp với nhau chạy trên các server. Mỗi node có thể được implement như là một physical server hoặc cũng có thể được implement như là một VM.

Tất cả các node trong cùng một loại cùng chạy trong một cấu hình active-active (có nghĩa là các node này luôn hoạt động song song với nhau) vì vậy sẽ không xảy ra hiện tượng nút thắt cổ chai. Thiết kế scale-out này vừa cung cấp khả năng dự phòng vừa cung cấp khả năng mở rộng theo chiều ngang.
- Configuration node giữa một bản copy lưu vào đĩa của trạng thái cấu hình dự kiến và chuyển đổi high-level service data model đến các low-level technology phù hợp để tương tác với các thành phần mạng. Tất cả chúng được lưu trong một NoSQL database.
- Control node implement một logically centrailzed control plane, chịu trách nhiệm cho việc duy trì trạng thái mạng tạm thời. Tương tác với các node mạng khác và các thành phần mạng khác để đảm bảo trạng thái mạng cuối cùng phù hợp với trạng thái mong muốn trong configuration node.
- Analytic node được implement để lưu trữ, tập hợp và phân tích thông tin của các thành phần mạng, virtual hoặc physical. Những thông tin này bao gồm log, thông số, sự kiên, và lỗi.

Ngoài các loại node trong OpenContrail System, chúng ta cũng định nghĩa một số loại node khác đối với các server vật lý và các thành phần mạng vật lý thực hiện các nhiệm vụ cụ thể trong hệ sinh thái OpenContrail System:
- Compute node là các server vật lý có hỗ trợ ảo hóa để tạo ra các VM. Các VM này có thể là các VM trong một project đang chạy một số application nào đó, hoặc các VM này cũng có thể là có node để implement các dịch vụ mạng chẳng hạn như firewall, load balance. Mỗi node compute chứa **một vRouter** để implement forwarding plane và các thành phần phân tán của control plane (được nói đến ở trên, là các phần được implement trên các virtual/physical router và switch để thực hiện điều khiển forwarding plane).
- Gateway node là các physical gateway router hoặc switch được sử đụng để kết nối các virtual network của các project đến physical network (chẳng hạn như Internet), hoặc là kết nối đến một mạng VPN, hoặc là một hạ tầng vật lý khác.
- Service node là các thành phần mạng vật lý được dùng để cung cấp các dịch vụ mạng chẳng hạn như Firewall, load balance. Chuỗi các service có thể chứa hỗn hợp cả virtual service (được implement trên các VM), và các physical service (nằm trên các service node).

Để phân biệt rõ các node bổ sung này, trong hình sau sẽ không thể hiện các router và swith vật lý mà tạo ra underlay IP over Ethernet network (mạng vật lý bên dưới). Bên cạnh đó còn có một interface từ mỗi node trong hệ thống đến node analytic cho việc monitor và các interface cũng không được thể hiện ở đây.
![](http://www.opencontrail.org/wp-content/uploads/2014/10/Figure02.png)

Như trong hình trên và cũng được nói đến ở phần trước, các node tương tác với nhau sử dụng các interface, là các giao thức nouthbound, southbound và east-west. Cụ thể, Control Node tương tác với Compute Node, Gateway Node và Service Node thông qua southbound interface là các giao thức XMPP, BGP và Netconf. Các node trong cùng kiểu sử dụng các giao thức east-west là IBGP để tương tác với nhau để tạo ra mô hình high availability và scalable. Configuration node và Control Node tương tác với nhau sử dụng giao thức IF-MAP để truyền các đối tượng low-level technology từ configuration node đến control node. Analytic node tương tác với các application hoặc oschestration system thông qua nouthbound REST API.

Tiếp theo, chúng ta sẽ đi vào tìm hiểu cụ thể các node bổ sung này.
## 2.1.2. Compute node
Compute node là các server vật lý hỗ trợ ảo hóa để tạo ra các VM. Các VM này có thể là các VM trong một project nào đấy đang chạy một ứng dụng nào đấy chẳng hạn như một web server, database server, hoặc là một ứng dụng doanh nghiệp. Hoặc các VM này có thể là các virtual service node được sử dụng để implement các dịch vụ mạng như Firewall, load balance,... để tạo ra chuỗi các dịch vụ mạng. Cấu hình chuẩn của compute node là sử dụng hệ điều hành nhân linux và hypervisor là KVM or XEN. Ngoài ra, thành phần vRouter forwarding plane nằm trong linux kernel; và thành phần vRouter agent là local control plane (là phần distributed của control plane). Cấu trúc này được thể hiện trong hình 3 dưới đây.

Các hệ điều hành khác và các nền tảng ảo hóa khác như VMware có thể được hỗ trợ trong các phiên bản tiếp theo.

![](http://www.opencontrail.org/wp-content/uploads/2014/10/Figure03.png)

Hai khối được xây dựng trong compute node là để implement một vRouter là: vRouter agent và vRouter forwarding plane. Chúng được mô tả trong phần tiếp theo.
## 2.1.2.1. vRouter agent
vRouter agent là một user space process (là không gian xử lý được điều khiển bởi người dùng) đang chạy trong linux. Nó hoạt động như local control plane và chịu trách nhiệm đối với các chức năng sau:
- Trao đổi trạng thái điều khiển chẳng hạn như trao đổi các route entry với Control node thông qua XMPP.
- Nhận các trạng thái cấu hình low-level (chính là các đối tượng low-level service data mode mô tả các công nghệ sẽ sử dụng trong mạng) như là các routing instance và forwarding policy từ Control node trong qua XMPP.
- Báo cáo trạng thái các thông số chẳng hạn như log, event, error, thông số đến các analytic node.
- Khám phá sự tồn tại và thuộc tính của các VM được tạo ra bởi Nova Agent trong OpenStack.
- Thực hiện các chính sách forwarding đối với các gói tin đầu tiên của những flow mới (mà không phù hợp với bất kỳ một flow entry nào trong thành phần kernel module) và cài đặt để thêm một flow entry trong flow table của forwarding plane.
- Thực hiện các chức năng DHCP, ARP, DNS và MDNS proxy.

Mỗi vRouter agent được kết nối đến tối thiểu hai node control cho cơ chế dự phòng trong chế độ dự phòng active-active.
## 2.1.2.2. vRouter forwarding plane
Thành phần vRouter forwarding plane hoạt động như một kernel module trong linux, và thực hiện các chức năng sau:
- Đóng gói các gói tin được gửi đi ra overlay network và tách các gói tin nhận được từ overlay network.
- Gán các gói tin đến một routing instance:
 - Các gói tin nhận được từ overlay network được gán đến một routing instance dựa trên MPLS label (label này được sử dụng trong giao thức MPLS - giao thức định tuyến thông qua label) hoặc Virtual Network Indentifer (VNI).
 - Các virtual interface của các local virtual machine được gán đến các routing instance.
 - Việc tìm kiếm địa chỉ địa (Destinaiton address) trong Forwarding Information Base (FIB) và forwarding gói tin đến đúng đích. Định tuyến có thể là IP address layer 3 hoặc MAC address layer 2.
 - Tùy chọn, thực hiện chính sách forwarding sử dụng một flow table:
   - Đối chiếu các gói tin phù hợp với gói tin đến một flow table và áp dụng các hành động tương ứng với flow phù hợp với gói tin.
   - Chuyển các gói tin không phù hợp với bất kỳ một flow entry nào trong flow table đến vRouter agent để tạo ra một flow entry mới cho packet.
   - Chuyển các gói tin cụ thể như gói tin DHCP, ARP, MDNS đến vRouter agent.

Hình 4 dưới đây sẽ thể hiện cấu trúc bên trong của thành phần vRouter Forwarding Plane.
![](http://www.opencontrail.org/wp-content/uploads/2014/10/Figure04.png)

Forwarding plane hỗ trợ MPLS thông qua cơ chế đóng gói packet theo GRE/UDP và VXLAN trong overlay network. Forwarding plane hỗ trợ cả layer-3 forwarding theo địa chỉ IP và layer-2 forwarding theo địa chỉ MAC. Hiện tại thì vRouter forwarding plane chỉ hỗ trợ IPv4.
## 2.1.3. Control node
Hình 5 thể hiện cấu trúc bên trong của một control node.

Control node giao tiếp với các control node khác với với các loại node khác trong hệ sinh thái OpenContrail System. Cụ thể như sau:
- Control node nhận trạng thái cấu hình từ configuration node sử dụng IF-MAP.
- Control node trao đổi các định tuyến (route entry) với các control node khác sử dụng IBGP để đảm bảo rằng tất cả các control node đều có cùng một trạng thái mạng.
- Control node trao đổi các định tuyến (route entry) với các vRouter agent trên các compute node sử dụng XMPP. Chúng cũng sử dụng XMPP để gửi trạng thái cấu hình chẳng hạn như routing instance và forwarding policy đến các vRouter agent.
- Controle node cũng thực hiện proxy đối với mội số traffic cụ thể chẳng hạn như DHCP, ARP, MDNS. Các proxy request này cũng được nhận từ compute node thông qua XMPP.
- Control node trao đổi các định tuyến với các gateway node (các router hoặc switch vật lý) sử dụng BGP. Chúng cũng gửi các configuration state sử dụng Netconf.

![](http://www.opencontrail.org/wp-content/uploads/2014/10/Figure05.png)
## 2.1.4. Configuration node
Hình 6 thể hiện cấu trúc bên trong của một configuration node. Configuration node giao tiếp với Oschestration system thông qua REST API, với các configuration node khác thông qua cơ chế đồng bộ được phân tán, và với control node thông qua IF-MAP.

Configuration node cũng cung cấp một dịch vụ khám phá (discovery service) mà các client có thể sử dụng để xác định vị trí của nhà cung cấp dịch vụ (ví dụ, xác định node cung cấp một dịch vụ cụ thể nào đó). Một ví dụ cụ thể, khi một vRouter agent trong một compute node muốn kết nối đến một control node (cụ thể hơn là một cặp active-active Control node implement trên hai VM), nó sẽ sử dụng discovery service để khám phá địa chỉ IP của control node. Client sẽ sử dụng local configuration, DHCP hoặc DNS để xác định vị trí dịch vụ discovery service.

Configuration node chứa các thành phần sau:
- Một REST API server cung cấp  north-bound interface đến các oschestration system hoặc các application. Các interface này được sử dụng để cài đặt configuration state sử dụng hig-level data model.
- Một Redis message bus là message queue để cung cấp kết nối giữa các thành phần bên trong configuration node.
- Một database Cassandra cho việc lưu trữ trên đĩa các configuration. Cassandra là database fault-tolerant và horizontally scalable ( chịu lỗi tốt và có khả năng mở rộng theo chiều ngang).
- Một cơ chế transform để chuyển đổi những thay đổi trong high-level data model đến các thay đổi tương ứng trong low-level data model.
- Một IF-MAP server cung cấp một south-bound interface để truyền các low-level configuration xuống control node.
- Zookeeper được sử dụng để cấp phát định danh duy nhất cho các đối tượng và thực hiện các giao dịch. Zookeeper không được thể hiện trong hình sau.

![](http://www.opencontrail.org/wp-content/uploads/2014/10/Figure06.png)

## 2.1.5. Analytic node
Hình 7 thể hiện cấu trúc bên trong của một analytic node. Một analytic node giao tiếp với các application sử dụng một north-bound REST API, với các analytic node khác sử dụng một cơ chế đồng bộ phân tán, và với các thành phần trong các configuration node và control node sử dụng một giao thức dựa trên XML được gọi là **Sandesh** được sử dụng để xử lý cho các dữ liệu khối lượng lớn.

Analytic node chứa các thành phần sau:
- Một Conllector đển trao đổi các message Sandesh (sẽ được mô tả chi tiết trong phần sau) với các thành phần trong configuration node và control node để tập hợp các thông tin cho việc phân tích.
- Một NoSQL database để lưu trữ các thông tin này.
- Một rule engine để tự động thu thập trạng thái hoạt động khi có một sự kiện cụ thể xuất hiện.
- Một REST API server để cung cấp các north-bound interface cho việc truy vấn database và nhận các trạng thái hoạt động.
- Một Query engine để thực thi các truy vấn được nhận bởi north-bound interface. Engine này cung cấp khả năng truy cập linh hoạt đối với các database với khối lượng lớn dữ liệu.

![](http://www.opencontrail.org/wp-content/uploads/2014/10/Figure07.png)

Sandesh áp dụng cho hai kiểu message: Các message không đồng bộ được nhận bởi các analytic node cho mục đích report log, event, error; và các message đồng bộ trong đó một analytic node có thể gửi các request và nhận các response để thu thập các trạng thái hoạt động cụ thể.

Tất cả thông tin được thu thập bởi thành phần Conllector được lưu trữ trên đĩa trong NoSQL database.

Các analytic node cung cấp một northbound REST API để cho phép các ứng dụng client có thể submit các truy vấn.

Các analytic node cung cấp cơ chế **scatter-gather** được gọi là "aggregation". Có nghĩa là một GET request (từ CLI của một client) sẽ được ánh xạ đến một tập các request message và các kết quả của các request message này sẽ được kết hợp lại để trả về cho client application.

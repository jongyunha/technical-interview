# TCP/IP 프로토콜에 대한 기술 면접 답변

## 주요 질문
"TCP/IP 프로토콜의 동작 방식과 각 계층별 역할에 대해 설명해주세요. TCP와 UDP의 차이점은 무엇인가요?"

## 1. TCP/IP 계층 구조

### 1.1 4계층 구조
```
[애플리케이션 계층]  - HTTP, FTP, SMTP, DNS
         ↓
[전송 계층]         - TCP, UDP
         ↓
[인터넷 계층]        - IP, ICMP, ARP
         ↓
[네트워크 접근 계층]  - Ethernet, Wi-Fi
```

### 1.2 TCP 3-way Handshake
```
[Client]                      [Server]
   |                             |
   |------- SYN (seq=x) ------->|
   |                             |
   |<-- SYN-ACK (seq=y,ack=x+1) |
   |                             |
   |------ ACK (ack=y+1) ------>|
   |                             |
```

## 2. TCP vs UDP 비교

### 2.1 TCP (Transmission Control Protocol)
```java
// TCP 소켓 프로그래밍 예시
public class TCPServer {
    public static void main(String[] args) {
        try (ServerSocket serverSocket = new ServerSocket(8080)) {
            Socket clientSocket = serverSocket.accept();
            BufferedReader in = new BufferedReader(
                new InputStreamReader(clientSocket.getInputStream()));
            PrintWriter out = new PrintWriter(clientSocket.getOutputStream(), true);
            
            String input = in.readLine();
            out.println("서버 응답: " + input);
        }
    }
}
```

특징:
- 연결 지향적 (Connection-oriented)
- 신뢰성 있는 데이터 전송
- 흐름 제어와 혼잡 제어
- 순서 보장

### 2.2 UDP (User Datagram Protocol)
```java
// UDP 소켓 프로그래밍 예시
public class UDPServer {
    public static void main(String[] args) {
        try (DatagramSocket socket = new DatagramSocket(8080)) {
            byte[] buffer = new byte[1024];
            DatagramPacket packet = new DatagramPacket(buffer, buffer.length);
            
            socket.receive(packet);
            String received = new String(packet.getData(), 0, packet.getLength());
            
            // 응답 전송
            byte[] response = "서버 응답".getBytes();
            DatagramPacket responsePacket = new DatagramPacket(
                response, response.length, 
                packet.getAddress(), packet.getPort());
            socket.send(responsePacket);
        }
    }
}
```

특징:
- 비연결형 (Connectionless)
- 신뢰성 없는 데이터 전송
- 빠른 전송 속도
- 순서 보장되지 않음

## 3. TCP 흐름 제어와 혼잡 제어

### 3.1 흐름 제어 (Flow Control)
```
[송신측]                         [수신측]
   |                               |
   |--- 데이터 (Window Size=1000)-->|
   |                               |
   |<---- ACK (Window Size=800) ---|
   |                               |
```

### 3.2 혼잡 제어 (Congestion Control)
```java
public class TCPCongestionControl {
    private int cwnd;  // Congestion Window
    private int ssthresh;  // Slow Start Threshold
    
    // Slow Start
    public void slowStart() {
        if (cwnd < ssthresh) {
            cwnd *= 2;  // 지수적 증가
        } else {
            congestionAvoidance();
        }
    }
    
    // Congestion Avoidance
    public void congestionAvoidance() {
        cwnd += 1;  // 선형적 증가
    }
}
```

## 4. IP 주소와 서브넷

### 4.1 IPv4 주소 체계
```java
public class IPAddress {
    public static boolean isValidIPv4(String ipAddress) {
        String[] parts = ipAddress.split("\\.");
        if (parts.length != 4) return false;
        
        for (String part : parts) {
            try {
                int value = Integer.parseInt(part);
                if (value < 0 || value > 255) return false;
            } catch (NumberFormatException e) {
                return false;
            }
        }
        return true;
    }
}
```

### 4.2 서브넷 마스크
```java
public class SubnetCalculator {
    public static String calculateNetworkAddress(
            String ipAddress, String subnetMask) {
        String[] ipParts = ipAddress.split("\\.");
        String[] maskParts = subnetMask.split("\\.");
        StringBuilder networkAddress = new StringBuilder();
        
        for (int i = 0; i < 4; i++) {
            int ip = Integer.parseInt(ipParts[i]);
            int mask = Integer.parseInt(maskParts[i]);
            networkAddress.append(ip & mask);
            if (i < 3) networkAddress.append(".");
        }
        
        return networkAddress.toString();
    }
}
```

## 근거 자료

### 1. RFC (Request for Comments) 문서
- [RFC 793 - TCP](https://datatracker.ietf.org/doc/html/rfc793)
- [RFC 768 - UDP](https://datatracker.ietf.org/doc/html/rfc768)
- [RFC 791 - IP](https://datatracker.ietf.org/doc/html/rfc791)

### 2. 네트워킹 교과서
- [Computer Networks (Tanenbaum)](https://www.pearson.com/en-us/subject-catalog/p/computer-networks/P200000003179)
- [TCP/IP Illustrated](https://www.informit.com/store/tcp-ip-illustrated-volume-1-the-protocols-9780321336316)

### 3. 온라인 자료
- [IETF (Internet Engineering Task Force)](https://www.ietf.org/)
- [Cisco Networking Academy](https://www.netacad.com/)

### 4. 실무 자료
- [Wireshark Documentation](https://www.wireshark.org/docs/)
- [tcpdump Documentation](https://www.tcpdump.org/manpages/tcpdump.1.html)

## 실무 관련 추가 질문

1. "TCP Keepalive는 언제 사용하며, 어떤 장단점이 있나요?"

2. "TCP의 TIME_WAIT 상태는 왜 필요하며, 어떻게 관리해야 할까요?"

3. "네트워크 성능 튜닝을 위해 어떤 설정들을 고려해야 하나요?"

4. "TCP와 UDP 중 어떤 상황에서 어떤 프로토콜을 선택하시나요?"

## 실제 사용 예시

### 소켓 타임아웃 설정
```java
public class SocketTimeoutExample {
    public void configureSocket() throws SocketException {
        Socket socket = new Socket();
        
        // 연결 타임아웃 설정
        socket.connect(new InetSocketAddress("host", 8080), 3000);
        
        // 읽기 타임아웃 설정
        socket.setSoTimeout(3000);
        
        // TCP Keepalive 설정
        socket.setKeepAlive(true);
        
        // TCP_NODELAY 설정 (Nagle 알고리즘 비활성화)
        socket.setTcpNoDelay(true);
    }
}
```

### 네트워크 모니터링
```java
public class NetworkMonitor {
    public void monitorNetwork() throws IOException {
        NetworkInterface networkInterface = 
            NetworkInterface.getByName("eth0");
            
        // MTU 확인
        System.out.println("MTU: " + networkInterface.getMTU());
        
        // IP 주소 확인
        networkInterface.getInetAddresses().forEachRemaining(address -> {
            System.out.println("IP: " + address.getHostAddress());
        });
        
        // 네트워크 상태 확인
        System.out.println("Up? " + networkInterface.isUp());
        System.out.println("Loopback? " + networkInterface.isLoopback());
        System.out.println("Multicast? " + networkInterface.supportsMulticast());
    }
}
```

실제 면접에서는 이론적인 지식과 함께 실제 프로젝트에서 네트워크 관련 문제를 해결한 경험을 구체적으로 설명하는 것이 좋습니다.
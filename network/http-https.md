# HTTP와 HTTPS에 대한 기술 면접 답변

## 주요 질문
"HTTP와 HTTPS의 차이점을 설명해주세요. HTTPS는 어떻게 보안을 제공하나요?"

## 1. HTTP와 HTTPS 기본 개념

### 1.1 HTTP (Hypertext Transfer Protocol)
```
[Client]        [Plain Text]        [Server]
    | -------- HTTP Request -------> |
    |                                |
    | <------- HTTP Response ------- |
```

특징:
- 평문 통신 (암호화되지 않음)
- 기본 포트: 80
- 빠른 통신 속도
- 중간자 공격에 취약

### 1.2 HTTPS (HTTP Secure)
```
[Client]     [Encrypted Data]     [Server]
    | ----- SSL/TLS Handshake -----> |
    | <---- Exchange Certificates --- |
    | ----- Establish Secure Key ---> |
    |                                 |
    | ---- Encrypted HTTP Traffic --> |
```

특징:
- SSL/TLS를 통한 암호화 통신
- 기본 포트: 443
- 데이터 무결성 보장
- 신뢰성 있는 통신

## 2. HTTPS 동작 방식

### 2.1 SSL/TLS Handshake 과정
```java
public class TLSHandshake {
    public void explainHandshake() {
        /*
        1. ClientHello
           - 지원하는 암호화 suite 목록
           - 클라이언트 랜덤 값
        
        2. ServerHello
           - 선택된 암호화 suite
           - 서버 랜덤 값
           - 서버 인증서
        
        3. 클라이언트 인증서 확인
           - 인증서 유효성 검증
           - 인증 기관(CA) 확인
        
        4. Key Exchange
           - Pre-master secret 생성
           - 공개키로 암호화하여 전송
        
        5. Session Key 생성
           - 클라이언트/서버 랜덤 값과
           - Pre-master secret으로
           - 대칭키 생성
        */
    }
}
```

### 2.2 인증서 검증 과정
```java
public class CertificateVerification {
    public boolean verifyCertificate(X509Certificate cert) 
            throws CertificateException {
        try {
            // 1. 인증서 만료 여부 확인
            cert.checkValidity();
            
            // 2. 발급자 확인
            String issuer = cert.getIssuerDN().getName();
            
            // 3. 서명 검증
            PublicKey publicKey = cert.getPublicKey();
            cert.verify(publicKey);
            
            return true;
        } catch (Exception e) {
            return false;
        }
    }
}
```

## 3. 보안 요소

### 3.1 대칭키/비대칭키 암호화
```java
public class EncryptionExample {
    // 대칭키 암호화 예시
    public byte[] symmetricEncrypt(String data, SecretKey key) 
            throws Exception {
        Cipher cipher = Cipher.getInstance("AES");
        cipher.init(Cipher.ENCRYPT_MODE, key);
        return cipher.doFinal(data.getBytes());
    }
    
    // 비대칭키 암호화 예시
    public byte[] asymmetricEncrypt(String data, PublicKey publicKey) 
            throws Exception {
        Cipher cipher = Cipher.getInstance("RSA");
        cipher.init(Cipher.ENCRYPT_MODE, publicKey);
        return cipher.doFinal(data.getBytes());
    }
}
```

### 3.2 인증서 관리
```java
public class CertificateManager {
    private KeyStore keyStore;
    
    public void loadCertificate(String keystorePath, String password) 
            throws Exception {
        keyStore = KeyStore.getInstance("JKS");
        try (FileInputStream fis = new FileInputStream(keystorePath)) {
            keyStore.load(fis, password.toCharArray());
        }
    }
    
    public X509Certificate getCertificate(String alias) 
            throws Exception {
        return (X509Certificate) keyStore.getCertificate(alias);
    }
}
```

## 4. 실제 구현 예시

### 4.1 HTTPS 서버 구현
```java
public class HTTPSServer {
    public void startServer() throws Exception {
        // SSL 컨텍스트 설정
        SSLContext sslContext = SSLContext.getInstance("TLS");
        KeyManagerFactory kmf = KeyManagerFactory.getInstance(
            KeyManagerFactory.getDefaultAlgorithm());
        KeyStore ks = KeyStore.getInstance("JKS");
        
        // 키스토어 로드
        ks.load(new FileInputStream("keystore.jks"), "password".toCharArray());
        kmf.init(ks, "password".toCharArray());
        sslContext.init(kmf.getKeyManagers(), null, null);
        
        // HTTPS 서버 시작
        HttpsServer server = HttpsServer.create(
            new InetSocketAddress(8443), 0);
        server.setHttpsConfigurator(
            new HttpsConfigurator(sslContext));
        
        server.createContext("/", new HttpHandler() {
            @Override
            public void handle(HttpExchange exchange) throws IOException {
                String response = "Secure Hello!";
                exchange.sendResponseHeaders(200, response.length());
                try (OutputStream os = exchange.getResponseBody()) {
                    os.write(response.getBytes());
                }
            }
        });
        
        server.start();
    }
}
```

### 4.2 HTTPS 클라이언트 구현
```java
public class HTTPSClient {
    public String makeSecureRequest(String url) throws Exception {
        // SSL 컨텍스트 설정
        SSLContext sslContext = SSLContext.getInstance("TLS");
        sslContext.init(null, getTrustManagers(), new SecureRandom());
        
        // HTTPS 연결 설정
        HttpsURLConnection conn = (HttpsURLConnection) 
            new URL(url).openConnection();
        conn.setSSLSocketFactory(sslContext.getSocketFactory());
        
        // 응답 읽기
        try (BufferedReader br = new BufferedReader(
                new InputStreamReader(conn.getInputStream()))) {
            return br.lines().collect(Collectors.joining("\n"));
        }
    }
    
    private TrustManager[] getTrustManagers() {
        return new TrustManager[] {
            new X509TrustManager() {
                public X509Certificate[] getAcceptedIssuers() { return null; }
                public void checkClientTrusted(X509Certificate[] certs, String authType) {}
                public void checkServerTrusted(X509Certificate[] certs, String authType) {}
            }
        };
    }
}
```

## 근거 자료

### 1. 표준 문서
- [RFC 2616 - HTTP/1.1](https://datatracker.ietf.org/doc/html/rfc2616)
- [RFC 8446 - TLS 1.3](https://datatracker.ietf.org/doc/html/rfc8446)
- [RFC 5246 - TLS 1.2](https://datatracker.ietf.org/doc/html/rfc5246)

### 2. 기술 문서
- [OWASP HTTPS Best Practices](https://cheatsheetseries.owasp.org/cheatsheets/Transport_Layer_Protection_Cheat_Sheet.html)
- [Mozilla SSL Configuration Generator](https://ssl-config.mozilla.org/)

### 3. 보안 관련 자료
- [Let's Encrypt Documentation](https://letsencrypt.org/docs/)
- [SSL Labs Best Practices](https://github.com/ssllabs/research/wiki/SSL-and-TLS-Deployment-Best-Practices)

## 실무 관련 추가 질문

1. "HTTPS 적용 시 성능 이슈는 어떻게 해결하시나요?"

2. "인증서 갱신은 어떻게 관리하시나요?"

3. "Mixed Content 문제는 어떻게 해결하시나요?"

4. "TLS 1.0/1.1을 비활성화해야 하는 이유는 무엇인가요?"

실제 면접에서는 이론적인 지식과 함께 HTTPS 구축 경험, 인증서 관리 경험, 보안 이슈 해결 경험 등을 구체적으로 설명하는 것이 좋습니다.
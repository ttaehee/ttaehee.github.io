---
title: 암호화를 통한 API 통신 및 데이터베이스 저장 과정의 보안 강화
excerpt: encryption algorithm (RSA4096, AES256)
---

<br/>

이번에 새롭게 배포되는 기능에서는 앱과 웹 내에서 개인 민감 정보(주민등록번호 등)를 받는 플로우가 포함되어 있었다     
이에 따라 API 통신 및 데이터베이스 저장 과정에서의 보안 강화가 필수적이었다        
최종적으로 API 통신 보안에는 RSA4096을 적용을, 데이터베이스 저장 보안에는 AES256을 적용하였는데 이를 정리해보겠다      

<br/>

## API 통신 보안 :: 비대칭키 암호화 방식
사용자 데이터를 안전하게 보호하기 위해 통신 과정에서 발생할 수 있는 탈취 등의 외부 공격을 방어하는 것이 중요했다     
단순히 데이터 탈취 자체도 경계해야 하지만, 혹시나 탈취되더라도 복호화가 불가하도록 비대칭키 암호화 방식을 선택했다       
(server에서 복호화를 통해 입력값 유효성 검증을 해야했기에 단방향 암호화는 제외하였다)     

<br/>

비대칭키 암호화는 공개키와 개인키를 이용하여 데이터를 암호화하고 복호화하는 방식으로, 중간자 공격을 차단하는 데 효과적이다   
중간자 공격(man-in-the-middle attack) 같은 상황에서도, 공격자가 암호화된 데이터를 가로채는 것은 가능하지만, 개인키 없이는 의미 있는 데이터를 해석할 수 없기 때문에 탈취된 정보는 쓸모가 없게 된다   

- 공개키 암호화 : client는 server에서 제공된 공개키를 이용해 데이터를 암호화하여 전송
- 개인키 복호화 : server만이 개인키를 통해 이를 복호화 가능

대칭키 방식에 비해 암복호화 연산에 자원 소모가 더 크고 키의 관리가 복잡하지만, 해당 api 에서는 보안의 우선순위가 더 크다고 판단하였고 그 중에서도 필요한 데이터에만 비대칭 암호화를 적용하여 효율성을 유지하면서 보안을 확보하기로 하였다   

<br/>

### 비대칭 암호화 알고리즘   
비대칭 암호화 알고리즘에는 RSA(Rivest Shamir Adleman), ECC(Elliptic Curve Cryptography), DSA(Digital Signature Algorithm) 등이 있다    
일단 DSA는 서명에 최적화된 알고리즘이고 검증에 중점을 두고 있어 이번 목적에 적합하지 않았다       
ECC는 RSA보다 짧은 키 길이로 동일한 보안성을 제공해 자원 효율성에선 좋지만 RSA에 비해 상대적으로 최근에 사용되기 시작하였다      

<br/>

위의 이유로 좀 더 긴 기간동안 여러 보안 공격에 대한 저항력이 입증된 RSA를 최종적으로 선택하였다 (보안은 안전하게 가야한다는 주의)      
또한 서버 자원이 충분하고 해당 API는 빈번하게 호출되지 않는 최종 단계에서만 사용되기 때문에 성능은 충분하다고 판단했다   

<br/>

### RSA (Rivest-Shamir-Adleman)   
RSA 는 큰 수의 소인수분해 수를 알기 어렵다는 것에 기반한 알고리즘이다 (말이 좀 이상한가)    

<br/>

키 생성 알고리즘
- 서로 다른 두 소수 p, g
  - ex) p = 7, q = 13 
- N = p * q
  - ex) N = 7 * 13 = 91
- K = (p-1) * (q-1)
  - ex) K = (7-1) * (13-1) = 72
- K와 최대공약수가 1인 수(서로소인 수) e (1 < e < K) 찾기
  - ex) K와 서로소인 수 e = 5, 7, 11... 11 선택
- e*d를 K로 나눈 나머지가 1이 되는 d 찾기
  - ex) 11 * d % 72 = 1인 d = 57, 131... 57 선택
- N과 e를 공개한다 공개키: [ N, e ]
  - ex) [ 91, 11 ] 
- d는 키 생성자가 보관한다 개인키: [ N, d]
  - ex) [ 91, 57 ]

<br/>

암/복호화
- 암호화(암호문 생성) : C = P^e mod N
- 복호화(평문 생성) : P = C^d mod N      

<br/>

공개키 [ N, e ]로 부터 비밀키 d를 알아내기 위해서는, N으로 부터 K를 구하면 d를 쉽게 알 수 있다 = 개인키를 알아내려면 N의 소인수분해가 필요하다     
따라서, RSA의 안전성은 소수 p와 q의 선택에 달려 있으며, 이를 위해 2048비트 또는 4096비트 키 길이가 권장된다    
2048비트 키 길이가 오늘날의 표준이지만, 컴퓨터 연산 능력이 계속 향상됨에 따라 비밀키를 유추하는 것이 쉬워지고 있어 4096비트 키를 선택했다   

<br/>

더 긴 키 길이는 암호화 및 복호화에 추가적인 연산을 요구하지만, 위에서 말한것처럼 API 호출 빈도가 낮기 때문에 연산이 자주 이루어지지 않아 성능 저하가 체감될 가능성이 적다         
또한, 서버 자원이 충분한 경우 이러한 추가 연산을 감당할 수 있어 성능에 미치는 영향이 크지 않으므로 더 긴 키 길이가 문제가 되지 않는다고 판단했다      
실제로 해당 api 요청 테스트 시, 처리 속도가 크게 차이나지 않았다    

<br/>

### 코드 구현

```java
@Getter
@Component
public class RsaCipher {
    String cipherTransformation = "RSA/ECB/OAEPWithSHA-1AndMGF1Padding";

    private final PublicKey publicKey;
    private final PrivateKey privateKey;

    RsaCipher(RsaConfig rsaConfig) throws NoSuchAlgorithmException, InvalidKeySpecException {
        this.publicKey = getPublicKeyFromBase64Encrypted(rsaConfig.getPublicKeyText());
        this.privateKey = getPrivateKeyFromBase64Encrypted(rsaConfig.getPrivateKeyText());
    }

    private PublicKey getPublicKeyFromBase64Encrypted(String base64PublicKey)
        throws NoSuchAlgorithmException, InvalidKeySpecException {
        base64PublicKey = base64PublicKey.replace("-----BEGIN PUBLIC KEY-----", "")
            .replaceAll("\\n", "")
            .replace("-----END PUBLIC KEY-----", "");

        byte[] decodedBase64PubKey = Base64.getDecoder().decode(base64PublicKey);
        X509EncodedKeySpec keySpec = new X509EncodedKeySpec(decodedBase64PubKey);

        PublicKey publicKey = KeyFactory.getInstance("RSA")
            .generatePublic(keySpec);
        return publicKey;
    }

    private PrivateKey getPrivateKeyFromBase64Encrypted(String base64PrivateKey)
        throws NoSuchAlgorithmException, InvalidKeySpecException {
        base64PrivateKey = base64PrivateKey.replace("-----BEGIN RSA PRIVATE KEY-----", "")
            .replaceAll("\\n", "")
            .replace("-----END RSA PRIVATE KEY-----", "");
        byte[] decodedBase64PrivateKey = Base64.getDecoder().decode(base64PrivateKey);

        PrivateKey privateKey = KeyFactory.getInstance("RSA")
            .generatePrivate(new PKCS8EncodedKeySpec(decodedBase64PrivateKey));
        return privateKey;
    }

    /**
     * 4096비트 RSA 키쌍을 생성
     */
    public KeyPair genRSAKeyPair(int keySize) throws NoSuchAlgorithmException {
        KeyPairGenerator gen = KeyPairGenerator.getInstance("RSA");
        gen.initialize(keySize, new SecureRandom());
        return gen.genKeyPair();
    }

    /**
     * Public Key로 RSA 암호화를 수행
     */
    public String encrypt(String plainText) {
        try {
            Cipher cipher = Cipher.getInstance(cipherTransformation);
            cipher.init(Cipher.ENCRYPT_MODE, publicKey);
            byte[] bytePlain = cipher.doFinal(plainText.getBytes());
            return Base64.getEncoder().encodeToString(bytePlain);
        } catch (Exception e) {
            throw new CustomException(CustomErrorCode.ENCRYPTION_FAIL);
        }
    }

    /**
     * Private Key로 RSA 복호화를 수행
     */
    public String decrypt(String encrypted) {
        try {
            Cipher cipher = Cipher.getInstance(cipherTransformation);
            byte[] byteEncrypted = Base64.getDecoder().decode(encrypted.getBytes());

            cipher.init(Cipher.DECRYPT_MODE, privateKey);
            byte[] bytePlain = cipher.doFinal(byteEncrypted);
            return new String(bytePlain, StandardCharsets.UTF_8);
        } catch (Exception e) {
            throw new CustomException(CustomErrorCode.DECRYPTION_FAIL);
        }
    }
}
```

처음 잡은 틀은 요정도   
'-----BEGIN PUBLIC KEY-----' 요거는 따로 관리하게 수정할까,, 근데 사실 잘 안바뀔거같은 문구인데 훔    

<br/><br/>

## 데이터베이스 저장 보안 :: 대칭키 암호화 방식   
입력받은 고객 민감정보를 데이터베이스에도 저장해야했기에 암호화 방식을 도입했다        
이 과정에서 AES256을 사용하여 데이터를 암호화한 후 저장했다   

<br/>

AES256은 대칭키 암호화 방식으로 암호화와 복호화에 같은 키를 사용하는 방식이다  
RDS는 private vpn에 위치해있어 내부 네트워크를 통해 접근되기도 하고, 데이터 조회와 검색까지 용이하도록 비대칭이 아닌 대칭 암호화를 사용하였다    
(admin에서는 복호화를 통해 조회가 가능해야했기에 단방향 암호화는 제외하였다)   

<br/>

암호화 해두는 데이터로도 검색이 가능해야했어서ㅠ 요거는 AES 사용하니까 검색어를 암호화한 후 찾았다    
그래서 LIKE 검색은 불가    

<br/>

### 대칭 암호화 알고리즘   
대표적으로 AES(Advanced Encryption Standard), DES(Data Encryption Standard), 3DES(Triple DES) 등이 있다    

<br/>

DES는 과거에 널리 사용된 대칭 암호화 알고리즘으로, 56비트 키 길이를 사용한다   
현재는 키 길이가 짧아 보안상 취약하므로 더 이상 안전하다고 여겨지지 않아 선택지에서 제외하였다    
3DES는 DES를 세 번 적용하여 보안성을 높인 알고리즘으로 168비트 키 길이를 제공한다     
그러나 AES에 비해 보안성이 떨어지고 처리 속도도 느리다고 알려져 있어 선택지에서 제외하였다      

<br/>  

AES는 128비트, 192비트, 256비트 키 길이를 지원하며, 미국 정부의 공식 암호화 표준으로 채택었다       
빠르고 안전하며, 다양한 응용 프로그램에서 사용되는것으로 알려져있다      
마찬가지로 키 길이가 짧을수록 성능이 좋겠지만, 192와 256 비트 사용으로 암복호화 테스트를 했을때 처리 시간에는 유의미한 큰차이가 없었다    
그래서 AES256으로 결정    

<br/>

### AES (Advanced Encryption Standard) 
AES 알고리즘은 대칭키 암호화 방식 중 하나로, 고정된 크기(암호화되는 데이터 블록의 크기는 항상 128비트)의 데이터 블록을 여러 단계에 걸쳐 암호화하는 블록 암호 방식이다    
AES256은 256비트 키를 사용하여 데이터를 암호화함으로써, 높은 보안성을 제공하는 암호화 방식이다       
데이터베이스에서 데이터를 가져올 때는 해당 데이터를 AES256으로 복호화하여 원본 데이터를 얻을 수 있다   
빠른 속도로 암호화와 복호화를 수행할 수 있으며, 성능에 미치는 영향이 적으면서도 강력한 보안성을 유지한다    

데이터가 암호화된 상태로 저장되기 때문에, 만약 데이터베이스가 유출되더라도 데이터를 직접적으로 이용할 수 없기 때문에 AES256은 대규모 데이터를 안전하게 보호하는 데 적합해서 이번 목적에 딱이었다       

<br/>

### 코드 구현

```java
public class Aes256Util {
    private static final String SECRET_KEY = "";
    private static final String SALT = "";
    private static final byte[] iv = {0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0};

    public static String encrypt(String plainText) {
        try {
            IvParameterSpec ivSpec = new IvParameterSpec(iv);

            SecretKeyFactory factory = SecretKeyFactory.getInstance("PBKDF2WithHmacSHA256");
            KeySpec spec = new PBEKeySpec(SECRET_KEY.toCharArray(), SALT.getBytes(), 65536, 256);
            SecretKey tmp = factory.generateSecret(spec);
            SecretKeySpec secretKey = new SecretKeySpec(tmp.getEncoded(), "AES");

            Cipher cipher = Cipher.getInstance("AES/CBC/PKCS5Padding");
            cipher.init(Cipher.ENCRYPT_MODE, secretKey, ivSpec);
            return Base64.getEncoder()
                .encodeToString(cipher.doFinal(plainText.getBytes(StandardCharsets.UTF_8)));
        } catch (Exception e) {
            throw new CustomException(CustomErrorCode.ENCRYPTION_FAIL);
        }
    }

    public static String decrypt(String encryptText) {
        try {
            IvParameterSpec ivSpec = new IvParameterSpec(iv);

            SecretKeyFactory factory = SecretKeyFactory.getInstance("PBKDF2WithHmacSHA256");
            KeySpec spec = new PBEKeySpec(SECRET_KEY.toCharArray(), SALT.getBytes(), 65536, 256);
            SecretKey tmp = factory.generateSecret(spec);
            SecretKeySpec secretKey = new SecretKeySpec(tmp.getEncoded(), "AES");

            Cipher cipher = Cipher.getInstance("AES/CBC/PKCS5PADDING");
            cipher.init(Cipher.DECRYPT_MODE, secretKey, ivSpec);
            return new String(cipher.doFinal(Base64.getDecoder().decode(encryptText)));
        } catch (Exception e) {
            throw new CustomException(CustomErrorCode.DECRYPTION_FAIL);
        }
    }
}
```

처음 잡은 틀은 요정도   
SECRET_KEY와 SALT는 별도 설정파일에서 관리하도록 수정하고, IV(Initialization Vector)는 SecureRandom을 이용해서 암호화할 때마다 랜덤으로 생성하고 함께 저장하도록 추가 작업하였다              

<br/><br/>

## 결론    
API 통신과 데이터베이스 저장 과정에서 각각 RSA4096과 AES256을 적용하여 전체 시스템의 보안성을 강화하였다   
중간자 공격에 대비하여 RSA4096을 통해 데이터의 안전한 전달을 보장하고 네트워크 상에서 데이터가 가로채지더라도 유의미한 정보를 추출할 수 없다     
또한, 데이터베이스에 저장된 정보는 AES256으로 암호화되어 있어 외부 침입자가 데이터베이스에 접근하더라도 복호화 키 없이는 해당 데이터를 사용할 수 없다    
(키 관리를 잘 하자!)  

<br/><br/>

Reference     
- [ICPA::RSA](https://www.youtube.com/watch?v=kGUlfVpIfaQ)
- [Veritas::RSA](https://www.veritas.com/ko/kr/information-center/rsa-encryption)
- [Veritas::AES](https://www.veritas.com/ko/kr/information-center/aes-encryption)
- [NIST FIPS PUB 197: Announcing the Advanced Encryption Standard (AES)](https://nvlpubs.nist.gov/nistpubs/FIPS/NIST.FIPS.197.pdf)

<br/>

---
title: AES-128-CBC Base64加密——OC,Java,Golang联调
date: 2018-11-21 02:17:28
tags:
    - AES-128-CBC
    - Objective-C
    - Java
    - Golang
---
# AES-128-CBC
这里首先说说AES加密原理
AES加密算法采用分组密码体制，每个分组数据的长度为128位16个字节，密钥长度可以是128位16个字节、192位或256位，一共有四种加密模式（ECB、CBC、CFB、OFB），我们通常采用需要初始向量IV的CBC模式，初始向量的长度规定是128位16个字节。另外就是Padding，这里面有大坑。。。。先说一下Padding的三种模式PKCS5、PKCS7和NOPADDING。PKCS5是指分组数据缺少几个字节，就在数据的末尾填充几个字节的几，比如缺少5个字节，就在末尾填充5个字节的5。PKCS7是指分组数据缺少几个字节，就在数据的末尾填充几个字节的0，比如缺少7个字节，就在末尾填充7个字节的0。NoPadding是指不需要填充，也就是说数据的发送方肯定会保证最后一段数据也正好是16个字节。而PKCS5如果正好是16个字节且最后是16的时候则会再填充16个16用来区分，PKC7则是为0时填充16个0。而在iOS的OC方法里压根没提供PKCS5，只有PKCS7更坑的是真正对接时发现iOS上的PKCS7和其他端PKCS5是一样的。。。。所以才有了现在的想法分享一下踩过的坑，具体啥原因恐怕只有苹果自家知道，系统方法是真的坑！Java可以直接用系统方法填好设置结束战斗。。。Go的话padding这块自己写实现其他的系统都能设置。最后说一下密钥长度这里只有iOS是要自己设置好位数再对应位数写密钥，其他平台直接对应位数写密钥即可，所以最好各平台自己在封装下判断密钥长度出事向量长度，不然各端对应起来还是要犯傻。
<!-- more -->
# Base64
下面说一下Base64，这个也是个坑，iOS系统提供的base64可选类型压根就不是已知领域常用的，正常是padding和websafe，padding会填充=，而websafe则会替换"+"为"-"，"\\"为"_"
而iOS提供的则是下边的，完全不常用的。。。
```
NSDataBase64Encoding64CharacterLineLength      其作用是将生成的Base64字符串按照64个字符长度进行等分换行。  
NSDataBase64Encoding76CharacterLineLength      其作用是将生成的Base64字符串按照76个字符长度进行等分换行。  
NSDataBase64EncodingEndLineWithCarriageReturn  其作用是将生成的Base64字符串以回车结束。  
NSDataBase64EncodingEndLineWithLineFeed        其作用是将生成的Base64字符串以换行结束。  
```
基本上GTMBase64用定了，然后还要扩展一下padding设置，原版只是把websafe模式开放了padding设置，内部其实有对应逻辑只需要自己加个方法调用一下即可。下面就是添加的和微改的两个方法
```
+(NSString *)stringByEncodingData:(NSData *)data padded:(BOOL)padded{
NSString *result = nil;
NSData *converted = [self baseEncode:[data bytes]
length:[data length]
charset:kBase64EncodeChars
padded:padded];
if (converted) {
result = [[[NSString alloc] initWithData:converted
encoding:NSUTF8StringEncoding] autorelease];
}
return result;
}
+(NSData *)decodeString:(NSString *)string {
NSData *result = nil;
NSData *data = [string dataUsingEncoding:NSUTF8StringEncoding];
if (data) {
result = [self baseDecode:[data bytes]
length:[data length]
charset:kBase64DecodeChars
requirePadding:NO];
}
return result;
}
```
至于Java，android开发很爽直接用android.util.base64,里面直接可以设置nopadding和websafe等，而纯Java用java.util.base64就要自己写替换逻辑，具体代码见源码部分
最后说一下Go直接系统方法提供完美解决
```
base64.StdEncoding
base64.URLEncoding        websafe模式
base64.RawStdEncoding    nopadding
base64.RawURLEncoding    websafe模式nopadding
```
# AES-128-CBC +Base64-Nopadding源码
下面就是3中语言分别实现 AES-128-CBC +Base64-Nopadding，从编码体验和对应上很明显Java最清晰，Go要自己写点东西，OC则是连对应对和正常理解范围内有偏差。
## OC
```Objective-C
#import <Foundation/Foundation.h>
#import <CommonCrypto/CommonCryptor.h>

@interface NSData (Encryption)

- (NSData *)AES128EncryptWithKey:(NSString *)key Iv:(NSString *)Iv;   //加密
- (NSData *)AES128DecryptWithKey:(NSString *)key Iv:(NSString *)Iv;   //解密

@end

@implementation NSData (Encryption)

//(key和iv向量这里是16位的) 这里是CBC加密模式，安全性更高

- (NSData *)AES128EncryptWithKey:(NSString *)key Iv:(NSString *)Iv{//加密
// 'key' should be 32 bytes for AES128, will be null-padded otherwise
char keyPtr[kCCKeySizeAES128+1]; // room for terminator (unused)
bzero(keyPtr, sizeof(keyPtr)); // fill with zeroes (for padding)

// fetch key data
[key getCString:keyPtr maxLength:sizeof(keyPtr) encoding:NSUTF8StringEncoding];


char ivPtr[kCCKeySizeAES128+1];
memset(ivPtr, 0, sizeof(ivPtr));
[Iv getCString:ivPtr maxLength:sizeof(ivPtr) encoding:NSUTF8StringEncoding];

NSUInteger dataLength = [self length];

//See the doc: For block ciphers, the output size will always be less than or
//equal to the input size plus the size of one block.
//That's why we need to add the size of one block here
size_t bufferSize = dataLength + kCCBlockSizeAES128;
void *buffer = malloc(bufferSize);

size_t numBytesEncrypted = 0;
CCCryptorStatus cryptStatus = CCCrypt(kCCEncrypt, kCCAlgorithmAES128, kCCOptionPKCS7Padding,
keyPtr, kCCKeySizeAES128,
ivPtr /* initialization vector (optional) */,
[self bytes], dataLength, /* input */
buffer, bufferSize, /* output */
&numBytesEncrypted);
if (cryptStatus == kCCSuccess) {
//the returned NSData takes ownership of the buffer and will free it on deallocation
return [NSData dataWithBytesNoCopy:buffer length:numBytesEncrypted];
}

free(buffer); //free the buffer;
return nil;
}


- (NSData *)AES128DecryptWithKey:(NSString *)key Iv:(NSString *)Iv{//解密
char keyPtr[kCCKeySizeAES128+1]; // room for terminator (unused)
bzero(keyPtr, sizeof(keyPtr)); // fill with zeroes (for padding)

// fetch key data
[key getCString:keyPtr maxLength:sizeof(keyPtr) encoding:NSUTF8StringEncoding];

char ivPtr[kCCKeySizeAES128+1];
memset(ivPtr, 0, sizeof(ivPtr));
[Iv getCString:ivPtr maxLength:sizeof(ivPtr) encoding:NSUTF8StringEncoding];

NSUInteger dataLength = [self length];

//See the doc: For block ciphers, the output size will always be less than or
//equal to the input size plus the size of one block.
//That's why we need to add the size of one block here
size_t bufferSize = dataLength + kCCBlockSizeAES128;
void *buffer = malloc(bufferSize);

size_t numBytesDecrypted = 0;
CCCryptorStatus cryptStatus = CCCrypt(kCCDecrypt, kCCAlgorithmAES128, kCCOptionPKCS7Padding,
keyPtr, kCCKeySizeAES128,
ivPtr /* initialization vector (optional) */,
[self bytes], dataLength, /* input */
buffer, bufferSize, /* output */
&numBytesDecrypted);

if (cryptStatus == kCCSuccess) {
//the returned NSData takes ownership of the buffer and will free it on deallocation
return [NSData dataWithBytesNoCopy:buffer length:numBytesDecrypted];
}

free(buffer); //free the buffer;
return nil;
}

@end

@interface SecurityCore
+ (NSString*)encryptAESString:(NSString*)string;
+ (NSString*)decryptAESString:(NSString*)string;
@end

@implementation SecurityCore

#pragma mark - AES加密
const NSString * skey=@"dde4b1f8a9e6b814"
const NSString * ivParameter =@"dde4b1f8a9e6b814"

//将string转成带密码的data
+(NSString*)encryptAESString:(NSString*)string
{
//将nsstring转化为nsdata
NSData *data = [string dataUsingEncoding:NSUTF8StringEncoding];
//使用密码对nsdata进行加密
NSData *encryptedData = [data AES128EncryptWithKey:skey Iv:ivParameter];
NSString *encryptedString=[GTMBase64 stringByEncodingData:encryptedData padded:NO];

return encryptedString;
}

+ (NSString*)decryptAESString:(NSString*)string{


//将nsstring转化为nsdata
NSData *data = [GTMBase64 decodeString:string];

NSData *decryptData = [data AES128DecryptWithKey:skey Iv:ivParameter];

NSString *str = [[NSString alloc] initWithData:decryptData encoding:NSUTF8StringEncoding];
return [str autorelease];
}

@end

```
## Java
```Java
import javax.crypto.Cipher;
import javax.crypto.spec.IvParameterSpec;
import javax.crypto.spec.SecretKeySpec;

import java.util.Base64;


public class SecurityCore {
/*
* 加密用的Key 可以用26个字母和数字组成 此处使用AES-128-CBC加密模式，key需要为16位。
*/
private String sKey        = "dde4b1f8a9e6b814";
private String ivParameter = "dde4b1f8a9e6b814";
private static SecurityCore instance = null;

private SecurityCore() {

}

public static SecurityCore getInstance() {
if (instance == null)
instance = new SecurityCore();
return instance;
}

public static String webSafeBase64StringEncoding(byte[] sSrc,boolean padded) throws Exception {
String encodeString=Base64.getEncoder().encodeToString(sSrc);// 此处使用BASE64做转码。

//websafe base64
encodeString=encodeString.replace("+","-");
encodeString=encodeString.replace("/","_");

//nopadding base64
if (!padded) {
if (encodeString.endsWith("=")) {
encodeString = encodeString.substring(0, encodeString.length() - 1);
if (encodeString.endsWith("=")) {
encodeString = encodeString.substring(0, encodeString.length() - 1);
}
}
}
return encodeString;
}

public static byte[] webSafeBase64StringDecoding(String sSrc) throws Exception {
//websafe base64
sSrc=sSrc.replace("-","+");
sSrc=sSrc.replace("_","/");

return Base64.getDecoder().decode(sSrc);
}

public static String base64StringEncoding(byte[] sSrc,boolean padded) throws Exception {
String encodeString=Base64.getEncoder().encodeToString(sSrc);// 此处使用BASE64做转码。

//nopadding base64
if (!padded) {
if (encodeString.endsWith("=")) {
encodeString = encodeString.substring(0, encodeString.length() - 1);
if (encodeString.endsWith("=")) {
encodeString = encodeString.substring(0, encodeString.length() - 1);
}
}
}
return encodeString;
}

public static byte[] base64StringDecoding(String sSrc) throws Exception {
return Base64.getDecoder().decode(sSrc);
}

public static byte[] AES128CBCStringEncoding(String encData ,String secretKey,String vector) throws Exception {

if(secretKey == null) {
return null;
}
if(secretKey.length() != 16) {
return null;
}
if (vector != null && vector.length() != 16) {
return null;
}
Cipher cipher = Cipher.getInstance("AES/CBC/PKCS5Padding");
byte[] raw = secretKey.getBytes();
SecretKeySpec skeySpec = new SecretKeySpec(raw, "AES");
IvParameterSpec iv = new IvParameterSpec(vector.getBytes());// 使用CBC模式，需要一个向量iv，可增加加密算法的强度
cipher.init(Cipher.ENCRYPT_MODE, skeySpec, iv);
byte[] encrypted = cipher.doFinal(encData.getBytes("utf-8"));

return encrypted;
}

public static String AES128CBCStringDecoding(byte[] sSrc,String key,String ivs) throws Exception {
try {
if(key == null) {
return null;
}
if(key.length() != 16) {
return null;
}
if (ivs != null && ivs.length() != 16) {
return null;
}
byte[] raw = key.getBytes("ASCII");
SecretKeySpec skeySpec = new SecretKeySpec(raw, "AES");
Cipher cipher = Cipher.getInstance("AES/CBC/PKCS5Padding");
IvParameterSpec iv = new IvParameterSpec(ivs.getBytes());
cipher.init(Cipher.DECRYPT_MODE, skeySpec, iv);
byte[] original = cipher.doFinal(sSrc);
String originalString = new String(original, "utf-8");
return originalString;
} catch (Exception ex) {
return null;
}
}


// 加密
public String encrypt(String sSrc) throws Exception {
try {
String encodeString=base64StringEncoding(AES128CBCStringEncoding(sSrc,sKey,ivParameter),false);

return encodeString;
} catch (Exception ex) {
return null;
}
}

// 解密
public String decrypt(String sSrc) throws Exception {
try {
String decodeString=AES128CBCStringDecoding(base64StringDecoding(sSrc),sKey,ivParameter);
return decodeString;
} catch (Exception ex) {
return null;
}
}

//test
public static void main(String[] args) throws Exception {
// 需要加密的字串
String cSrc = "123";

// 加密
long lStart = System.currentTimeMillis();
String enString = SecurityCore.getInstance().encrypt(cSrc);
System.out.println("加密后的字串是：" + enString);

long lUseTime = System.currentTimeMillis() - lStart;
System.out.println("加密耗时：" + lUseTime + "毫秒");
// 解密
lStart = System.currentTimeMillis();
String DeString = SecurityCore.getInstance().decrypt(enString);
System.out.println("解密后的字串是：" + DeString);
lUseTime = System.currentTimeMillis() - lStart;
System.out.println("解密耗时：" + lUseTime + "毫秒");
}
}
```
# Golang
``` Golang
package main
import(
"fmt"
"crypto/aes"
"crypto/cipher"
"encoding/base64"
"bytes"
)

const (
sKey         = "dde4b1f8a9e6b814"
ivParameter     = "dde4b1f8a9e6b814"
)

/加密
func PswEncrypt(src string)(string){
key := []byte(sKey)
iv := []byte(ivParameter)

result, err := Aes128Encrypt([]byte(src), key, iv)
if err != nil {
panic(err)
}
return  base64.RawStdEncoding.EncodeToString(result)
}
//解密
func PswDecrypt(src string)(string) {

key := []byte(sKey)
iv := []byte(ivParameter)

var result []byte
var err error

result,err=base64.RawStdEncoding.DecodeString(src)
if err != nil {
panic(err)
}
origData, err := Aes128Decrypt(result, key, iv)
if err != nil {
panic(err)
}
return string(origData)

}
func Aes128Encrypt(origData, key []byte,IV []byte) ([]byte, error) {
if key == nil || len(key) != 16 {
return nil, nil
}
if IV != nil && len(IV) != 16 {
return nil, nil
}

block, err := aes.NewCipher(key)
if err != nil {
return nil, err
}
blockSize := block.BlockSize()
origData = PKCS5Padding(origData, blockSize)
blockMode := cipher.NewCBCEncrypter(block, IV[:blockSize])
crypted := make([]byte, len(origData))
// 根据CryptBlocks方法的说明，如下方式初始化crypted也可以
blockMode.CryptBlocks(crypted, origData)
return crypted, nil
}

func Aes128Decrypt(crypted, key []byte,IV []byte) ([]byte, error) {
if key == nil || len(key) != 16 {
return nil, nil
}
if IV != nil && len(IV) != 16 {
return nil, nil
}

block, err := aes.NewCipher(key)
if err != nil {
return nil, err
}
blockSize := block.BlockSize()
blockMode := cipher.NewCBCDecrypter(block,IV[:blockSize])
origData := make([]byte, len(crypted))
blockMode.CryptBlocks(origData, crypted)
origData = PKCS5UnPadding(origData)
return origData, nil
}

func PKCS5Padding(ciphertext []byte, blockSize int) []byte {
padding := blockSize - len(ciphertext)%blockSize
padtext := bytes.Repeat([]byte{byte(padding)}, padding)
return append(ciphertext, padtext...)
}

func PKCS5UnPadding(origData []byte) []byte {
length := len(origData)
// 去掉最后一个字节 unpadding 次
unpadding := int(origData[length-1])
return origData[:(length - unpadding)]
}

func main(){
encodingString := PswEncrypt("123")
decodingString := PswDecrypt(encodingString);
fmt.Printf("AES-128-CBC\n加密：%s\n解密：%s\n",encodingString,decodingString)
}
```

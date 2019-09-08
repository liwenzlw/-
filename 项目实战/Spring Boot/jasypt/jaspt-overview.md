[官网](http://www.jasypt.org/)

[GitHub](https://github.com/daichk/jasypt)

示例1：

对密码进行加密并检查密码的正确性

```java
StrongPasswordEncryptor passwordEncryptor = new StrongPasswordEncryptor();
String encryptedPassword = passwordEncryptor.encryptPassword(userPassword);
//校验
if (passwordEncryptor.checkPassword(inputPassword, encryptedPassword)) {
  // correct!
} else {
  // bad login!
}
```



示例二：

对文本进行加密解密

```java
AES256TextEncryptor textEncryptor = new AES256TextEncryptor();
textEncryptor.setPassword(myEncryptionPassword);
String myEncryptedText = textEncryptor.encrypt(myText);
...
String plainText = textEncryptor.decrypt(myEncryptedText);
```


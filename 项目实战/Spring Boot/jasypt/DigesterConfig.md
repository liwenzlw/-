```java

/**
 * Digester(摘要计算器)的通用配置类接口
 * digester对象会在初始化的时候从DigesterConfig中读取配置信息（只会读取一次）
 * 通过实现DigesterConfig可以从其他源中读取配置信息，如 远程服务器，LDAP，数据库等
 */
public interface DigesterConfig {

    /**
     * 返回hashing算法的名称，如 like "MD5" or "SHA-1".
	 * 该算法一定要被JVM支持，并且可以创建该算法的java.security.MessageDigest实例
     * digester对象会忽略null返回值
     * 标准的算法名称可以参照:
	 * http://java.sun.com/j2se/1.5.0/docs/guide/security/CryptoSpec.html
     */
    public String getAlgorithm();

    
    /**
     * 用来计算摘要的salt的字节长度
     * 如果返回值为0，则不会使用salt进行摘要计算
     * digester对象会忽略null返回值
     */
    public Integer getSaltSizeBytes();

    
    /**
     * 返回hashing函数递归的次数  <i>h(h(...h(x)...))</i>
	 * 返回值为null，将不会进行递归hashing
     */
    public Integer getIterations();

    
    /**
     * digester对象使用的{@link SaltGenerator}
     * digester对象会忽略null返回值
     */
    public SaltGenerator getSaltGenerator();


    /**
     * <p>
     * Returns the name of the <tt>java.security.Provider</tt> implementation
     * to be used by the digester for obtaining the digest algorithm. This
     * provider must have been registered beforehand.
     * </p>
     * <p>
     * If this method returns null, the digester will ignore this parameter
     * when deciding the name of the security provider to be used.
     * </p>
     * <p>
     * If this method does not return null, and neither does {@link #getProvider()},
     * <tt>providerName</tt> will be ignored, and the provider object returned
     * by <tt>getProvider()</tt> will be used.
     * </p>
     * 
     * @since 1.3
     * 
     * @return the name of the security provider to be used.
     */
    public String getProviderName();

    
    /**
     * <p>
     * Returns the <tt>java.security.Provider</tt> implementation object
     * to be used by the digester for obtaining the digest algorithm.
     * </p>
     * <p>
     * If this method returns null, the digester will ignore this parameter
     * when deciding the security provider object to be used.
     * </p>
     * <p>
     * If this method does not return null, and neither does {@link #getProviderName()},
     * <tt>providerName</tt> will be ignored, and the provider object returned
     * by <tt>getProvider()</tt> will be used.
     * </p>
     * <p>
     * The provider returned by this method <b>does not need to be
     * registered beforehand<b>, and its use will not result in its 
     * being registered.
     * </p>
     * 
     * @since 1.3
     * 
     * @return the security provider object to be asked for the digest
     *         algorithm.
     */
    public Provider getProvider();
    
    
    /**
     * <p>
     * Returns <tt>Boolean.TRUE</tt> if the salt bytes are to be appended after the 
     * message ones before performing the digest operation on the whole. The 
     * default behaviour is to insert those bytes before the message bytes, but 
     * setting this configuration item to <tt>true</tt> allows compatibility 
     * with some external systems and specifications (e.g. LDAP {SSHA}).
     * </p>
     * 
     * @since 1.7
     * 
     * @return whether salt will be appended after the message before applying 
     *         the digest operation on the whole, instead of inserted before it
     *         (which is the default). If null is returned, the default 
     *         behaviour will be applied.
     */
    public Boolean getInvertPositionOfSaltInMessageBeforeDigesting();
    
    
    /**
     * <p>
     * Returns <tt>Boolean.TRUE</tt> if the plain (not hashed) salt bytes are to 
     * be appended after the digest operation result bytes. The default behaviour is 
     * to insert them before the digest result, but setting this configuration 
     * item to <tt>true</tt> allows compatibility with some external systems
     * and specifications (e.g. LDAP {SSHA}).
     * </p>
     * 
     * @since 1.7
     * 
     * @return whether plain salt will be appended after the digest operation 
     *         result instead of inserted before it (which is the 
     *         default). If null is returned, the default behaviour will be 
     *         applied.
     */
    public Boolean getInvertPositionOfPlainSaltInEncryptionResults();

    
    
    
    /**
     * <p>
     * Returns <tt>Boolean.TRUE</tt> if digest matching operations will allow matching
     * digests with a salt size different to the one configured in the "saltSizeBytes"
     * property. This is possible because digest algorithms will produce a fixed-size 
     * result, so the remaining bytes from the hashed input will be considered salt.
     * </p>
     * <p>
     * This will allow the digester to match digests produced in environments which do not
     * establish a fixed salt size as standard (for example, SSHA password encryption
     * in LDAP systems).  
     * </p>
     * <p>
     * The value of this property will <b>not</b> affect the creation of digests, 
     * which will always have a salt of the size established by the "saltSizeBytes" 
     * property. It will only affect digest matching.  
     * </p>
     * <p>
     * Setting this property to <tt>true</tt> is not compatible with {@link SaltGenerator}
     * implementations which return false for their 
     * {@link SaltGenerator#includePlainSaltInEncryptionResults()} property. 
     * </p>
     * <p>
     * Also, be aware that some algorithms or algorithm providers might not support
     * knowing the size of the digests beforehand, which is also incompatible with
     * a lenient behaviour.
     * </p>
     * <p>
     * <b>Default is <tt>FALSE</tt></b>.
     * </p>
     *
     * @since 1.7
     * 
     * @return whether the digester will allow matching of digests with different
     *         salt sizes than established or not (default is false).
     */
    public Boolean getUseLenientSaltSizeCheck();
    

    
    
    /**
     * <p>
     * Get the size of the pool of digesters to be created.
     * </p>
     * <p>
     * <b>This parameter will be ignored if used with a non-pooled digester</b>.
     * </p>
     *
     * @since 1.7
     * 
     * @return the size of the pool to be used if this configuration is used with a
     *         pooled digester
     */
    public Integer getPoolSize();
    
}
```


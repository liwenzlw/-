1. 将request.getInputStream() 返回的输入流进行适配，使其支持重复读
2. 适配request，将输入流的数据缓存到buffer中，每次调用request.getInputStream()都构造一个新的对象返回

比较：
    方案1 ：缺点：整个线程上下文使用的是同一个inputStream，上游读取数据的状态会干扰下游读取（比如第一次读了第一行，第二次必然从第二次开始读取），不可控性很强。而且读取到最后还需要reset。整个读取流程会很乱
    方案2：而方案二每次都会返回一个新的对象，读取操作非常独立，流程很干净，互不干扰。


示例一：

```java

public class InputStreamReadRepeatableRequestWrapper extends HttpServletRequestWrapper {

    private ServletInputStream inputStream;

    public InputStreamReadRepeatableRequestWrapper(HttpServletRequest request) {
        super(request);
    }

    @Override
    public ServletInputStream getInputStream() throws IOException {
        if (this.getContentLength() < 1) {
            return null;
        }
        if (this.inputStream == null) {
            this.inputStream = new InputStreamReadRepeatableRequestWrapper.ReadRepeatableInputStream(
                this.getRequest().getInputStream(),
                this.getRequest().getContentLength()
                );
        }
        return inputStream;
    }

    private class ReadRepeatableInputStream extends ServletInputStream {
        
        private final ByteArrayInputStream bi;

        private ReadRepeatableInputStream(ServletInputStream inputStream, int length) throws IOException {
            if (length > 0) {
                byte[] bytes = new byte[length];
                IOUtils.read(inputStream, bytes, 0, length);
                bi = new ByteArrayInputStream(bytes, 0, length);
            } else {
                bi = null;
            }
        }
        @Override
        public int read() throws IOException {
            int bt = -1;
            if (bi != null) {
                bt = this.bi.read();
                if (bt == -1) {
                    this.bi.reset();
                }
            }
            return bt;
        }
        @Override
        public void reset() {
            this.bi.reset();
        }
    }
}
```

示例二：

```java
public class RepeatReaderRequestWrapper extends HttpServletRequestWrapper {

    protected static final Logger logger = LoggerFactory.getLogger(RepeatReaderRequestWrapper.class);

    private byte[] bytes;

    public RepeatReaderRequestWrapper(HttpServletRequest request) {
        super(request);
        try {
            ServletInputStream inputStream = request.getInputStream();
            if (null == inputStream) {
                return;
            }
            bytes = IOUtils.toByteArray(inputStream);
        } catch (IOException e) {
            logger.error("封装RepeatReaderRequest错误：{}", e);
            throw new RuntimeException(e);
        }
    }

    @Override
    public BufferedReader getReader() throws IOException {
        if (null == bytes) {
            return null;
        } else {
            InputStreamReader inputStreamReader = new InputStreamReader(new ByteArrayInputStream(bytes));
            return new BufferedReader(inputStreamReader);
        }
    }

    @Override
    public ServletInputStream getInputStream() throws IOException {
        if (null == bytes) {
            return null;
        } else {
            return new DelegatingServletInputStream(new ByteArrayInputStream(bytes));
        }
    }

}
```
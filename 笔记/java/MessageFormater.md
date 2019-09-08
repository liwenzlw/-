对如下代码片段进行源码解析：
```java
String format = MessageFormat.format("{0},{1}", null, 1);
```

`MessageFormat.format`调用的是parents方式`Format.format`,我们看下调用链路

```java
public static String format(String pattern, Object ... arguments) {
    MessageFormat temp = new MessageFormat(pattern);
    return temp.format(arguments); // @a
}
// @a
public final String format (Object obj) {
    return format(obj, new StringBuffer(), new FieldPosition(0)).toString(); // @b
}   
// @b
public final StringBuffer format(Object arguments, StringBuffer result,  FieldPosition pos) {
    return subformat((Object[]) arguments, result, pos, null); // @c
}
```
我们看一下创建`MessageFormat`的时候做了什么
```java
    public MessageFormat(String pattern) {
        this.locale = Locale.getDefault(Locale.Category.FORMAT); // 获取系统的默认Local对象，并初始化格式化相关的东西
        applyPattern(pattern);
    }
```

我们再看下 `applyPattern("{0},{1}")` 做了什么

```java
    public void applyPattern(String pattern) {
            StringBuilder[] segments = new StringBuilder[4];
            // Allocate only segments[SEG_RAW] here. The rest are
            // allocated on demand.
            segments[SEG_RAW] = new StringBuilder();

            int part = SEG_RAW; // SEG_RAW = 0
            int formatNumber = 0;
            boolean inQuote = false; // 是否被单引号包裹 如： '1231'
            int braceStack = 0;
            maxOffset = -1;
            for (int i = 0; i < pattern.length(); ++i) { // pattern.length() = 7
                char ch = pattern.charAt(i);
                if (part == SEG_RAW) { // @a SEG_RAW = 0
                    if (ch == '\'') { // 如果碰到单引号
                        if (i + 1 < pattern.length() && pattern.charAt(i+1) == '\'') { // 碰到两个连续的单引号
                            segments[part].append(ch);  // 将两个连续的单引号 解析为 一个单引号字符
                            ++i;
                        } else {
                            inQuote = !inQuote; 
                        }
                    } else if (ch == '{' && !inQuote) { // '{' 且不在单引号内，会跳出@a逻辑，进入到@b逻辑（说明此处需要一个参数值）
                        part = SEG_INDEX; // SEG_INDEX = 1 索引片段
                        if (segments[SEG_INDEX] == null) {
                            segments[SEG_INDEX] = new StringBuilder();
                        }
                    } else {
                        segments[part].append(ch); // 追加纯文本
                    }
                } else  { // @b
                    if (inQuote) {              // just copy quotes in parts
                        segments[part].append(ch);
                        if (ch == '\'') {
                            inQuote = false;
                        }
                    } else {
                        switch (ch) {
                        case ',':
                            if (part < SEG_MODIFIER) { // SEG_MODIFIER = 3
                                if (segments[++part] == null) {
                                    segments[part] = new StringBuilder();
                                }
                            } else {
                                segments[part].append(ch);
                            }
                            break;
                        case '{':
                            ++braceStack;
                            segments[part].append(ch);
                            break;
                        case '}':
                            if (braceStack == 0) {
                                part = SEG_RAW; // 复位 part
                                makeFormat(i, formatNumber, segments); // @d 创建该索引处的格式
                                formatNumber++; 
                                // 复位 SEG_RAW 以外的 segments
                                segments[SEG_INDEX] = null;
                                segments[SEG_TYPE] = null;
                                segments[SEG_MODIFIER] = null;
                            } else {
                                --braceStack;
                                segments[part].append(ch);
                            }
                            break;
                        case ' ':
                            // Skip any leading space chars for SEG_TYPE.
                            if (part != SEG_TYPE || segments[SEG_TYPE].length() > 0) {
                                segments[part].append(ch);
                            }
                            break;
                        case '\'':
                            inQuote = true;
                            // fall through, so we keep quotes in other parts
                        default:
                            segments[part].append(ch); // 追加纯文本
                            break;
                        }
                    }
                }
            }
            if (braceStack == 0 && part != 0) {
                maxOffset = -1;
                throw new IllegalArgumentException("Unmatched braces in the pattern.");
            }
            this.pattern = segments[0].toString();
    }


```

最终调用的是`subformat`方法,我们做一下参数推演得到最终调用为 `subformat([null,],new StringBuffer(),new FieldPosition(0),null)`
这就是格式化的核心之处了，我们详细解读一下

```java
    private StringBuffer subformat(Object[] arguments, StringBuffer result,
                                   FieldPosition fp, List<AttributedCharacterIterator> characterIterators) {
        // note: this implementation assumes a fast substring & index.
        // if this is not true, would be better to append chars one by one.
        int lastOffset = 0;
        int last = result.length(); // 初始为 0 
        for (int i = 0; i <= maxOffset; ++i) {
            result.append(pattern.substring(lastOffset, offsets[i]));
            lastOffset = offsets[i];
            int argumentNumber = argumentNumbers[i];
            if (arguments == null || argumentNumber >= arguments.length) {
                result.append('{').append(argumentNumber).append('}');
                continue;
            }
            // int argRecursion = ((recursionProtection >> (argumentNumber*2)) & 0x3);
            if (false) { // if (argRecursion == 3){
                // prevent loop!!!
                result.append('\uFFFD');
            } else {
                Object obj = arguments[argumentNumber];
                String arg = null;
                Format subFormatter = null;
                if (obj == null) {
                    arg = "null";
                } else if (formats[i] != null) {
                    subFormatter = formats[i];
                    if (subFormatter instanceof ChoiceFormat) {
                        arg = formats[i].format(obj);
                        if (arg.indexOf('{') >= 0) {
                            subFormatter = new MessageFormat(arg, locale);
                            obj = arguments;
                            arg = null;
                        }
                    }
                } else if (obj instanceof Number) {
                    // format number if can
                    subFormatter = NumberFormat.getInstance(locale);
                } else if (obj instanceof Date) {
                    // format a Date if can
                    subFormatter = DateFormat.getDateTimeInstance(
                             DateFormat.SHORT, DateFormat.SHORT, locale);//fix
                } else if (obj instanceof String) {
                    arg = (String) obj;

                } else {
                    arg = obj.toString();
                    if (arg == null) arg = "null";
                }

                // At this point we are in two states, either subFormatter
                // is non-null indicating we should format obj using it,
                // or arg is non-null and we should use it as the value.

                if (characterIterators != null) {
                    // If characterIterators is non-null, it indicates we need
                    // to get the CharacterIterator from the child formatter.
                    if (last != result.length()) {
                        characterIterators.add(
                            createAttributedCharacterIterator(result.substring
                                                              (last)));
                        last = result.length();
                    }
                    if (subFormatter != null) {
                        AttributedCharacterIterator subIterator =
                                   subFormatter.formatToCharacterIterator(obj);

                        append(result, subIterator);
                        if (last != result.length()) {
                            characterIterators.add(
                                         createAttributedCharacterIterator(
                                         subIterator, Field.ARGUMENT,
                                         Integer.valueOf(argumentNumber)));
                            last = result.length();
                        }
                        arg = null;
                    }
                    if (arg != null && arg.length() > 0) {
                        result.append(arg);
                        characterIterators.add(
                                 createAttributedCharacterIterator(
                                 arg, Field.ARGUMENT,
                                 Integer.valueOf(argumentNumber)));
                        last = result.length();
                    }
                }
                else {
                    if (subFormatter != null) {
                        arg = subFormatter.format(obj);
                    }
                    last = result.length();
                    result.append(arg);
                    if (i == 0 && fp != null && Field.ARGUMENT.equals(
                                  fp.getFieldAttribute())) {
                        fp.setBeginIndex(last);
                        fp.setEndIndex(result.length());
                    }
                    last = result.length();
                }
            }
        }
        result.append(pattern.substring(lastOffset, pattern.length()));
        if (characterIterators != null && last != result.length()) {
            characterIterators.add(createAttributedCharacterIterator(
                                   result.substring(last)));
        }
        return result;
    }
```

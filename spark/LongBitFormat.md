# Java中使用 Long 表示枚举类
*在日常的开发过程中，很多时候我们需要枚举类(`enum`)来表示对象的各种状态，并且每个状态往往会关联到指定的数字，如：*
```
    private enum Color {
        RED(11), GREEN(21), YELLOW(31), BLACK(160);
		...
    };
```
或者用枚举类来表示一系列状态的转变关系：
```
    enum Week{
        SUNDAY(1), MONDAY(2), TUESDAY(3), WEDNESDAY(4), THRUSDAY(5), FRIDAY(6), SATRUDAY7);
		...
    };
```
那么，如何用最少的存储来实现这类需求，答案很简单，位存储。如 `1bit` 表示 `0,1` 两种状态，`2bit` 表示 `00,01,10,11` 四种状态，所以我们可以用一个 `long` 类型(`64bit`)/`int` 类型(`32bit`)存储多种状态，如下图：

![位存储示例][1] 

但是每新建一个枚举类都需要自己操作 `bit`：
1. 导致程序不易理解
2. 容易出错，耗费精力

`Hadoop hdfs` 的实现中，也遇到类似的问题，它借助于 `LongBitFormat.java` 类封装了 `bit` 操作：
```
    public class LongBitFormat implements Serializable {
    private static final long serialVersionUID = 1L;

    private final String NAME;
    /** Bit offset */
    private final int OFFSET;
    /** Bit length */
    private final int LENGTH;
    /** Minimum value */
    private final long MIN;
    /** Maximum value */
    private final long MAX;
    /** Bit mask */
    private final long MASK;

    public LongBitFormat(String name, LongBitFormat previous, int length,
                         long min) {
        NAME = name;
        OFFSET = previous == null ? 0 : previous.OFFSET + previous.LENGTH;
        LENGTH = length;
        MIN = min;
        MAX = ((-1L) >>> (64 - LENGTH));//移动的位数，右移64-Leng位，相当于保留length位
        MASK = MAX << OFFSET;
    }

    /** Retrieve the value from the record. */
    public long retrieve(long record) {
        return (record & MASK) >>> OFFSET;
    }

    /** Combine the value to the record. */
    public long combine(long value, long record) {
        if (value < MIN) {
            throw new IllegalArgumentException(
                    "Illagal value: " + NAME + " = " + value + " < MIN = " + MIN);
        }
        if (value > MAX) {
            throw new IllegalArgumentException(
                    "Illagal value: " + NAME + " = " + value + " > MAX = " + MAX);
        }
        return (record & ~MASK) | (value << OFFSET);
    }

    public long getMin() {
        return MIN;
    }
}
```
当然，你也可以实现 `IntBigFormat`,`ShortBitFormat` 等

**首先分析该类的构造方法：**
```
        NAME = name;
        OFFSET = previous == null? 0: previous.OFFSET + previous.LENGTH;
        LENGTH = length;
        MIN = min;
        MAX = ((-1L) >>> (64 - LENGTH));//移动的位数，右移64-Leng位，相当于保留length位
        MASK = MAX << OFFSET;
```
**字段：**
`NAME`：状态名，可自定义

`OFFSET`：该状态在 `long` 字节中的偏移

`LENGTH`：用多少位存储该状态关联的数字

`MIN`：该状态关联的最小值

`MAX`：该状态关联的最大值

`MASK`：掩码，`（OFFSET～OFFSET+LENGTH - 1） == 1`

类方法：
`retrieve(long record) `：获得该状态关联的数字

`combine(long value, long record)`：将一个 `value` 加到 `record` 中，例如：将 `value`  值对应的枚举类存储在 `32-40`，则先将 `32-40bits` 清零，再将`value` 对应的二进制加入到 `32-40`

**那么如何使用该类：**
```
public class LongFormatTest {

    static enum HeaderFormat {
        PREFERRED_BLOCK_SIZE(null, 48, 1),
        REPLICATION(PREFERRED_BLOCK_SIZE.BITS, 12, 1),
        STORAGE_POLICY_ID(REPLICATION.BITS, 4, 0);

        private final LongBitFormat BITS;

        HeaderFormat(LongBitFormat previous, int length, long min) {
            BITS = new LongBitFormat(name(), previous, length, min);
        }

        static short getReplication(long header) {
            return (short) REPLICATION.BITS.retrieve(header);
        }

        static long getPreferredBlockSize(long header) {
            return PREFERRED_BLOCK_SIZE.BITS.retrieve(header);
        }

        static byte getStoragePolicyID(long header) {
            return (byte) STORAGE_POLICY_ID.BITS.retrieve(header);
        }

        static long toLong(long preferredBlockSize, long replication,
                           long storagePolicyID) {
            long h = 0;
            h = PREFERRED_BLOCK_SIZE.BITS.combine(preferredBlockSize, h);
            h = REPLICATION.BITS.combine(replication, h);
            h = STORAGE_POLICY_ID.BITS.combine(storagePolicyID, h);
            return h;
        }
    }

    public static void main(String[] args) {

        long blockSize = 512;
        long replication = 3L;
        long storagePolicyID = 2L;
        long combine = HeaderFormat.toLong(blockSize,replication,storagePolicyID);
        System.out.println("block size:         " + HeaderFormat.getPreferredBlockSize(combine));
        System.out.println("replication:        " + HeaderFormat.getReplication(combine));
        System.out.println("storagePolicyID:    " + HeaderFormat.getStoragePolicyID(combine));
    }
}
```
`tolong` 方法的返回值也就是我们状态存储的封装

  [1]: ./images/untitled_page_1.png "位存储示例"
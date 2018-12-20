title: Nicely Formatted Tabular Data in Java
date: 2013-08-06

Occasionally, it’s useful to be able to print nicely formatted tables
of data to a textual output stream. This is particularly the case when
writing command line tools. To make the output of these tools more
readable, any tables they write should have columns that line-up from
row to row. The Unix tool `ls` does this when it prints out long form
directory listings. In this example, notice how the dates line up,
even though the file size column varies in width.

```nohighlight
drwxr-xr-x+ 1 mschaef Domain Users        0 Oct  3 09:20 docs
-rwxr-xr-x  1 mschaef Domain Users 29109013 Oct 10 13:38 file.zip
-rwxr-xr-x  1 mschaef Domain Users    77500 Oct 10 13:17 file2.zip
```

To accomplish this, it’s necessary to accumulate all of the lines of
text to be written, compute the column widths when all lines are
known, and then print the lines out, with appropriate padding to
ensure that columns occupy the same width in each row. This is easy to
accomplish, with just a bit of reusable Java.

```java
package com.ksmpartners.utility;
 
import java.util.LinkedList;
import java.util.List;
 
import org.apache.commons.lang3.StringUtils;
 
public class TableBuilder
{
    List<String[]> rows = new LinkedList<String[]>();
 
    public void addRow(String... cols)
    {
        rows.add(cols);
    }
 
    private int[] colWidths()
    {
        int cols = -1;
 
        for(String[] row : rows)
            cols = Math.max(cols, row.length);
 
        int[] widths = new int[cols];
 
        for(String[] row : rows) {
            for(int colNum = 0; colNum < row.length; colNum++) {
                widths[colNum] =
                    Math.max(
                        widths[colNum],
                        StringUtils.length(row[colNum]));
            }
        }
 
        return widths;
    }
 
    @Override
    public String toString()
    {
        StringBuilder buf = new StringBuilder();
 
        int[] colWidths = colWidths();
 
        for(String[] row : rows) {
            for(int colNum = 0; colNum < row.length; colNum++) {
                buf.append(
                    StringUtils.rightPad(
                        StringUtils.defaultString(
                            row[colNum]), colWidths[colNum]));
                buf.append(' ');
            }
 
            buf.append('\n');
        }
 
        return buf.toString();
    }
 
}
```

The calling convention for this class is very much in line with the
calling convention for Java’s `StringBuilder`.

```java
TableBuilder tb = new TableBuilder();
 
tb.addRow("alpha", "beta", "gamma");
tb.addRow("-----", "----", "-----");
tb.addRow("1", "20000000", "foo");
tb.addRow("x", "yzz", "y");
 
System.out.println(tb.toString());
```

That code will write the following output:

```nohighlight
alpha beta     gamma
----- ----     -----
1     20000000 foo
x     yzz      y
```

This isn’t necessarily the prettiest output in the world, but it’s
easy to accomplish and is much better than many of the alternatives.

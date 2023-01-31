> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [codefordev.com](https://codefordev.com/discuss/6120617872/java-native-method-source-code)

> You can download OpenJdk source code here.

Answer #1

100 %

You can download OpenJdk source code [here](http://hg.openjdk.java.net/jdk7/jdk7).

In the folder `jdk\src\share` you can get source code.

`jdk\src\share\native` is the natice method souce write in c and c++.

1.  `jdk\src\linux` source for linux.
2.  `jdk\src\windows` source for windows.
3.  `jdk\src\solaris` souce for solaris.
4.  `jd\src\share` common source.

eg: System.arrayCopy();

int file `hotspot\src\share\vm\oops\objArrayKlass.cpp` line 168:

```
void objArrayKlass::copy_array(arrayOop s, int src_pos, arrayOop d,
                           int dst_pos, int length, TRAPS) {
assert(s->is_objArray(), "must be obj array");

if (!d->is_objArray()) {
  THROW(vmSymbols::java_lang_ArrayStoreException());
}

// Check is all offsets and lengths are non negative
if (src_pos < 0 || dst_pos < 0 || length < 0) {
  THROW(vmSymbols::java_lang_ArrayIndexOutOfBoundsException());
}
// Check if the ranges are valid
if  ( (((unsigned int) length + (unsigned int) src_pos) > (unsigned int) s->length())
   || (((unsigned int) length + (unsigned int) dst_pos) > (unsigned int) d->length()) )   {
  THROW(vmSymbols::java_lang_ArrayIndexOutOfBoundsException());
}

// Special case. Boundary cases must be checked first
// This allows the following call: copy_array(s, s.length(), d.length(), 0).
// This is correct, since the position is supposed to be an 'in between point', i.e., s.length(),
// points to the right of the last element.
if (length==0) {
  return;
}
if (UseCompressedOops) {
  narrowOop* const src = objArrayOop(s)->obj_at_addr(src_pos);
  narrowOop* const dst = objArrayOop(d)->obj_at_addr(dst_pos);
  do_copy(s, src, d, dst, length, CHECK);
} else {
  oop* const src = objArrayOop(s)->obj_at_addr(src_pos);
  oop* const dst = objArrayOop(d)->obj_at_addr(dst_pos);
  do_copy (s, src, d, dst, length, CHECK);
  }
}
```
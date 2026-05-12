---
date: '2026-05-12T11:50:06+03:30'
draft: false
title: 'کلاس sequence در پایتون'
---

# کلاس Sequence در پایتون: از اجداد تا فرزندان

## مقدمه‌ای که باید بخوانید

وقتی برای اولین بار پایتون یاد می‌گیرید، سریع با چیزهایی مثل لیست، تاپل و رشته آشنا می‌شوید. این‌ها ابزارهای روزمره‌ی هر برنامه‌نویس پایتونی هستند. اما آیا تا به حال از خودتان پرسیده‌اید که چرا هر سه‌ی این‌ها رفتارهای مشترکی دارند؟ چرا می‌توانید روی هر سه‌ی آن‌ها حلقه بزنید؟ چرا هر سه از اندیس‌گذاری پشتیبانی می‌کنند؟ چرا هر سه تابع `len()` را قبول می‌کنند؟

جواب در یک مفهوم نهفته است: **پروتکل Sequence**.

پایتون یک زبان با سیستم تایپ پویا (dynamic typing) است، اما این به معنای بی‌نظمی نیست. در دل پایتون، یک سیستم دقیق و زیبا طراحی شده که رفتار انواع داده را از طریق پروتکل‌ها و کلاس‌های پایه‌ی انتزاعی (Abstract Base Classes یا ABCs) مشخص می‌کند. کلاس `Sequence` یکی از مهم‌ترین این کلاس‌های پایه است.

در این مقاله، یک سفر کامل خواهیم داشت:

- از **اجداد** `Sequence` شروع می‌کنیم (کلاس‌هایی مثل `Iterable`، `Container` و `Sized`)
- خود **`Sequence`** را زیر ذره‌بین می‌گذاریم
- تمام **فرزندان** آن (مثل `list`، `tuple`، `str`، `range` و ...) را بررسی می‌کنیم
- یاد می‌گیریم چطور **Sequence سفارشی** بسازیم
- و در نهایت با **`MutableSequence`** و تفاوت‌هایش آشنا می‌شویم

---

## بخش اول: قبل از Sequence — اجداد و ریشه‌ها

### ۱.۱ — ماژول `collections.abc` چیست؟

همه‌ی ماجرا در ماژول `collections.abc` اتفاق می‌افتد. این ماژول مجموعه‌ای از **کلاس‌های پایه‌ی انتزاعی** (Abstract Base Classes) را در اختیار ما می‌گذارد که رفتار انواع داده‌ای مختلف را تعریف می‌کنند.

قبل از پایتون ۳.۳، این کلاس‌ها مستقیماً در `collections` بودند. از پایتون ۳.۳ به بعد، به `collections.abc` منتقل شدند و از پایتون ۳.۱۰ به بعد، استفاده از `collections.MutableSequence` (بدون `.abc`) دیگر کار نمی‌کند.

```python
# روش صحیح import
from collections.abc import (
    Sequence,
    MutableSequence,
    Iterable,
    Iterator,
    Reversible,
    Container,
    Sized,
    Collection,
)
```

### ۱.۲ — درخت کامل وراثت

برای اینکه بفهمیم `Sequence` از کجا آمده، باید کل درخت وراثت را ببینیم:

```
object
└── Hashable          (__hash__)
└── Iterable          (__iter__)
    └── Iterator      (__next__, __iter__)
        └── Generator (__next__, send, throw, ...)
    └── Reversible    (__reversed__)
    └── Collection    (__contains__, __iter__, __len__)  [ترکیب سه تا]
        ├── Sized     (__len__)
        ├── Container (__contains__)
        └── Iterable  (__iter__)
            └── Sequence   (__getitem__, __len__)
                └── MutableSequence  (__setitem__, __delitem__, insert)
                └── ByteString  (deprecated در ۳.۱۲)
```

این نمودار نشان می‌دهد که `Sequence` در واقع ترکیب و تکامل چندین مفهوم ساده‌تر است.

### ۱.۳ — جد اول: `Iterable`

اولین و مهم‌ترین جد `Sequence` کلاس `Iterable` است.

**تعریف:** هر شیئی که می‌توانیم روی آن حلقه بزنیم، یک `Iterable` است.

**متد انتزاعی:** `__iter__`

```python
from collections.abc import Iterable

class MyRange:
    def __init__(self, n):
        self.n = n
    
    def __iter__(self):
        current = 0
        while current < self.n:
            yield current
            current += 1

r = MyRange(5)
print(isinstance(r, Iterable))  # True — چون __iter__ داره

for x in r:
    print(x)  # 0, 1, 2, 3, 4
```

نکته‌ی مهم: پایتون یک مکانیزم fallback هم دارد. اگر شیئی `__iter__` نداشته باشد اما `__getitem__` داشته باشد، پایتون سعی می‌کند با ایندکس‌های ۰، ۱، ۲، ... روی آن iterate کند تا به `IndexError` برسد. این یک legacy behavior است:

```python
class OldStyle:
    def __getitem__(self, index):
        if index >= 5:
            raise IndexError
        return index * 2

obj = OldStyle()
# این کار می‌کند، حتی بدون __iter__:
for x in obj:
    print(x)  # 0, 2, 4, 6, 8

# اما isinstance چک می‌کند که آیا __iter__ وجود داره:
print(isinstance(obj, Iterable))  # False!
```

### ۱.۴ — جد دوم: `Sized`

**تعریف:** هر شیئی که می‌توان طول آن را با `len()` بدست آورد.

**متد انتزاعی:** `__len__`

```python
from collections.abc import Sized

class MyCollection:
    def __init__(self, data):
        self.data = data
    
    def __len__(self):
        return len(self.data)

mc = MyCollection([1, 2, 3, 4, 5])
print(len(mc))           # 5
print(isinstance(mc, Sized))  # True
```

نکته‌ی ظریف: `__len__` باید یک عدد صحیح غیرمنفی برگرداند. اگر منفی برگردانید، پایتون `ValueError` می‌دهد:

```python
class BadSized:
    def __len__(self):
        return -1  # اشتباه!

bs = BadSized()
try:
    len(bs)
except ValueError as e:
    print(e)  # __len__() should return >= 0
```

### ۱.۵ — جد سوم: `Container`

**تعریف:** هر شیئی که می‌توان با `in` بررسی عضویت کرد.

**متد انتزاعی:** `__contains__`

```python
from collections.abc import Container

class EvenNumbers:
    def __contains__(self, item):
        return isinstance(item, int) and item % 2 == 0

evens = EvenNumbers()
print(2 in evens)   # True
print(3 in evens)   # False
print(100 in evens) # True
print(isinstance(evens, Container))  # True
```

### ۱.۶ — جد چهارم: `Collection`

`Collection` ترکیب سه جد بالاست: `Sized` + `Iterable` + `Container`.

این یعنی هر `Collection`:
- طول دارد (`__len__`)
- قابل iterate است (`__iter__`)
- از عملگر `in` پشتیبانی می‌کند (`__contains__`)

```python
from collections.abc import Collection

class NumberSet(Collection):
    def __init__(self, *args):
        self._data = set(args)
    
    def __contains__(self, item):
        return item in self._data
    
    def __iter__(self):
        return iter(self._data)
    
    def __len__(self):
        return len(self._data)

ns = NumberSet(1, 2, 3, 4, 5)
print(len(ns))        # 5
print(3 in ns)        # True
print(list(ns))       # [1, 2, 3, 4, 5] (ترتیب ممکن است فرق کند)
```

### ۱.۷ — جد پنجم: `Reversible`

**تعریف:** شیئی که می‌توان با `reversed()` آن را معکوس کرد.

**متد انتزاعی:** `__reversed__`

```python
from collections.abc import Reversible

class CountDown(Reversible):
    def __init__(self, start):
        self.start = start
    
    def __iter__(self):
        for i in range(self.start + 1):
            yield i
    
    def __reversed__(self):
        for i in range(self.start, -1, -1):
            yield i

cd = CountDown(5)
print(list(cd))            # [0, 1, 2, 3, 4, 5]
print(list(reversed(cd)))  # [5, 4, 3, 2, 1, 0]
```

---

## بخش دوم: خود Sequence — قلب ماجرا

### ۲.۱ — Sequence چیست؟

`Sequence` یک **Collection مرتب و با دسترسی به اندیس** است. این تعریف کوتاه اما پر از معناست:

- **مرتب:** عناصر ترتیب دارند و ترتیب اهمیت دارد
- **با دسترسی به اندیس:** می‌توان با `obj[0]`، `obj[1]`، ... به هر عنصر دسترسی داشت
- **طول مشخص:** می‌توان طول آن را با `len()` بدست آورد

### ۲.۲ — متدهای انتزاعی (Abstract Methods)

برای پیاده‌سازی یک `Sequence`، باید دقیقاً **دو متد** را پیاده‌سازی کنید:

| متد | توضیح |
|-----|-------|
| `__getitem__(self, index)` | دسترسی به عنصر با اندیس یا slice |
| `__len__(self)` | تعداد عناصر |

همین دو متد کافی است! بقیه متدها به صورت خودکار از این دو مشتق می‌شوند.

### ۲.۳ — متدهای Mixin (رایگان!)

وقتی از `Sequence` ارث می‌برید و آن دو متد را پیاده‌سازی می‌کنید، به صورت **رایگان** این متدها هم به کلاستان اضافه می‌شوند:

| متد | توضیح | نحوه‌ی مشتق‌شدن |
|-----|-------|----------------|
| `__contains__` | عملگر `in` | با iterate کردن و مقایسه |
| `__iter__` | حلقه `for` | با اندیس از ۰ تا `len-1` |
| `__reversed__` | تابع `reversed()` | با اندیس از `len-1` تا ۰ |
| `index` | پیدا کردن اندیس عنصر | با iterate کردن |
| `count` | شمردن تعداد تکرار | با iterate کردن |

بیایید این را در عمل ببینیم:

```python
from collections.abc import Sequence

class FibSequence(Sequence):
    """دنباله فیبوناچی به صورت Sequence"""
    
    def __init__(self, length):
        self._length = length
        self._cache = {}
    
    def _fib(self, n):
        if n in self._cache:
            return self._cache[n]
        if n <= 1:
            return n
        result = self._fib(n-1) + self._fib(n-2)
        self._cache[n] = result
        return result
    
    def __getitem__(self, index):
        if isinstance(index, slice):
            # پشتیبانی از slice
            indices = range(*index.indices(self._length))
            return [self._fib(i) for i in indices]
        
        if index < 0:
            index += self._length
        if not 0 <= index < self._length:
            raise IndexError("index out of range")
        return self._fib(index)
    
    def __len__(self):
        return self._length


# ایجاد و استفاده
fib = FibSequence(10)

# __getitem__ (که خودمان نوشتیم)
print(fib[0])   # 0
print(fib[1])   # 1
print(fib[7])   # 13
print(fib[-1])  # 34

# __len__ (که خودمان نوشتیم)
print(len(fib)) # 10

# __iter__ (رایگان از Sequence!)
print(list(fib))  # [0, 1, 1, 2, 3, 5, 8, 13, 21, 34]

# __contains__ (رایگان از Sequence!)
print(8 in fib)   # True
print(9 in fib)   # False

# __reversed__ (رایگان از Sequence!)
print(list(reversed(fib)))  # [34, 21, 13, 8, 5, 3, 2, 1, 1, 0]

# index (رایگان از Sequence!)
print(fib.index(13))  # 7

# count (رایگان از Sequence!)
print(fib.count(1))   # 2  (چون هم fib[1]=1 و هم fib[2]=1)

# slice (که ما هندل کردیم)
print(fib[2:6])  # [1, 2, 3, 5]

# isinstance چک
from collections.abc import Sequence as SeqABC
print(isinstance(fib, SeqABC))  # True
```

### ۲.۴ — پیاده‌سازی داخلی متدهای Mixin

بیایید ببینیم پایتون چطور این متدها را "رایگان" می‌دهد. کد ساده‌شده‌ی CPython:

```python
# این کد ساده‌سازی‌شده‌ی پیاده‌سازی داخلی است
class SequenceBase:
    
    # این دو را باید خودتان پیاده‌سازی کنید:
    def __getitem__(self, index):
        raise NotImplementedError
    
    def __len__(self):
        raise NotImplementedError
    
    # اینها رایگان هستند:
    
    def __contains__(self, value):
        # یک به یک عناصر را بررسی می‌کند
        for v in self:
            if v is value or v == value:
                return True
        return False
    
    def __iter__(self):
        # از اندیس ۰ شروع می‌کند تا IndexError بگیرد
        i = 0
        try:
            while True:
                v = self[i]
                yield v
                i += 1
        except IndexError:
            return
    
    def __reversed__(self):
        # از آخر به اول می‌رود
        for i in reversed(range(len(self))):
            yield self[i]
    
    def index(self, value, start=0, stop=None):
        if stop is None:
            stop = len(self)
        # نرمال‌سازی
        if start < 0:
            start = max(len(self) + start, 0)
        if stop < 0:
            stop = max(len(self) + stop, 0)
        
        i = start
        while i < stop:
            v = self[i]
            if v is value or v == value:
                return i
            i += 1
        raise ValueError(f"{value!r} is not in sequence")
    
    def count(self, value):
        cnt = 0
        for v in self:
            if v is value or v == value:
                cnt += 1
        return cnt
```

### ۲.۵ — Virtual Subclassing (ثبت مجازی)

یکی از قدرتمندترین ویژگی‌های ABC در پایتون، **Virtual Subclassing** است. این یعنی می‌توانید کلاسی را به عنوان زیرکلاس یک ABC ثبت کنید، حتی اگر واقعاً از آن ارث نبرده باشد!

```python
from collections.abc import Sequence

class MyCustomList:
    """این کلاس از Sequence ارث نمی‌برد"""
    
    def __init__(self, data):
        self._data = list(data)
    
    def __getitem__(self, index):
        return self._data[index]
    
    def __len__(self):
        return len(self._data)

# ثبت به عنوان Sequence مجازی
Sequence.register(MyCustomList)

mcl = MyCustomList([1, 2, 3])
print(isinstance(mcl, Sequence))  # True — حتی بدون ارث‌بری واقعی!
print(issubclass(MyCustomList, Sequence))  # True
```

اما توجه کنید: در این حالت متدهای Mixin رایگان نیستند! چون ارث‌بری واقعی اتفاق نیفتاده.

```python
# بدون ارث‌بری واقعی، index و count ندارید:
try:
    mcl.index(2)
except AttributeError as e:
    print(e)  # 'MyCustomList' object has no attribute 'index'
```

### ۲.۶ — `__subclasshook__` — منطق سفارشی برای isinstance

هر ABC می‌تواند یک `__subclasshook__` داشته باشد که منطق `isinstance` را سفارشی کند:

```python
from collections.abc import Iterable
import inspect

# Iterable.__subclasshook__ تقریباً اینطور کار می‌کند:
@classmethod
def __subclasshook__(cls, C):
    if cls is Iterable:
        # بررسی می‌کند که آیا __iter__ در MRO هست
        if any("__iter__" in B.__dict__ for B in C.__mro__):
            return True
    return NotImplemented
```

این است که چرا بدون ثبت مجازی، هر کلاسی که `__iter__` داشته باشد، `isinstance(obj, Iterable)` برایش `True` می‌دهد.

---

---

## بخش سوم: فرزندان Sequence — بررسی کامل

### ۳.۱ — نقشه‌ی فرزندان

قبل از اینکه وارد جزئیات بشیم، ببینیم فرزندان `Sequence` چه کسانی هستند:

```
Sequence
├── list          (mutable — از MutableSequence هم ارث می‌بره)
├── tuple         (immutable)
├── str           (immutable — فقط کاراکتر)
├── range         (immutable — lazy)
├── bytes         (immutable — فقط بایت)
├── bytearray     (mutable — از MutableSequence هم ارث می‌بره)
├── memoryview    (mutable یا immutable بسته به buffer)
└── array.array   (mutable — از MutableSequence ارث می‌بره)
```

هر کدام از این‌ها ویژگی‌های خاص خودشان را دارند. بیایید یکی یکی بررسی کنیم.

---

### ۳.۲ — فرزند اول: `list`

`list` احتمالاً پرکاربردترین Sequence در پایتون است. یک آرایه‌ی پویا (dynamic array) که می‌تواند هر نوع شیئی را نگه دارد.

#### ساختار داخلی list

`list` در CPython به عنوان یک **آرایه‌ی پویا از اشاره‌گرها** پیاده‌سازی شده. این یعنی:

- هر عنصر در واقع یک **اشاره‌گر** (pointer) به یک شیء پایتون است
- خود اشیاء جای دیگری در حافظه هستند
- وقتی ظرفیت تمام می‌شود، یک آرایه‌ی بزرگتر ساخته می‌شه و همه چیز کپی می‌شه

```python
import sys

# ببینیم list چقدر حافظه مصرف می‌کند
empty_list = []
print(sys.getsizeof(empty_list))  # 56 bytes (overhead خود list)

# با یک عنصر
one_element = [1]
print(sys.getsizeof(one_element))  # 64 bytes

# هر اشاره‌گر 8 بایت است (روی سیستم 64 بیتی)
```

#### growth factor در list

وقتی یک `list` پر می‌شود، پایتون حافظه‌ی بیشتری اختصاص می‌دهد. این فرمول تقریبی است:

```python
# بررسی capacity واقعی list
import ctypes

def get_list_capacity(lst):
    """ظرفیت واقعی list را برمی‌گرداند"""
    # ob_alloc در ساختار PyListObject
    return lst.__sizeof__() // 8 - 1  # تقریبی

# ببینیم چطور رشد می‌کند
lst = []
prev_size = sys.getsizeof(lst)
for i in range(20):
    lst.append(i)
    new_size = sys.getsizeof(lst)
    if new_size != prev_size:
        print(f"بعد از {i+1} عنصر: {new_size} bytes")
        prev_size = new_size

# خروجی تقریبی:
# بعد از 1 عنصر: 88 bytes   (4 slot)
# بعد از 5 عنصر: 120 bytes  (8 slot)
# بعد از 9 عنصر: 184 bytes  (16 slot)
# بعد از 17 عنصر: 248 bytes (25 slot)
```

فرمول رشد: `new_capacity = (old_capacity * 9 // 8) + 6`

#### پیچیدگی زمانی عملیات‌های list

```python
# O(1) — دسترسی به اندیس
lst = [1, 2, 3, 4, 5]
x = lst[2]  # سریع، همیشه O(1)

# O(1) amortized — append به انتها
lst.append(6)  # معمولاً سریع

# O(n) — insert در ابتدا یا وسط
lst.insert(0, 0)  # همه عناصر باید جابجا شوند!

# O(n) — حذف از ابتدا
lst.pop(0)  # همه عناصر به چپ shift می‌شوند

# O(1) — حذف از انتها
lst.pop()  # سریع

# O(n) — جستجو
lst.index(3)  # باید همه را بررسی کند

# O(n log n) — مرتب‌سازی
lst.sort()  # از Timsort استفاده می‌کند

# O(k) — slice گرفتن (k طول slice است)
sub = lst[1:4]  # یک list جدید می‌سازد
```

#### تمام متدهای list با مثال

```python
lst = [3, 1, 4, 1, 5, 9, 2, 6, 5, 3]

# ── متدهای اضافه‌کردن ──

# append: یک عنصر به انتها اضافه می‌کند — O(1) amortized
lst.append(7)
print(lst)  # [3, 1, 4, 1, 5, 9, 2, 6, 5, 3, 7]

# insert: در موقعیت مشخص — O(n)
lst.insert(2, 99)
print(lst)  # [3, 1, 99, 4, 1, 5, 9, 2, 6, 5, 3, 7]

# extend: عناصر یک iterable را اضافه می‌کند — O(k)
lst.extend([10, 11])
print(lst)  # [3, 1, 99, 4, 1, 5, 9, 2, 6, 5, 3, 7, 10, 11]

# ── متدهای حذف‌کردن ──

# remove: اولین تکرار مقدار را حذف می‌کند — O(n)
lst.remove(99)
print(lst)  # [3, 1, 4, 1, 5, 9, 2, 6, 5, 3, 7, 10, 11]

# pop: عنصر را حذف و برمی‌گرداند
last = lst.pop()      # آخرین عنصر — O(1)
first = lst.pop(0)    # اولین عنصر — O(n)
print(last, first)    # 11, 3

# clear: همه عناصر را حذف می‌کند — O(n)
temp = [1, 2, 3]
temp.clear()
print(temp)  # []

# del با اندیس یا slice
lst2 = [1, 2, 3, 4, 5]
del lst2[2]
print(lst2)   # [1, 2, 4, 5]
del lst2[1:3]
print(lst2)   # [1, 5]

# ── متدهای جستجو ──

lst = [3, 1, 4, 1, 5, 9, 2, 6, 5, 3]

# index: اندیس اولین تکرار — O(n)
print(lst.index(5))        # 4
print(lst.index(5, 5))     # 8  (جستجو از اندیس 5)
print(lst.index(5, 5, 9))  # 8  (جستجو از 5 تا 9)

# count: تعداد تکرار — O(n)
print(lst.count(1))  # 2
print(lst.count(5))  # 2
print(lst.count(7))  # 0

# ── متدهای ترتیب ──

# sort: مرتب‌سازی درجا — O(n log n) — Timsort
lst.sort()
print(lst)  # [1, 1, 2, 3, 3, 4, 5, 5, 6, 9]

lst.sort(reverse=True)
print(lst)  # [9, 6, 5, 5, 4, 3, 3, 2, 1, 1]

# مرتب‌سازی با key
words = ['banana', 'apple', 'cherry', 'date']
words.sort(key=len)
print(words)  # ['date', 'apple', 'banana', 'cherry']

words.sort(key=lambda x: x[-1])  # بر اساس آخرین حرف
print(words)  # ['banana', 'apple', 'date', 'cherry']

# reverse: معکوس کردن درجا — O(n)
lst.reverse()
print(lst)  # [1, 1, 2, 3, 3, 4, 5, 5, 6, 9]

# ── کپی ──

# copy: کپی سطحی — O(n)
original = [[1, 2], [3, 4]]
shallow = original.copy()
shallow[0].append(99)
print(original)  # [[1, 2, 99], [3, 4]] — original هم تغییر کرد!

# برای کپی عمیق:
import copy
deep = copy.deepcopy(original)
deep[0].append(100)
print(original)  # [[1, 2, 99], [3, 4]] — تغییر نکرد
```

#### List Comprehension — قدرت واقعی list

```python
# ساخت list با comprehension
squares = [x**2 for x in range(10)]
print(squares)  # [0, 1, 4, 9, 16, 25, 36, 49, 64, 81]

# با شرط
even_squares = [x**2 for x in range(10) if x % 2 == 0]
print(even_squares)  # [0, 4, 16, 36, 64]

# تودرتو
matrix = [[1, 2, 3], [4, 5, 6], [7, 8, 9]]
flat = [x for row in matrix for x in row]
print(flat)  # [1, 2, 3, 4, 5, 6, 7, 8, 9]

# با چند شرط
result = [x for x in range(100) 
          if x % 2 == 0 
          if x % 3 == 0]
print(result)  # [0, 6, 12, 18, ...]

# comprehension تودرتو برای transpose
transpose = [[row[i] for row in matrix] for i in range(3)]
print(transpose)  # [[1, 4, 7], [2, 5, 8], [3, 6, 9]]
```

#### نکات پیشرفته list

```python
# ── Unpacking ──
a, b, c = [1, 2, 3]
first, *rest = [1, 2, 3, 4, 5]
*beginning, last = [1, 2, 3, 4, 5]
first, *middle, last = [1, 2, 3, 4, 5]
print(first, middle, last)  # 1 [2, 3, 4] 5

# ── مقایسه list‌ها ──
print([1, 2, 3] == [1, 2, 3])  # True
print([1, 2, 3] < [1, 2, 4])   # True (مقایسه لغوی)
print([1, 2] < [1, 2, 3])      # True (کوتاه‌تر کوچک‌تر است)

# ── ضرب list ──
zeros = [0] * 5
print(zeros)  # [0, 0, 0, 0, 0]

# دام مشهور: با شیء mutable!
matrix_wrong = [[0] * 3] * 3  # اشتباه!
matrix_wrong[0][0] = 1
print(matrix_wrong)  # [[1, 0, 0], [1, 0, 0], [1, 0, 0]] — همه تغییر کردند!

matrix_correct = [[0] * 3 for _ in range(3)]  # درست
matrix_correct[0][0] = 1
print(matrix_correct)  # [[1, 0, 0], [0, 0, 0], [0, 0, 0]]

# ── list به عنوان stack و queue ──

# Stack (LIFO) — با list خوب کار می‌کند
stack = []
stack.append(1)   # push
stack.append(2)
stack.append(3)
print(stack.pop())  # 3 — pop از انتها O(1)

# Queue (FIFO) — با list بد است! از deque استفاده کنید
from collections import deque
queue = deque()
queue.append(1)    # enqueue
queue.append(2)
queue.append(3)
print(queue.popleft())  # 1 — O(1)
```

---

### ۳.۳ — فرزند دوم: `tuple`

`tuple` یک Sequence **تغییرناپذیر (immutable)** است. اما این تغییرناپذیری چیزی بیشتر از یک محدودیت است — یک **قرارداد** است.

#### ساختار داخلی tuple

```python
import sys

# tuple کمتر از list حافظه مصرف می‌کند
lst = [1, 2, 3, 4, 5]
tpl = (1, 2, 3, 4, 5)

print(sys.getsizeof(lst))  # 104 bytes
print(sys.getsizeof(tpl))  # 80 bytes  — کمتر!
```

چرا کمتر؟ چون `tuple` نیازی به ظرفیت اضافه ندارد. دقیقاً به اندازه‌ای که لازم دارد ساخته می‌شود.

#### tuple interning

پایتون برخی tuple‌های خاص را **intern** می‌کند — یعنی یک نسخه می‌سازد و بارها استفاده می‌کند:

```python
# tuple خالی همیشه یکی است
a = ()
b = ()
print(a is b)  # True — همان شیء!

# tuple یک عنصری ممکن است intern شود
x = (1,)
y = (1,)
print(x is y)  # True یا False بسته به implementation

# tuple‌های بزرگتر معمولاً intern نمی‌شوند
p = (1, 2, 3)
q = (1, 2, 3)
print(p is q)  # احتمالاً False
```

#### کاربردهای tuple

```python
# ── Named Tuple — tuple با اسم ──
from collections import namedtuple

Point = namedtuple('Point', ['x', 'y'])
p = Point(3, 4)
print(p.x, p.y)   # 3 4
print(p[0], p[1]) # 3 4 — هنوز مثل tuple کار می‌کند
print(p)          # Point(x=3, y=4)

# با مقدار پیشفرض
Person = namedtuple('Person', ['name', 'age', 'city'], defaults=['Tehran'])
alice = Person('Alice', 30)
print(alice.city)  # Tehran

# ── typing.NamedTuple — نسخه مدرن ──
from typing import NamedTuple

class Coordinate(NamedTuple):
    x: float
    y: float
    z: float = 0.0

c = Coordinate(1.0, 2.0)
print(c)       # Coordinate(x=1.0, y=2.0, z=0.0)
print(c.z)     # 0.0

# ── tuple به عنوان کلید dictionary ──
# چون hashable است!
locations = {
    (35.6892, 51.3890): "Tehran",
    (48.8566, 2.3522): "Paris",
    (51.5074, -0.1278): "London",
}
print(locations[(35.6892, 51.3890)])  # Tehran

# list نمی‌تواند کلید باشد:
try:
    bad = {[1, 2]: "value"}
except TypeError as e:
    print(e)  # unhashable type: 'list'

# ── tuple unpacking ──
coordinates = [(1, 2), (3, 4), (5, 6)]
for x, y in coordinates:
    print(f"x={x}, y={y}")

# swap با tuple
a, b = 1, 2
a, b = b, a
print(a, b)  # 2 1

# تابع که چند مقدار برمی‌گرداند
def min_max(lst):
    return min(lst), max(lst)  # یک tuple برمی‌گرداند

minimum, maximum = min_max([3, 1, 4, 1, 5, 9])
print(minimum, maximum)  # 1 9

# ── tuple با یک عنصر — دام مشهور! ──
not_a_tuple = (42)    # این یک عدد است!
is_a_tuple = (42,)    # این یک tuple است — کاما مهم است!
print(type(not_a_tuple))  # <class 'int'>
print(type(is_a_tuple))   # <class 'tuple'>

# ── تغییرناپذیری ──
t = (1, [2, 3], 4)
# نمی‌توانیم عنصر tuple را تغییر دهیم:
try:
    t[1] = [5, 6]
except TypeError as e:
    print(e)  # 'tuple' object does not support item assignment

# اما اگر عنصر mutable باشد، خودش قابل تغییر است:
t[1].append(99)
print(t)  # (1, [2, 3, 99], 4) — tuple "تغییر" کرد ولی نه!

# به همین دلیل tuple با عنصر mutable قابل hash نیست:
try:
    hash(t)
except TypeError as e:
    print(e)  # unhashable type: 'list'
```

---

### ۳.۴ — فرزند سوم: `str`

`str` یک Sequence از **کاراکترهای یونیکد** است. یکی از پیچیده‌ترین و قدرتمندترین نوع داده در پایتون.

#### ساختار داخلی str

پایتون ۳ از یونیکد استفاده می‌کند. اما CPython یک بهینه‌سازی جالب دارد:

```python
import sys

# پایتون بسته به محتوا، encoding متفاوتی انتخاب می‌کند:

# Latin-1 (1 بایت per کاراکتر)
s1 = "hello"
print(sys.getsizeof(s1))  # 54 bytes

# UCS-2 (2 بایت per کاراکتر) — وقتی کاراکتر غیر-latin داریم
s2 = "héllo"
print(sys.getsizeof(s2))  # حدود 76 bytes

# UCS-4 (4 بایت per کاراکتر) — وقتی emoji داریم
s3 = "hello 🎉"
print(sys.getsizeof(s3))  # حدود 120 bytes
```

#### تمام متدهای str

```python
text = "  Hello, World! Python is Amazing!  "

# ── متدهای case ──
print("hello WORLD".upper())        # HELLO WORLD
print("hello WORLD".lower())        # hello world
print("hello WORLD".swapcase())     # HELLO world
print("hello world".capitalize())   # Hello world
print("hello world".title())        # Hello World

# ── متدهای بررسی ──
print("Hello123".isalpha())    # False (عدد دارد)
print("Hello".isalpha())       # True
print("123".isdigit())         # True
print("123.45".isnumeric())    # False (نقطه دارد)
print("Hello123".isalnum())    # True
print("   ".isspace())         # True
print("Hello World".istitle()) # True
print("HELLO".isupper())       # True
print("hello".islower())       # True

# ── متدهای جستجو ──
s = "Hello, Hello, World"
print(s.find("Hello"))       # 0  (اولین تکرار)
print(s.find("Hello", 1))    # 7  (بعد از اندیس 1)
print(s.find("xyz"))         # -1 (پیدا نشد — استثنا نمی‌دهد)
print(s.rfind("Hello"))      # 7  (آخرین تکرار)

print(s.index("Hello"))      # 0  (مثل find ولی استثنا می‌دهد)
try:
    s.index("xyz")
except ValueError as e:
    print(e)  # substring not found

print(s.count("Hello"))      # 2

# startswith و endswith با tuple هم کار می‌کنند:
print("hello.py".endswith(('.py', '.txt', '.js')))  # True

# ── متدهای trim ──
text = "  Hello, World!  "
print(repr(text.strip()))   # 'Hello, World!'
print(repr(text.lstrip()))  # 'Hello, World!  '
print(repr(text.rstrip()))  # '  Hello, World!'

# با کاراکتر مشخص:
print("***Hello***".strip('*'))   # Hello
print("xxxHelloxxx".strip('x'))   # Hello

# ── متدهای جایگزینی ──
s = "Hello, World! Hello, Python!"
print(s.replace("Hello", "Hi"))         # Hi, World! Hi, Python!
print(s.replace("Hello", "Hi", 1))      # Hi, World! Hello, Python!

# translate — جایگزینی کاراکتر به کاراکتر
table = str.maketrans('aeiou', '12345')
print("hello world".translate(table))  # h2ll4 w4rld

# برای حذف کاراکترها:
table = str.maketrans('', '', 'aeiou')  # حذف vowel‌ها
print("hello world".translate(table))  # hll wrld

# ── متدهای split و join ──
csv = "apple,banana,cherry,date"
fruits = csv.split(',')
print(fruits)  # ['apple', 'banana', 'cherry', 'date']

# با maxsplit:
print(csv.split(',', 2))  # ['apple', 'banana', 'cherry,date']

# rsplit — از راست:
print(csv.rsplit(',', 1))  # ['apple,banana,cherry', 'date']

# splitlines — برای متن چند خطی:
multiline = "line1\nline2\r\nline3\rline4"
print(multiline.splitlines())  # ['line1', 'line2', 'line3', 'line4']

# join — معکوس split:
print(", ".join(fruits))    # apple, banana, cherry, date
print("".join(['H','e','l','l','o']))  # Hello
print("\n".join(["line1", "line2", "line3"]))

# ── متدهای format ──
name, age = "Alice", 30

# f-string (پایتون ۳.۶+) — بهترین روش
print(f"{name} is {age} years old")

# format method
print("{} is {} years old".format(name, age))
print("{name} is {age} years old".format(name=name, age=age))

# format با عملیات:
pi = 3.14159265
print(f"{pi:.2f}")       # 3.14
print(f"{pi:10.2f}")     # '      3.14'  (راست‌چین با عرض 10)
print(f"{pi:<10.2f}")    # '3.14      '  (چپ‌چین)
print(f"{pi:^10.2f}")    # '  3.14    '  (وسط‌چین)
print(f"{1000000:,}")    # 1,000,000
print(f"{255:08b}")      # 11111111  (باینری با padding)
print(f"{255:#010x}")    # 0x000000ff (هگزادسیمال)

# zfill — padding با صفر:
print("42".zfill(5))    # 00042
print("-42".zfill(5))   # -0042

# ljust, rjust, center:
print("Hello".ljust(10))        # 'Hello     '
print("Hello".rjust(10))        # '     Hello'
print("Hello".center(11))       # '   Hello   '
print("Hello".center(11, '*'))  # '***Hello***'

# ── متدهای encode/decode ──
s = "Hello, دنیا!"
encoded = s.encode('utf-8')
print(encoded)           # b'Hello, \xd8\xaf\xd9\x86\xdb\x8c\xd8\xa7!'
print(encoded.decode('utf-8'))  # Hello, دنیا!

# ── partition ──
url = "https://www.example.com/path"
protocol, sep, rest = url.partition("://")
print(protocol)  # https
print(rest)      # www.example.com/path
```

#### String interning

```python
# پایتون بسیاری از string‌های کوچک را intern می‌کند
a = "hello"
b = "hello"
print(a is b)  # True — همان شیء در حافظه

# رشته‌های بزرگتر یا با فضا ممکن است intern نشوند
c = "hello world"
d = "hello world"
print(c is d)  # ممکن است True یا False باشد

# می‌توان صریحاً intern کرد:
import sys
e = sys.intern("hello world")
f = sys.intern("hello world")
print(e is f)  # True
```

---

### ۳.۵ — فرزند چهارم: `range`

`range` یک Sequence **lazy** است — عناصرش را در حافظه ذخیره نمی‌کند بلکه آن‌ها را **محاسبه** می‌کند.

#### چرا range یک Sequence است؟

```python
from collections.abc import Sequence

r = range(10)
print(isinstance(r, Sequence))  # True

# همه ویژگی‌های Sequence را دارد:
print(r[5])          # 5  — دسترسی به اندیس O(1)
print(len(r))        # 10 — طول O(1)
print(5 in r)        # True — بررسی عضویت O(1)!
print(r.index(7))    # 7
print(r.count(5))    # 1
print(r[:5])         # range(0, 5) — slice برمی‌گرداند range نه list
print(list(r))       # [0, 1, 2, 3, 4, 5, 6, 7, 8, 9]
```

**نکته مهم:** `5 in range(1000000000)` فوری است! چون range ریاضی حساب می‌کند نه iterate.

```python
import sys

# range یک عدد خیلی بزرگ:
big_range = range(10**18)
print(sys.getsizeof(big_range))  # فقط 48 bytes! (نه 8 * 10^18 bytes)

# بررسی عضویت سریع:
import time

start = time.time()
result = 10**17 in range(10**18)
end = time.time()
print(f"نتیجه: {result}, زمان: {(end-start)*1000:.3f} ms")  # تقریباً 0 ms!
```

#### ویژگی‌های range

```python
# سه فرم:
r1 = range(5)        # 0, 1, 2, 3, 4
r2 = range(2, 8)     # 2, 3, 4, 5, 6, 7
r3 = range(0, 20, 3) # 0, 3, 6, 9, 12, 15, 18
r4 = range(10, 0, -1) # 10, 9, 8, 7, 6, 5, 4, 3, 2, 1

# ویژگی‌ها:
r = range(2, 20, 3)
print(r.start)  # 2
print(r.stop)   # 20
print(r.step)   # 3

# range‌ها مقایسه‌پذیر هستند:
print(range(3) == range(0, 3, 1))  # True — محتوای یکسان
print(range(0) == range(1, 0))     # True — هر دو خالی

# range قابل hash است:
print(hash(range(10)))  # یک عدد

# range با اعداد منفی:
r = range(10, -1, -1)
print(list(r))  # [10, 9, 8, 7, 6, 5, 4, 3, 2, 1, 0]

# برش از range:
r = range(0, 100, 2)
print(r[5:10])   # range(10, 20, 2)
print(r[-5:])    # range(90, 100, 2)
```

---

### ۳.۶ — فرزند پنجم: `bytes`

`bytes` یک Sequence از **اعداد صحیح ۰ تا ۲۵۵** است. اما وقتی نشان داده می‌شود، مثل یک رشته‌ی ASCII به نظر می‌رسد.

```python
# ساخت bytes
b1 = b"Hello"
b2 = bytes([72, 101, 108, 108, 111])  # همان!
b3 = bytes(5)        # 5 بایت صفر: b'\x00\x00\x00\x00\x00'
b4 = "Hello".encode('utf-8')

print(b1 == b2)  # True
print(b1 == b4)  # True

# هر عنصر یک عدد است:
for byte in b"ABC":
    print(byte, end=" ")  # 65 66 67

# اندیس‌گذاری عدد برمی‌گرداند:
print(b"Hello"[0])  # 72

# slice می‌دهد bytes:
print(b"Hello"[1:4])  # b'ell'

# تبدیل:
print(list(b"Hello"))  # [72, 101, 108, 108, 111]
print(bytes([72, 101, 108, 108, 111]).decode())  # Hello

# عملیات bytes:
b = b"Hello, World!"
print(b.upper())          # B'HELLO, WORLD!'
print(b.replace(b"l", b"L"))  # b'HeLLo, WorLd!'
print(b.split(b","))     # [b'Hello', b' World!']
print(b.hex())           # 48656c6c6f2c20576f726c6421
print(bytes.fromhex("48656c6c6f"))  # b'Hello'
```

---

### ۳.۷ — فرزند ششم: `bytearray`

`bytearray` نسخه‌ی **mutable** از `bytes` است:

```python
ba = bytearray(b"Hello")

# می‌توان تغییر داد:
ba[0] = 74  # J
print(ba)  # bytearray(b'Jello')

ba.append(33)  # !
print(ba)  # bytearray(b'Jello!')

ba.extend(b" World")
print(ba)  # bytearray(b'Jello! World')

# تبدیل به bytes:
b = bytes(ba)
print(b)  # b'Jello! World'
```

---

### ۳.۸ — فرزند هفتم: `memoryview`

`memoryview` یکی از پیشرفته‌ترین انواع Sequence در پایتون است. به شما اجازه می‌دهد **بدون کپی** به داده‌های باینری دسترسی داشته باشید.

```python
import sys

data = bytearray(b"Hello, World!")

# بدون memoryview — کپی ایجاد می‌شود:
sub1 = data[0:5]  # یک bytearray جدید ساخته می‌شود

# با memoryview — بدون کپی:
mv = memoryview(data)
sub2 = mv[0:5]  # فقط یک view — بدون کپی!

print(bytes(sub2))  # b'Hello'

# تغییر از طریق memoryview:
mv[0] = 74  # J
print(data)  # bytearray(b'Jello, World!')

# کاربرد اصلی: کار با داده‌های بزرگ بدون کپی
big_data = bytearray(10**6)  # 1 MB
mv = memoryview(big_data)

# این کپی نمی‌گیرد!
chunk = mv[100:200]
print(sys.getsizeof(chunk))  # کوچک — فقط یک view

# تبدیل فرمت:
import struct
numbers = bytearray(struct.pack('5i', 1, 2, 3, 4, 5))  # 5 integer
mv = memoryview(numbers).cast('i')  # cast به int
print(list(mv))  # [1, 2, 3, 4, 5]
```

---

### ۳.۹ — فرزند هشتم: `array.array`

`array.array` یک Sequence مانند `list` است اما **homogeneous** — همه عناصر باید یک نوع باشند:

```python
import array

# ساخت array با type code
int_array = array.array('i', [1, 2, 3, 4, 5])    # signed int
float_array = array.array('f', [1.0, 2.0, 3.0])  # float
byte_array = array.array('b', [127, -128, 0])     # signed byte

# type codes مهم:
# 'b': signed char (1 byte)
# 'B': unsigned char (1 byte)
# 'h': signed short (2 bytes)
# 'H': unsigned short (2 bytes)
# 'i': signed int (2+ bytes)
# 'I': unsigned int (2+ bytes)
# 'l': signed long (4+ bytes)
# 'f': float (4 bytes)
# 'd': double (8 bytes)

import sys

# مقایسه حافظه با list:
lst = list(range(1000))
arr = array.array('i', range(1000))

print(sys.getsizeof(lst))  # حدود 8056 bytes
print(sys.getsizeof(arr))  # حدود 4064 bytes — نصف!

# عملیات:
arr = array.array('i', [1, 2, 3, 4, 5])
arr.append(6)
arr.extend([7, 8, 9])
arr.insert(0, 0)
print(arr)  # array('i', [0, 1, 2, 3, 4, 5, 6, 7, 8, 9])

# ذخیره و خواندن باینری:
with open("numbers.bin", "wb") as f:
    arr.tofile(f)

arr2 = array.array('i')
with open("numbers.bin", "rb") as f:
    arr2.fromfile(f, len(arr))
print(arr2)  # همان array
```

---

---

## بخش چهارم: MutableSequence — فرزند قدرتمند

### ۴.۱ — تفاوت Sequence و MutableSequence

`Sequence` یعنی **خواندنی**. `MutableSequence` یعنی **خواندنی و نوشتنی**.

جدول مقایسه:

| ویژگی | Sequence | MutableSequence |
|-------|----------|-----------------|
| `__getitem__` | ✅ (abstract) | ✅ (abstract) |
| `__len__` | ✅ (abstract) | ✅ (abstract) |
| `__setitem__` | ❌ | ✅ (abstract) |
| `__delitem__` | ❌ | ✅ (abstract) |
| `insert` | ❌ | ✅ (abstract) |
| `append` | ❌ | ✅ (mixin) |
| `clear` | ❌ | ✅ (mixin) |
| `reverse` | ❌ | ✅ (mixin) |
| `extend` | ❌ | ✅ (mixin) |
| `pop` | ❌ | ✅ (mixin) |
| `remove` | ❌ | ✅ (mixin) |
| `__iadd__` | ❌ | ✅ (mixin) |

پس برای پیاده‌سازی `MutableSequence`، باید **پنج متد** را پیاده‌سازی کنید:
1. `__getitem__`
2. `__len__`
3. `__setitem__`
4. `__delitem__`
5. `insert`

و در ازا، **هفت متد رایگان** می‌گیرید.

### ۴.۲ — پیاده‌سازی داخلی متدهای Mixin در MutableSequence

بیایید ببینیم این متدهای رایگان چطور کار می‌کنند:

```python
# این کد ساده‌سازی‌شده‌ی پیاده‌سازی داخلی است

class MutableSequenceBase(SequenceBase):

    # باید پیاده‌سازی شوند:
    def __setitem__(self, index, value):
        raise NotImplementedError

    def __delitem__(self, index):
        raise NotImplementedError

    def insert(self, index, value):
        raise NotImplementedError

    # متدهای رایگان:

    def append(self, value):
        # درست مثل insert در انتها
        self.insert(len(self), value)

    def clear(self):
        # یک به یک از انتها حذف می‌کند
        try:
            while True:
                self.pop()
        except IndexError:
            pass

    def reverse(self):
        # swap از دو طرف به وسط
        n = len(self)
        for i in range(n // 2):
            self[i], self[n - 1 - i] = self[n - 1 - i], self[i]

    def extend(self, values):
        # یک به یک append می‌کند
        for v in values:
            self.append(v)

    def pop(self, index=-1):
        # مقدار را می‌خواند، حذف می‌کند، برمی‌گرداند
        v = self[index]
        del self[index]
        return v

    def remove(self, value):
        # اولین تکرار را پیدا و حذف می‌کند
        del self[self.index(value)]

    def __iadd__(self, values):
        # += operator
        self.extend(values)
        return self
```

### ۴.۳ — پیاده‌سازی MutableSequence سفارشی: SortedList

بیایید یک `SortedList` بسازیم — لیستی که همیشه مرتب است:

```python
from collections.abc import MutableSequence
import bisect


class SortedList(MutableSequence):
    """
    یک لیست که همیشه مرتب نگه می‌دارد عناصر را.
    از binary search برای درج بهره می‌گیرد — O(log n) برای یافتن جای درج.
    """

    def __init__(self, iterable=None):
        self._data = []
        if iterable is not None:
            self.extend(iterable)

    # ── پنج متد اجباری ──

    def __getitem__(self, index):
        if isinstance(index, slice):
            result = SortedList()
            result._data = self._data[index]
            return result
        return self._data[index]

    def __len__(self):
        return len(self._data)

    def __setitem__(self, index, value):
        # در SortedList نمی‌توان مستقیم set کرد
        # چون ترتیب را خراب می‌کند
        raise TypeError(
            "SortedList does not support item assignment. "
            "Use add() instead."
        )

    def __delitem__(self, index):
        del self._data[index]

    def insert(self, index, value):
        # insert را override می‌کنیم تا ترتیب حفظ شود
        # index را نادیده می‌گیریم و در جای درست insert می‌کنیم
        self.add(value)

    # ── متد اضافه — قلب SortedList ──

    def add(self, value):
        """
        با bisect جای درست را پیدا می‌کند — O(log n)
        اما خود insert در لیست O(n) است.
        """
        position = bisect.bisect_left(self._data, value)
        self._data.insert(position, value)

    # ── override append برای صحت ──

    def append(self, value):
        # append در MutableSequence از insert استفاده می‌کند
        # که ما add را صدا می‌زند
        self.add(value)

    # ── متدهای اضافی ──

    def __contains__(self, value):
        """
        با binary search بهینه‌تر از iterate کردن است
        O(log n) به جای O(n)
        """
        i = bisect.bisect_left(self._data, value)
        return i < len(self._data) and self._data[i] == value

    def index(self, value, start=0, stop=None):
        """
        با binary search O(log n) پیدا می‌کند
        """
        if stop is None:
            stop = len(self._data)
        i = bisect.bisect_left(self._data, value, start, stop)
        if i < stop and self._data[i] == value:
            return i
        raise ValueError(f"{value!r} is not in SortedList")

    def irange(self, minimum=None, maximum=None):
        """
        تمام عناصر بین minimum و maximum را برمی‌گرداند
        این یک متد اختصاصی SortedList است
        """
        left = bisect.bisect_left(self._data, minimum) \
               if minimum is not None else 0
        right = bisect.bisect_right(self._data, maximum) \
                if maximum is not None else len(self._data)
        return self._data[left:right]

    def __repr__(self):
        return f"SortedList({self._data!r})"


# ── استفاده از SortedList ──

sl = SortedList([5, 3, 1, 4, 2])
print(sl)  # SortedList([1, 2, 3, 4, 5])

sl.add(3)
print(sl)  # SortedList([1, 2, 3, 3, 4, 5])

sl.add(0)
print(sl)  # SortedList([0, 1, 2, 3, 3, 4, 5])

# متدهای رایگان از MutableSequence:
sl.remove(3)       # اولین ۳ را حذف می‌کند
print(sl)          # SortedList([0, 1, 2, 3, 4, 5])

popped = sl.pop()  # آخرین عنصر
print(popped)      # 5
print(sl)          # SortedList([0, 1, 2, 3, 4])

# extend هم کار می‌کند:
sl.extend([10, 7, 8])
print(sl)  # SortedList([0, 1, 2, 3, 4, 7, 8, 10])

# جستجو سریع:
print(7 in sl)     # True  — O(log n)
print(99 in sl)    # False — O(log n)

# بازه:
print(sl.irange(2, 8))  # [2, 3, 4, 7, 8]

# reverse از MutableSequence — اما ترتیب را خراب می‌کند!
# بهتر است override کنیم:
print(list(reversed(sl)))  # [10, 8, 7, 4, 3, 2, 1, 0]

# isinstance:
from collections.abc import Sequence, MutableSequence
print(isinstance(sl, Sequence))         # True
print(isinstance(sl, MutableSequence))  # True
```

---

## بخش پنجم: ساخت Sequence سفارشی — از صفر تا کامل

### ۵.۱ — مثال اول: FixedArray — آرایه با اندازه ثابت

```python
from collections.abc import MutableSequence


class FixedArray(MutableSequence):
    """
    یک آرایه با اندازه ثابت.
    بر خلاف list، نمی‌توان عنصر اضافه یا حذف کرد.
    اما می‌توان مقادیر را تغییر داد.
    """

    def __init__(self, size, default=None):
        self._size = size
        self._data = [default] * size

    # ── پنج متد اجباری ──

    def __getitem__(self, index):
        if isinstance(index, slice):
            # slice برمی‌گردانیم ولی نه FixedArray
            # چون اندازه‌اش ممکن است متفاوت باشد
            return self._data[index]
        self._check_index(index)
        return self._data[index]

    def __setitem__(self, index, value):
        if isinstance(index, slice):
            # برای slice، باید اندازه یکسان باشد
            new_values = list(value)
            indices = range(*index.indices(self._size))
            if len(new_values) != len(indices):
                raise ValueError(
                    f"FixedArray: cannot change size. "
                    f"Expected {len(indices)} items, got {len(new_values)}"
                )
            for i, v in zip(indices, new_values):
                self._data[i] = v
        else:
            self._check_index(index)
            self._data[index] = value

    def __delitem__(self, index):
        raise TypeError("FixedArray does not support item deletion")

    def __len__(self):
        return self._size

    def insert(self, index, value):
        raise TypeError("FixedArray does not support item insertion")

    # ── متد کمکی ──

    def _check_index(self, index):
        if isinstance(index, int):
            if index < 0:
                index += self._size
            if not 0 <= index < self._size:
                raise IndexError(
                    f"FixedArray index {index} out of range "
                    f"for size {self._size}"
                )

    # ── override append و extend تا خطای بهتری بدهند ──

    def append(self, value):
        raise TypeError("FixedArray does not support append")

    def extend(self, values):
        raise TypeError("FixedArray does not support extend")

    def __repr__(self):
        return f"FixedArray({self._size}, data={self._data!r})"

    def __str__(self):
        return str(self._data)


# ── استفاده ──

fa = FixedArray(5, default=0)
print(fa)  # [0, 0, 0, 0, 0]

fa[0] = 10
fa[1] = 20
fa[-1] = 99
print(fa)  # [10, 20, 0, 0, 99]

fa[1:4] = [100, 200, 300]
print(fa)  # [10, 100, 200, 300, 99]

# متدهای رایگان:
print(list(fa))          # [10, 100, 200, 300, 99]
print(list(reversed(fa))) # [99, 300, 200, 100, 10]
print(200 in fa)         # True
print(fa.index(300))     # 3
print(fa.count(0))       # 0

# این‌ها خطا می‌دهند:
try:
    fa.append(5)
except TypeError as e:
    print(e)  # FixedArray does not support append

try:
    del fa[2]
except TypeError as e:
    print(e)  # FixedArray does not support item deletion
```

### ۵.۲ — مثال دوم: CircularBuffer — بافر دایره‌ای

یکی از کاربردی‌ترین ساختارهای داده که می‌توان با `MutableSequence` پیاده‌سازی کرد:

```python
from collections.abc import MutableSequence


class CircularBuffer(MutableSequence):
    """
    یک بافر دایره‌ای با اندازه ثابت.
    وقتی پر می‌شود، قدیمی‌ترین عنصر را جایگزین می‌کند.
    
    کاربرد: ذخیره‌ی آخرین N لاگ، نمونه‌های سنسور، و...
    """

    def __init__(self, capacity):
        if capacity <= 0:
            raise ValueError("Capacity must be positive")
        self._capacity = capacity
        self._data = [None] * capacity
        self._start = 0   # اندیس اولین عنصر واقعی
        self._size = 0    # تعداد عناصر واقعی

    def _real_index(self, logical_index):
        """تبدیل اندیس منطقی به اندیس واقعی در آرایه"""
        if logical_index < 0:
            logical_index += self._size
        if not 0 <= logical_index < self._size:
            raise IndexError("CircularBuffer index out of range")
        return (self._start + logical_index) % self._capacity

    # ── پنج متد اجباری ──

    def __getitem__(self, index):
        if isinstance(index, slice):
            indices = range(*index.indices(self._size))
            return [self._data[self._real_index(i)] for i in indices]
        return self._data[self._real_index(index)]

    def __setitem__(self, index, value):
        if isinstance(index, slice):
            indices = range(*index.indices(self._size))
            values = list(value)
            if len(values) != len(indices):
                raise ValueError("slice assignment size mismatch")
            for i, v in zip(indices, values):
                self._data[self._real_index(i)] = v
        else:
            self._data[self._real_index(index)] = value

    def __delitem__(self, index):
        # حذف یک عنصر از وسط بافر دایره‌ای پیچیده است
        if isinstance(index, slice):
            # برای سادگی، slice‌ها را به صورت معکوس حذف می‌کنیم
            indices = sorted(range(*index.indices(self._size)), reverse=True)
            for i in indices:
                self._del_single(i)
        else:
            self._del_single(index)

    def _del_single(self, logical_index):
        """حذف یک عنصر و shift کردن بقیه"""
        real_idx = self._real_index(logical_index)
        # عناصر بعد از این را به چپ shift می‌کنیم
        for i in range(logical_index, self._size - 1):
            curr = self._real_index(i)
            next_idx = self._real_index(i + 1)
            self._data[curr] = self._data[next_idx]
        self._size -= 1

    def __len__(self):
        return self._size

    def insert(self, index, value):
        """
        اگر بافر پر نیست: عنصر را insert می‌کند
        اگر پر است: قدیمی‌ترین عنصر را جایگزین می‌کند
        """
        if self._size < self._capacity:
            # هنوز جا داریم
            # عناصر بعد از index را به راست shift می‌کنیم
            self._size += 1
            for i in range(self._size - 1, index, -1):
                real_curr = self._real_index(i)
                real_prev = self._real_index(i - 1)
                self._data[real_curr] = self._data[real_prev]
            real_insert = self._real_index(index)
            self._data[real_insert] = value
        else:
            # بافر پر است
            # اگر insert به انتها است، قدیمی‌ترین را جایگزین کن
            end_pos = (self._start + self._size) % self._capacity
            self._data[end_pos] = value
            self._start = (self._start + 1) % self._capacity

    # ── override append برای بافر دایره‌ای ──

    def append(self, value):
        """اضافه کردن به انتها — اگر پر بود، قدیمی‌ترین را حذف می‌کند"""
        if self._size < self._capacity:
            end_pos = (self._start + self._size) % self._capacity
            self._data[end_pos] = value
            self._size += 1
        else:
            # بافر پر است: روی قدیمی‌ترین بنویس و start را جلو ببر
            self._data[self._start] = value
            self._start = (self._start + 1) % self._capacity

    @property
    def capacity(self):
        return self._capacity

    @property
    def is_full(self):
        return self._size == self._capacity

    def __repr__(self):
        items = list(self)
        return (f"CircularBuffer(capacity={self._capacity}, "
                f"items={items!r})")


# ── استفاده ──

cb = CircularBuffer(5)

# اضافه کردن عناصر
for i in range(5):
    cb.append(i)
    print(f"بعد از append({i}): {list(cb)}")

# خروجی:
# بعد از append(0): [0]
# بعد از append(1): [0, 1]
# بعد از append(2): [0, 1, 2]
# بعد از append(3): [0, 1, 2, 3]
# بعد از append(4): [0, 1, 2, 3, 4]

print(f"\nبافر پر است: {cb.is_full}")  # True

# حالا اضافه کردن وقتی پر است:
cb.append(99)
print(f"\nبعد از append(99): {list(cb)}")
# بعد از append(99): [1, 2, 3, 4, 99]  — ۰ حذف شد!

cb.append(100)
print(f"بعد از append(100): {list(cb)}")
# بعد از append(100): [2, 3, 4, 99, 100]

# اندیس‌گذاری:
print(f"\nاولین عنصر: {cb[0]}")   # 2
print(f"آخرین عنصر: {cb[-1]}")   # 100

# متدهای رایگان:
print(f"99 in cb: {99 in cb}")    # True
print(f"cb.index(4): {cb.index(4)}")  # 2
print(f"slice: {cb[1:4]}")        # [3, 4, 99]

# کاربرد واقعی: ذخیره آخرین N لاگ
log_buffer = CircularBuffer(3)
for msg in ["error1", "warning1", "info1", "error2", "info2"]:
    log_buffer.append(msg)

print(f"\nآخرین ۳ لاگ: {list(log_buffer)}")
# آخرین ۳ لاگ: ['info1', 'error2', 'info2']
```

### ۵.۳ — مثال سوم: LazySequence — دنباله تنبل

یک Sequence که عناصرش را **فقط وقتی نیاز است** محاسبه می‌کند:

```python
from collections.abc import Sequence
from typing import Callable, TypeVar, Generic

T = TypeVar('T')


class LazySequence(Sequence):
    """
    یک Sequence که عناصر را lazy محاسبه و cache می‌کند.
    مناسب برای محاسبات سنگین که ممکن است به همه عناصر نیاز نداشته باشیم.
    """

    def __init__(self, length: int, compute_fn: Callable[[int], T]):
        """
        length: تعداد عناصر
        compute_fn: تابعی که با دادن اندیس، مقدار را محاسبه می‌کند
        """
        self._length = length
        self._compute = compute_fn
        self._cache = {}
        self._computed_count = 0

    def __getitem__(self, index):
        if isinstance(index, slice):
            indices = range(*index.indices(self._length))
            return [self[i] for i in indices]

        # نرمال‌سازی اندیس منفی
        if index < 0:
            index += self._length
        if not 0 <= index < self._length:
            raise IndexError(f"index {index} out of range")

        # اگر cache داریم، از آن استفاده می‌کنیم
        if index not in self._cache:
            self._cache[index] = self._compute(index)
            self._computed_count += 1

        return self._cache[index]

    def __len__(self):
        return self._length

    @property
    def cache_info(self):
        return {
            "total": self._length,
            "computed": self._computed_count,
            "hit_rate": (
                f"{(1 - self._computed_count / max(1, len(self._cache))) * 100:.1f}%"
            )
        }

    def precompute(self, start=0, stop=None):
        """از پیش محاسبه کردن یک بازه"""
        if stop is None:
            stop = self._length
        for i in range(start, stop):
            _ = self[i]  # این cache را پر می‌کند

    def __repr__(self):
        computed = sorted(self._cache.items())[:5]
        preview = {k: v for k, v in computed}
        return (f"LazySequence(length={self._length}, "
                f"computed={self._computed_count}, "
                f"preview={preview})")


# ── استفاده ──

import time
import math


def expensive_computation(n):
    """یک محاسبه سنگین (شبیه‌سازی شده)"""
    time.sleep(0.001)  # ۱ میلی‌ثانیه تاخیر
    return math.factorial(n) % (10**10)  # factorial بزرگ


# ساخت یک دنباله ۱۰۰۰ عنصری
lazy = LazySequence(1000, expensive_computation)

# فقط به چند عنصر نیاز داریم:
start = time.time()
print(lazy[0])    # محاسبه می‌شود
print(lazy[500])  # محاسبه می‌شود
print(lazy[999])  # محاسبه می‌شود
print(lazy[0])    # از cache — سریع!
print(lazy[500])  # از cache — سریع!
elapsed = time.time() - start

print(f"\nزمان: {elapsed:.3f}s")  # حدود ۰.۰۰۳ ثانیه
print(f"اطلاعات cache: {lazy.cache_info}")
# {'total': 1000, 'computed': 3, ...}

# بدون LazySequence، اگر همه را محاسبه می‌کردیم:
# ۱۰۰۰ * ۰.۰۰۱ = ۱ ثانیه!


# ── مثال عملی: اعداد اول ──

def is_prime(n):
    if n < 2:
        return False
    for i in range(2, int(n**0.5) + 1):
        if n % i == 0:
            return False
    return True


def nth_prime(n):
    """اعداد اول را به ترتیب برمی‌گرداند"""
    count = 0
    candidate = 2
    while True:
        if is_prime(candidate):
            if count == n:
                return candidate
            count += 1
        candidate += 1


primes = LazySequence(100, nth_prime)
print(f"\nاولین اول: {primes[0]}")    # 2
print(f"دهمین اول: {primes[9]}")     # 29
print(f"صدمین اول: {primes[99]}")    # 541
print(f"اول‌های ۵ تا ۱۰: {primes[5:10]}")  # [13, 17, 19, 23, 29]
```

### ۵.۴ — مثال چهارم: TypedList — لیست با نوع مشخص

```python
from collections.abc import MutableSequence
from typing import Type, TypeVar, Generic, Iterator

T = TypeVar('T')


class TypedList(MutableSequence, Generic[T]):
    """
    یک لیست که فقط نوع مشخصی را قبول می‌کند.
    مثل List[int] در زبان‌های statically typed.
    """

    def __init__(self, item_type: Type[T], iterable=None):
        self._type = item_type
        self._data = []
        if iterable is not None:
            self.extend(iterable)

    def _validate(self, value):
        if not isinstance(value, self._type):
            raise TypeError(
                f"TypedList[{self._type.__name__}] "
                f"cannot accept {type(value).__name__!r} — "
                f"expected {self._type.__name__!r}"
            )

    def __getitem__(self, index):
        if isinstance(index, slice):
            result = TypedList(self._type)
            result._data = self._data[index]
            return result
        return self._data[index]

    def __setitem__(self, index, value):
        if isinstance(index, slice):
            values = list(value)
            for v in values:
                self._validate(v)
            self._data[index] = values
        else:
            self._validate(value)
            self._data[index] = value

    def __delitem__(self, index):
        del self._data[index]

    def __len__(self):
        return len(self._data)

    def insert(self, index, value):
        self._validate(value)
        self._data.insert(index, value)

    def __repr__(self):
        return f"TypedList[{self._type.__name__}]({self._data!r})"


# ── استفاده ──

# لیست فقط اعداد صحیح
int_list: TypedList[int] = TypedList(int, [1, 2, 3, 4, 5])
print(int_list)  # TypedList[int]([1, 2, 3, 4, 5])

int_list.append(6)
print(int_list)  # TypedList[int]([1, 2, 3, 4, 5, 6])

# این خطا می‌دهد:
try:
    int_list.append("hello")
except TypeError as e:
    print(e)
    # TypedList[int] cannot accept 'str' — expected 'int'

try:
    int_list[0] = 3.14
except TypeError as e:
    print(e)
    # TypedList[int] cannot accept 'float' — expected 'int'

# لیست فقط رشته
str_list: TypedList[str] = TypedList(str)
str_list.extend(["hello", "world"])
str_list.insert(1, "beautiful")
print(str_list)  # TypedList[str](['hello', 'beautiful', 'world'])

# متدهای رایگان:
print(str_list.count("hello"))   # 1
print(str_list.index("world"))   # 2
str_list.reverse()
print(str_list)  # TypedList[str](['world', 'beautiful', 'hello'])
```

---

## بخش ششم: الگوهای پیشرفته با Sequence

### ۶.۱ — الگوی Proxy — پوشش یک Sequence موجود

```python
from collections.abc import MutableSequence


class LoggedList(MutableSequence):
    """
    یک پوشش روی list که تمام عملیات را لاگ می‌کند.
    الگوی Proxy/Decorator
    """

    def __init__(self, data=None, logger=None):
        self._data = list(data) if data else []
        self._logger = logger or print
        self._operations = []

    def _log(self, operation, *args):
        msg = f"[LoggedList] {operation}: {args}"
        self._logger(msg)
        self._operations.append({"op": operation, "args": args})

    def __getitem__(self, index):
        result = self._data[index]
        self._log("__getitem__", index, "->", result)
        return result

    def __setitem__(self, index, value):
        old = self._data[index]
        self._data[index] = value
        self._log("__setitem__", index, f"{old!r} -> {value!r}")

    def __delitem__(self, index):
        old = self._data[index]
        del self._data[index]
        self._log("__delitem__", index, f"removed {old!r}")

    def __len__(self):
        return len(self._data)

    def insert(self, index, value):
        self._data.insert(index, value)
        self._log("insert", index, value)

    def operation_history(self):
        return self._operations.copy()

    def __repr__(self):
        return f"LoggedList({self._data!r})"


# ── استفاده ──

ll = LoggedList([1, 2, 3])
ll.append(4)         # [LoggedList] insert: (3, 4)
ll[0] = 99          # [LoggedList] __setitem__: (0, '1 -> 99')
del ll[1]           # [LoggedList] __delitem__: (1, "removed 2")
print(ll)           # LoggedList([99, 3, 4])
```

### ۶.۲ — الگوی Composite — Sequence از Sequence‌ها

```python
from collections.abc import Sequence


class ChainedSequence(Sequence):
    """
    چندین Sequence را به هم وصل می‌کند بدون کپی گرفتن.
    مثل itertools.chain اما با دسترسی به اندیس.
    """

    def __init__(self, *sequences):
        self._seqs = [list(s) if not isinstance(s, Sequence)
                      else s for s in sequences]
        # محاسبه offset برای هر دنباله
        self._offsets = []
        offset = 0
        for seq in self._seqs:
            self._offsets.append(offset)
            offset += len(seq)
        self._total_len = offset

    def _find_seq_and_index(self, index):
        """اندیس کلی را به اندیس محلی تبدیل می‌کند"""
        if index < 0:
            index += self._total_len
        if not 0 <= index < self._total_len:
            raise IndexError("ChainedSequence index out of range")

        # binary search برای پیدا کردن دنباله مناسب
        import bisect
        seq_idx = bisect.bisect_right(self._offsets, index) - 1
        local_idx = index - self._offsets[seq_idx]
        return seq_idx, local_idx

    def __getitem__(self, index):
        if isinstance(index, slice):
            indices = range(*index.indices(self._total_len))
            return [self[i] for i in indices]
        seq_idx, local_idx = self._find_seq_and_index(index)
        return self._seqs[seq_idx][local_idx]

    def __len__(self):
        return self._total_len

    def __repr__(self):
        return f"ChainedSequence({self._seqs!r})"


# ── استفاده ──

a = [1, 2, 3]
b = (4, 5, 6)
c = range(7, 10)

chain = ChainedSequence(a, b, c)
print(len(chain))         # 9
print(chain[0])           # 1
print(chain[4])           # 5
print(chain[-1])          # 9
print(chain[2:7])         # [3, 4, 5, 6, 7]
print(list(chain))        # [1, 2, 3, 4, 5, 6, 7, 8, 9]
print(5 in chain)         # True  — از __contains__ رایگان


# بدون ChainedSequence، باید کپی می‌گرفتیم:
merged = list(a) + list(b) + list(c)  # حافظه اضافه مصرف می‌کند
```

### ۶.۳ — الگوی View — نمای فیلترشده

```python
from collections.abc import Sequence


class FilteredView(Sequence):
    """
    یک نمای فیلترشده از یک Sequence موجود.
    تغییرات در منبع اصلی منعکس می‌شوند.
    بدون کپی گرفتن از داده.
    """

    def __init__(self, source: Sequence, predicate):
        self._source = source
        self._predicate = predicate
        # محاسبه اندیس‌های معتبر
        self._indices = [i for i, x in enumerate(source)
                         if predicate(x)]

    def refresh(self):
        """به‌روزرسانی وقتی source تغییر کرده"""
        self._indices = [i for i, x in enumerate(self._source)
                         if self._predicate(x)]

    def __getitem__(self, index):
        if isinstance(index, slice):
            indices = range(*index.indices(len(self)))
            return [self._source[self._indices[i]] for i in indices]
        if index < 0:
            index += len(self._indices)
        if not 0 <= index < len(self._indices):
            raise IndexError("FilteredView index out of range")
        return self._source[self._indices[index]]

    def __len__(self):
        return len(self._indices)

    def __repr__(self):
        return f"FilteredView({list(self)!r})"


# ── استفاده ──

numbers = [1, 2, 3, 4, 5, 6, 7, 8, 9, 10]

# نمای اعداد زوج
evens = FilteredView(numbers, lambda x: x % 2 == 0)
print(evens)       # FilteredView([2, 4, 6, 8, 10])
print(len(evens))  # 5
print(evens[2])    # 6
print(evens[-1])   # 10

# نمای اعداد اول
primes_view = FilteredView(numbers, is_prime)
print(primes_view)  # FilteredView([2, 3, 5, 7])

# تغییر source و refresh:
numbers.extend([11, 12, 13, 14, 15])
evens.refresh()
print(evens)   # FilteredView([2, 4, 6, 8, 10, 12, 14])

# ترکیب فیلترها:
large_primes = FilteredView(primes_view, lambda x: x > 5)
large_primes.refresh()
print(large_primes)  # FilteredView([7])
```

### ۶.۴ — Sequence و Protocol Structural Subtyping

در پایتون مدرن (۳.۸+) می‌توان با `typing.Protocol` یک Sequence بدون ارث‌بری از ABC تعریف کرد:

```python
from typing import Protocol, runtime_checkable, overload
from typing import TypeVar, Iterator

T_co = TypeVar('T_co', covariant=True)


@runtime_checkable
class SequenceProtocol(Protocol[T_co]):
    """
    پروتکل برای Sequence — structural subtyping
    هر کلاسی که این متدها را داشته باشد، Sequence است
    بدون نیاز به ارث‌بری
    """

    def __getitem__(self, index: int) -> T_co: ...
    def __len__(self) -> int: ...


# هر کلاسی با این دو متد، SequenceProtocol است:
class MyData:
    def __init__(self):
        self._items = [10, 20, 30]

    def __getitem__(self, index):
        return self._items[index]

    def __len__(self):
        return len(self._items)


d = MyData()
print(isinstance(d, SequenceProtocol))  # True — بدون ارث‌بری!
```

### ۶.۵ — بهینه‌سازی: __slots__ با Sequence

```python
from collections.abc import Sequence


class OptimizedPoint(Sequence):
    """
    یک نقطه در فضای سه‌بعدی با __slots__
    برای صرفه‌جویی در حافظه
    """
    __slots__ = ('_x', '_y', '_z')

    def __init__(self, x, y, z):
        self._x = x
        self._y = y
        self._z = z

    def __getitem__(self, index):
        return (self._x, self._y, self._z)[index]

    def __len__(self):
        return 3

    def __repr__(self):
        return f"Point({self._x}, {self._y}, {self._z})"


import sys

# مقایسه حافظه:
class NormalPoint(Sequence):
    def __init__(self, x, y, z):
        self._x = x
        self._y = y
        self._z = z

    def __getitem__(self, i):
        return (self._x, self._y, self._z)[i]

    def __len__(self):
        return 3


p1 = OptimizedPoint(1.0, 2.0, 3.0)
p2 = NormalPoint(1.0, 2.0, 3.0)

print(sys.getsizeof(p1))  # حدود 56 bytes — بدون __dict__
print(sys.getsizeof(p2))  # حدود 48 bytes + __dict__ حدود 232 bytes

# استفاده:
p = OptimizedPoint(1, 2, 3)
print(list(p))            # [1, 2, 3]
print(p[0], p[-1])        # 1 3
x, y, z = p              # unpacking کار می‌کند!
print(x, y, z)            # 1 2 3
```

---


---

## بخش هفتم: موارد گوشه‌ای و دام‌های پنهان

### ۷.۱ — دام اول: Slice در __getitem__

یکی از رایج‌ترین اشتباهات، فراموش کردن پشتیبانی از `slice` در `__getitem__` است:

```python
from collections.abc import Sequence


# ── نسخه اشتباه ──
class BadSequence(Sequence):
    def __init__(self, data):
        self._data = list(data)

    def __getitem__(self, index):
        # فقط اندیس عددی هندل می‌شود!
        return self._data[index]  # اگر list باشد این کار می‌کند
                                  # اما اگر ساختار دیگری بود...

    def __len__(self):
        return len(self._data)


bs = BadSequence([1, 2, 3, 4, 5])

# این کار می‌کند چون list خودش slice را هندل می‌کند:
print(bs[1:3])   # [2, 3] — list برمی‌گرداند نه BadSequence

# اما نوع برگشتی اشتباه است:
print(type(bs[1:3]))  # <class 'list'> — نه BadSequence!


# ── نسخه درست ──
class GoodSequence(Sequence):
    def __init__(self, data):
        self._data = list(data)

    def __getitem__(self, index):
        if isinstance(index, slice):
            # slice برمی‌گردانیم از همان نوع
            return GoodSequence(self._data[index])
        # اندیس منفی هم باید هندل شود
        if index < 0:
            index += len(self._data)
        if not 0 <= index < len(self._data):
            raise IndexError(f"index {index} out of range")
        return self._data[index]

    def __len__(self):
        return len(self._data)

    def __repr__(self):
        return f"GoodSequence({self._data!r})"


gs = GoodSequence([1, 2, 3, 4, 5])
print(gs[1:3])        # GoodSequence([2, 3])
print(type(gs[1:3]))  # <class 'GoodSequence'>
print(gs[-2:])        # GoodSequence([4, 5])
print(gs[::2])        # GoodSequence([1, 3, 5])


# ── بررسی slice.indices ──
# متد indices سه مقدار (start, stop, step) بهنجارشده برمی‌گرداند

s = slice(None, None, -1)  # یعنی [::-1]
indices = s.indices(5)      # برای دنباله‌ای با طول 5
print(indices)  # (4, -1, -1) — از 4 تا -1 (نه شامل) با گام -1

# استفاده در __getitem__:
def handle_slice(data, s):
    start, stop, step = s.indices(len(data))
    return [data[i] for i in range(start, stop, step)]

print(handle_slice([10, 20, 30, 40, 50], slice(1, 4)))    # [20, 30, 40]
print(handle_slice([10, 20, 30, 40, 50], slice(None, None, -1)))  # [50, 40, 30, 20, 10]
```

### ۷.۲ — دام دوم: تغییر Sequence حین iterate

```python
# ── دام مشهور: تغییر list حین for loop ──

numbers = [1, 2, 3, 4, 5, 6, 7, 8, 9, 10]

# اشتباه: حذف حین iterate
for n in numbers:
    if n % 2 == 0:
        numbers.remove(n)  # مشکل ساز!

print(numbers)  # [1, 3, 5, 7, 9] -- به نظر درست است
                # اما در واقع برخی عناصر skip شدند!

# چرا اشتباه است؟ بیایید با print ببینیم:
numbers = [1, 2, 3, 4, 5, 6]
for i, n in enumerate(numbers):
    print(f"i={i}, n={n}, list={numbers}")
    if n % 2 == 0:
        numbers.remove(n)

# خروجی:
# i=0, n=1, list=[1, 2, 3, 4, 5, 6]
# i=1, n=2, list=[1, 2, 3, 4, 5, 6]  -> حذف 2
# i=2, n=4, list=[1, 3, 4, 5, 6]     -> 3 skip شد!
# i=3, n=5, list=[1, 3, 5, 6]        -> حذف 4، 5 skip شد!
# i=4, n=6, list=[1, 3, 5, 6]        -> حذف 6

# راه‌حل ۱: کپی گرفتن
numbers = [1, 2, 3, 4, 5, 6]
for n in numbers[:]:  # یک کپی می‌سازیم
    if n % 2 == 0:
        numbers.remove(n)
print(numbers)  # [1, 3, 5]  -- درست!

# راه‌حل ۲: list comprehension (بهترین روش)
numbers = [1, 2, 3, 4, 5, 6]
numbers = [n for n in numbers if n % 2 != 0]
print(numbers)  # [1, 3, 5]

# راه‌حل ۳: filter
numbers = [1, 2, 3, 4, 5, 6]
numbers = list(filter(lambda n: n % 2 != 0, numbers))
print(numbers)  # [1, 3, 5]

# راه‌حل ۴: حذف از آخر به اول
numbers = [1, 2, 3, 4, 5, 6]
for i in range(len(numbers) - 1, -1, -1):
    if numbers[i] % 2 == 0:
        del numbers[i]
print(numbers)  # [1, 3, 5]
```

### ۷.۳ — دام سوم: مقایسه هویت vs مقایسه مقدار

```python
# ── is vs == ──

# is: بررسی هویت (آیا همان شیء در حافظه هستند؟)
# ==: بررسی مقدار (آیا مقدارشان برابر است؟)

a = [1, 2, 3]
b = [1, 2, 3]
c = a

print(a == b)  # True  -- مقدار یکسان
print(a is b)  # False -- دو شیء متفاوت در حافظه
print(a is c)  # True  -- همان شیء!

# دام با None:
def find_item(lst, value):
    # اشتباه: اگر اندیس 0 باشد، False برمی‌گردد!
    result = None
    for i, item in enumerate(lst):
        if item == value:
            result = i
            break
    if result:  # اشتباه! 0 هم falsy است
        return result
    return -1

print(find_item([5, 3, 1], 5))  # -1 -- اشتباه! باید 0 باشد

# درست:
def find_item_correct(lst, value):
    for i, item in enumerate(lst):
        if item == value:
            return i
    return -1

print(find_item_correct([5, 3, 1], 5))  # 0 -- درست!

# ── دام با __contains__ و None/False/0 ──
mixed = [0, False, None, "", [], ()]

print(0 in mixed)     # True
print(False in mixed) # True -- چون 0 == False
print(None in mixed)  # True

# نکته ظریف: False و 0 برابر هستند!
print(0 == False)   # True
print(0 is False)   # False

# در __contains__ پایتون از == استفاده می‌کند:
print([0].count(False))   # 1 -- چون 0 == False
print([False].count(0))   # 1
```

### ۷.۴ — دام چهارم: hashability و tuple

```python
# ── کِی tuple قابل hash است؟ ──

# tuple خودش hashable است اگر همه عناصرش hashable باشند
t1 = (1, 2, 3)
print(hash(t1))   # یک عدد

t2 = (1, "hello", (2, 3))
print(hash(t2))   # یک عدد -- tuple تودرتو هم ok است

t3 = (1, [2, 3])  # list داخل tuple
try:
    hash(t3)
except TypeError as e:
    print(e)  # unhashable type: 'list'

# ── این یعنی tuple با list نمی‌تواند کلید dict باشد ──
try:
    d = {(1, [2, 3]): "value"}
except TypeError as e:
    print(e)  # unhashable type: 'list'

# ── اما می‌تواند در set باشد اگر همه عناصر hashable باشند ──
valid_set = {(1, 2), (3, 4), (5, 6)}
print((1, 2) in valid_set)   # True
print((7, 8) in valid_set)   # False


# ── frozenset به جای set برای hashability ──
# اگر می‌خواهید یک set را در tuple بگذارید، از frozenset استفاده کنید:
t = (1, frozenset([2, 3, 4]))
print(hash(t))  # کار می‌کند!
```

### ۷.۵ — دام پنجم: کپی سطحی در Sequence

```python
import copy

# ── کپی سطحی ──
original = [[1, 2], [3, 4], [5, 6]]

# روش‌های مختلف کپی سطحی -- همه یکسان هستند:
copy1 = original[:]
copy2 = original.copy()
copy3 = list(original)
copy4 = copy.copy(original)

# همه کپی‌های سطحی هستند:
copy1[0].append(99)
print(original)  # [[1, 2, 99], [3, 4], [5, 6]] -- تغییر کرد!
print(copy2)     # [[1, 2, 99], [3, 4], [5, 6]] -- آن هم تغییر کرد!

# ── کپی عمیق ──
original = [[1, 2], [3, 4], [5, 6]]
deep = copy.deepcopy(original)
deep[0].append(99)
print(original)  # [[1, 2], [3, 4], [5, 6]] -- تغییر نکرد!
print(deep)      # [[1, 2, 99], [3, 4], [5, 6]]

# ── کِی کپی عمیق لازم است؟ ──
# وقتی عناصر mutable هستند و می‌خواهید تغییرات مستقل باشند

# ── مثال با Sequence سفارشی ──
class DeepCopyableList(list):
    def __copy__(self):
        # کپی سطحی
        return DeepCopyableList(self)

    def __deepcopy__(self, memo):
        # کپی عمیق
        return DeepCopyableList(copy.deepcopy(item, memo) for item in self)


dcl = DeepCopyableList([[1, 2], [3, 4]])
shallow = copy.copy(dcl)
deep_copy = copy.deepcopy(dcl)

dcl[0].append(99)
print(f"original: {dcl}")      # [[1, 2, 99], [3, 4]]
print(f"shallow: {shallow}")   # [[1, 2, 99], [3, 4]] -- تغییر کرد
print(f"deep: {deep_copy}")    # [[1, 2], [3, 4]] -- تغییر نکرد
```

### ۷.۶ — دام ششم: index و count در Sequence با NaN

```python
import math

# NaN مشکل عجیبی دارد: با هیچ چیز (حتی خودش) برابر نیست!
nan = float('nan')
print(nan == nan)   # False -- عجیب!
print(nan is nan)   # True  -- همان شیء است

lst = [1.0, nan, 3.0, nan, 5.0]

# count با NaN غلط کار می‌کند:
print(lst.count(nan))   # 0 -- انتظار داریم 2 باشد!

# index با NaN غلط کار می‌کند:
try:
    print(lst.index(nan))  # ValueError -- انتظار داریم 1 باشد!
except ValueError:
    print("NaN پیدا نشد!")

# چرا؟ چون count و index از == استفاده می‌کنند
# و nan == nan همیشه False است

# راه‌حل: استفاده از math.isnan
def count_nan(lst):
    return sum(1 for x in lst if isinstance(x, float) and math.isnan(x))

def index_nan(lst):
    for i, x in enumerate(lst):
        if isinstance(x, float) and math.isnan(x):
            return i
    raise ValueError("NaN not found")

print(count_nan(lst))   # 2
print(index_nan(lst))   # 1

# همین مشکل در numpy هم وجود دارد:
# np.nan != np.nan  -->  True
```

### ۷.۷ — دام هفتم: sort پایدار است اما...

```python
# Timsort در پایتون پایدار (stable) است
# یعنی عناصر با مقدار یکسان ترتیب نسبی‌شان حفظ می‌شود

data = [
    ("Alice", 30),
    ("Bob", 25),
    ("Charlie", 30),
    ("Dave", 25),
    ("Eve", 30),
]

# مرتب‌سازی بر اساس سن -- ترتیب نسبی حفظ می‌شود:
data.sort(key=lambda x: x[1])
print(data)
# [('Bob', 25), ('Dave', 25), ('Alice', 30), ('Charlie', 30), ('Eve', 30)]
# ← Bob قبل از Dave (هر دو 25)
# ← Alice قبل از Charlie قبل از Eve (هر سه 30)


# دام: sort درجا تغییر می‌دهد، sorted یک لیست جدید می‌سازد
original = [3, 1, 4, 1, 5, 9]

sorted_new = sorted(original)    # لیست جدید
original.sort()                  # درجا

# وقتی از sort استفاده می‌کنید، None برمی‌گرداند:
lst = [3, 1, 2]
result = lst.sort()
print(result)  # None -- نه lst مرتب‌شده!

# این اشتباه رایج است:
# wrong = [3, 1, 2].sort()  # None!
# correct = sorted([3, 1, 2])  # [1, 2, 3]


# ── مرتب‌سازی چندگانه ──
# چون sort پایدار است، می‌توانیم چند بار مرتب کنیم:
data = [
    ("Alice", 30, "Engineer"),
    ("Bob", 25, "Designer"),
    ("Charlie", 30, "Engineer"),
    ("Dave", 25, "Engineer"),
]

# اول بر اساس شغل، بعد سن (ترتیب معکوس اعمال):
data.sort(key=lambda x: x[1])      # مرحله ۲: سن
data.sort(key=lambda x: x[2])      # مرحله ۱: شغل

# نتیجه: بر اساس شغل مرتب، و در هر شغل بر اساس سن
print(data)
```

---

## بخش هشتم: مقایسه عملکرد

### ۸.۱ — list vs tuple: سرعت ساخت

```python
import timeit
import sys


def benchmark(name, code, setup="", n=100000):
    time = timeit.timeit(code, setup=setup, number=n)
    print(f"{name:40s}: {time*1000:.2f} ms")


print("=" * 60)
print("ساخت (۱۰۰,۰۰۰ بار)")
print("=" * 60)

benchmark("list literal [1,2,3,4,5]",
          "[1, 2, 3, 4, 5]")
benchmark("tuple literal (1,2,3,4,5)",
          "(1, 2, 3, 4, 5)")
benchmark("list() از range",
          "list(range(10))")
benchmark("tuple() از range",
          "tuple(range(10))")

# خروجی تقریبی:
# list literal [1,2,3,4,5]              : 8.32 ms
# tuple literal (1,2,3,4,5)             : 3.14 ms  <- سریع‌تر!
# list() از range                       : 42.15 ms
# tuple() از range                      : 38.92 ms


print("\n" + "=" * 60)
print("حافظه")
print("=" * 60)

sizes = [10, 100, 1000, 10000]
for n in sizes:
    lst = list(range(n))
    tpl = tuple(range(n))
    lst_size = sys.getsizeof(lst)
    tpl_size = sys.getsizeof(tpl)
    print(f"n={n:6d}: list={lst_size:8d}B, "
          f"tuple={tpl_size:8d}B, "
          f"نسبت={lst_size/tpl_size:.2f}x")

# خروجی تقریبی:
# n=    10: list=    184B, tuple=    160B, نسبت=1.15x
# n=   100: list=    920B, tuple=    840B, نسبت=1.10x
# n=  1000: list=   8056B, tuple=   8040B, نسبت=1.00x
```

### ۸.۲ — list vs deque: سرعت عملیات

```python
from collections import deque
import timeit

n = 10000

print("=" * 60)
print(f"عملیات روی {n} عنصر")
print("=" * 60)

# append به انتها
benchmark("list.append",
          "lst.append(1)",
          setup=f"lst = list(range({n}))",
          n=100000)
benchmark("deque.append",
          "dq.append(1)",
          setup=f"from collections import deque; dq = deque(range({n}))",
          n=100000)

# insert در ابتدا
benchmark("list.insert(0, x)  [O(n)]",
          "lst.insert(0, 1)",
          setup=f"lst = list(range({n}))")
benchmark("deque.appendleft   [O(1)]",
          "dq.appendleft(1)",
          setup=f"from collections import deque; dq = deque(range({n}))")

# دسترسی به اندیس وسط
benchmark("list[n//2]         [O(1)]",
          "x = lst[5000]",
          setup=f"lst = list(range({n}))")
benchmark("deque[n//2]        [O(n)]",
          "x = dq[5000]",
          setup=f"from collections import deque; dq = deque(range({n}))")

# خروجی تقریبی:
# list.append                            : 4.21 ms
# deque.append                           : 5.12 ms    # کمی کندتر
# list.insert(0, x)  [O(n)]             : 198.43 ms  # خیلی کند!
# deque.appendleft   [O(1)]             : 4.98 ms    # خیلی سریع!
# list[n//2]         [O(1)]             : 2.31 ms    # سریع
# deque[n//2]        [O(n)]             : 89.21 ms   # کند!
```

### ۸.۳ — جستجو: list vs set vs bisect

```python
import random
import bisect
import timeit

n = 10000
data = list(range(n))
data_set = set(data)
data_sorted = sorted(data)

target = random.randint(0, n-1)

print("=" * 60)
print(f"جستجوی یک عنصر در {n} عنصر")
print("=" * 60)

# جستجوی خطی در list
benchmark("list: 'x in list'  [O(n)]",
          f"{target} in data",
          setup=f"data = list(range({n}))")

# جستجو در set
benchmark("set: 'x in set'    [O(1)]",
          f"{target} in data_set",
          setup=f"data_set = set(range({n}))")

# جستجوی دودویی در list مرتب
benchmark("bisect: O(log n)",
          f"bisect.bisect_left(data_sorted, {target})",
          setup=f"import bisect; data_sorted = list(range({n}))")

# خروجی تقریبی:
# list: 'x in list'  [O(n)]             : 45.23 ms
# set: 'x in set'    [O(1)]             : 1.21 ms   # خیلی سریع!
# bisect: O(log n)                       : 3.42 ms


print("\n" + "=" * 60)
print("مقایسه حافظه برای جستجو")
print("=" * 60)

import sys
n = 100000
lst = list(range(n))
st = set(range(n))

print(f"list({n:,} عنصر):  {sys.getsizeof(lst):,} bytes")
print(f"set({n:,} عنصر):   {sys.getsizeof(st):,} bytes")
# set حدود ۸ برابر بیشتر حافظه مصرف می‌کند
# اما جستجو O(1) است
```

### ۸.۴ — range vs list: زمان و حافظه

```python
import sys
import timeit

print("=" * 60)
print("حافظه: range vs list")
print("=" * 60)

for exp in [3, 6, 9, 12]:
    n = 10 ** exp
    r = range(n)
    # ساخت list فقط تا n=10^6 عملی است
    if n <= 10**6:
        lst = list(range(n))
        lst_size = sys.getsizeof(lst)
    else:
        lst_size = n * 8  # تخمین

    range_size = sys.getsizeof(r)
    print(f"n=10^{exp:2d}: range={range_size:6d}B, "
          f"list≈{lst_size:15,}B, "
          f"نسبت={lst_size/range_size:,.0f}x")

# خروجی:
# n=10^ 3: range=    48B, list=          9,016B, نسبت=188x
# n=10^ 6: range=    48B, list=      8,000,056B, نسبت=166,668x
# n=10^ 9: range=    48B, list≈  8,000,000,000B, نسبت=166,666,667x
# n=10^12: range=    48B, list≈8,000,000,000,000B, نسبت=166,666,666,667x


print("\n" + "=" * 60)
print("سرعت: جستجو در range vs list")
print("=" * 60)

# جستجو در range: O(1) -- محاسبه ریاضی
benchmark("range(10**9): x in range",
          "500000000 in range(10**9)")

# جستجو در list: O(n)
# (برای n بزرگ عملی نیست)
benchmark("list(1000): x in list",
          "500 in lst",
          setup="lst = list(range(1000))")
```

### ۸.۵ — str concatenation: دام معروف

```python
import timeit

n = 1000

print("=" * 60)
print(f"ساخت string از {n} قطعه")
print("=" * 60)

# ── روش بد: += در loop -- O(n²) ──
def bad_concat(words):
    result = ""
    for word in words:
        result += word  # هر بار یک string جدید می‌سازد!
    return result

# ── روش خوب: join -- O(n) ──
def good_concat(words):
    return "".join(words)

words = ["word"] * n

time_bad = timeit.timeit(
    lambda: bad_concat(words), number=1000
)
time_good = timeit.timeit(
    lambda: good_concat(words), number=1000
)

print(f"'+=' در loop:  {time_bad*1000:.2f} ms")
print(f"''.join():     {time_good*1000:.2f} ms")
print(f"join چقدر سریع‌تر: {time_bad/time_good:.0f}x")

# خروجی تقریبی:
# '+=' در loop:  245.32 ms
# ''.join():       4.21 ms
# join چقدر سریع‌تر: 58x


# ── چرا؟ ──
# هر بار که += می‌زنیم، یک string جدید در حافظه ساخته می‌شود:
# iteration 1: "word"            -- 4 chars
# iteration 2: "wordword"        -- 8 chars
# iteration 3: "wordwordword"    -- 12 chars
# ...
# مجموع: 4 + 8 + 12 + ... + 4n = 4 * n*(n+1)/2 = O(n²)

# اما join یک بار همه را می‌خواند و یک بار string می‌سازد: O(n)


# ── با list هم همین مشکل وجود ندارد ──
# چون list از dynamic array استفاده می‌کند:
lst = []
for i in range(1000):
    lst.append(i)  # O(1) amortized

# vs string:
s = ""
for i in range(1000):
    s += str(i)   # O(n²) کل!
```

---

## بخش نهم: تست نوشتن برای Sequence‌ها

### ۹.۱ — تست‌های پایه برای هر Sequence

```python
import pytest
from collections.abc import Sequence, MutableSequence


class SequenceTests:
    """
    مجموعه تست‌هایی که هر Sequence باید pass کند.
    می‌توانید از این کلاس برای تست کلاس‌های خودتان ارث ببرید.
    """

    # این متد باید در زیرکلاس override شود
    def make_sequence(self, data):
        raise NotImplementedError

    # ── تست‌های اساسی ──

    def test_is_sequence(self):
        """باید Sequence باشد"""
        seq = self.make_sequence([1, 2, 3])
        assert isinstance(seq, Sequence)

    def test_len(self):
        """__len__ باید درست کار کند"""
        assert len(self.make_sequence([])) == 0
        assert len(self.make_sequence([1])) == 1
        assert len(self.make_sequence([1, 2, 3])) == 3

    def test_getitem_positive(self):
        """دسترسی با اندیس مثبت"""
        seq = self.make_sequence([10, 20, 30])
        assert seq[0] == 10
        assert seq[1] == 20
        assert seq[2] == 30

    def test_getitem_negative(self):
        """دسترسی با اندیس منفی"""
        seq = self.make_sequence([10, 20, 30])
        assert seq[-1] == 30
        assert seq[-2] == 20
        assert seq[-3] == 10

    def test_getitem_out_of_range(self):
        """اندیس خارج از محدوده باید IndexError بدهد"""
        seq = self.make_sequence([1, 2, 3])
        with pytest.raises(IndexError):
            _ = seq[3]
        with pytest.raises(IndexError):
            _ = seq[-4]

    def test_slice_basic(self):
        """slice پایه"""
        seq = self.make_sequence([1, 2, 3, 4, 5])
        assert list(seq[1:3]) == [2, 3]
        assert list(seq[:2]) == [1, 2]
        assert list(seq[3:]) == [4, 5]
        assert list(seq[:]) == [1, 2, 3, 4, 5]

    def test_slice_step(self):
        """slice با step"""
        seq = self.make_sequence([1, 2, 3, 4, 5])
        assert list(seq[::2]) == [1, 3, 5]
        assert list(seq[1::2]) == [2, 4]
        assert list(seq[::-1]) == [5, 4, 3, 2, 1]

    def test_slice_empty(self):
        """slice خالی"""
        seq = self.make_sequence([1, 2, 3])
        assert list(seq[5:10]) == []
        assert list(seq[2:1]) == []

    def test_contains(self):
        """عملگر in"""
        seq = self.make_sequence([1, 2, 3])
        assert 1 in seq
        assert 2 in seq
        assert 3 in seq
        assert 4 not in seq
        assert 0 not in seq

    def test_iter(self):
        """قابل iterate بودن"""
        data = [1, 2, 3, 4, 5]
        seq = self.make_sequence(data)
        assert list(seq) == data

    def test_reversed(self):
        """reversed"""
        seq = self.make_sequence([1, 2, 3])
        assert list(reversed(seq)) == [3, 2, 1]

    def test_index(self):
        """متد index"""
        seq = self.make_sequence([10, 20, 30, 20, 10])
        assert seq.index(10) == 0
        assert seq.index(20) == 1
        assert seq.index(30) == 2

    def test_index_with_start_stop(self):
        """index با start و stop"""
        seq = self.make_sequence([10, 20, 30, 20, 10])
        assert seq.index(20, 2) == 3   # از اندیس ۲ به بعد
        assert seq.index(10, 0, 3) == 0

    def test_index_not_found(self):
        """index باید ValueError بدهد وقتی پیدا نشود"""
        seq = self.make_sequence([1, 2, 3])
        with pytest.raises(ValueError):
            seq.index(99)

    def test_count(self):
        """متد count"""
        seq = self.make_sequence([1, 2, 1, 3, 1, 2])
        assert seq.count(1) == 3
        assert seq.count(2) == 2
        assert seq.count(3) == 1
        assert seq.count(99) == 0

    def test_empty_sequence(self):
        """Sequence خالی"""
        seq = self.make_sequence([])
        assert len(seq) == 0
        assert list(seq) == []
        assert list(reversed(seq)) == []
        assert 1 not in seq

    def test_single_element(self):
        """Sequence یک عنصری"""
        seq = self.make_sequence([42])
        assert len(seq) == 1
        assert seq[0] == 42
        assert seq[-1] == 42
        assert 42 in seq
        assert list(seq) == [42]


class MutableSequenceTests(SequenceTests):
    """
    تست‌های اضافه برای MutableSequence
    """

    def test_is_mutable_sequence(self):
        """باید MutableSequence باشد"""
        seq = self.make_sequence([1, 2, 3])
        assert isinstance(seq, MutableSequence)

    def test_setitem(self):
        """تغییر عنصر"""
        seq = self.make_sequence([1, 2, 3])
        seq[1] = 99
        assert seq[1] == 99
        assert list(seq) == [1, 99, 3]

    def test_setitem_negative(self):
        """تغییر با اندیس منفی"""
        seq = self.make_sequence([1, 2, 3])
        seq[-1] = 99
        assert list(seq) == [1, 2, 99]

    def test_delitem(self):
        """حذف عنصر"""
        seq = self.make_sequence([1, 2, 3, 4])
        del seq[1]
        assert list(seq) == [1, 3, 4]
        assert len(seq) == 3

    def test_insert(self):
        """درج عنصر"""
        seq = self.make_sequence([1, 2, 3])
        seq.insert(1, 99)
        assert list(seq) == [1, 99, 2, 3]

    def test_insert_at_beginning(self):
        """درج در ابتدا"""
        seq = self.make_sequence([1, 2, 3])
        seq.insert(0, 0)
        assert list(seq) == [0, 1, 2, 3]

    def test_insert_at_end(self):
        """درج در انتها"""
        seq = self.make_sequence([1, 2, 3])
        seq.insert(len(seq), 4)
        assert list(seq) == [1, 2, 3, 4]

    def test_append(self):
        """append"""
        seq = self.make_sequence([1, 2])
        seq.append(3)
        assert list(seq) == [1, 2, 3]

    def test_extend(self):
        """extend"""
        seq = self.make_sequence([1, 2])
        seq.extend([3, 4, 5])
        assert list(seq) == [1, 2, 3, 4, 5]

    def test_iadd(self):
        """+= operator"""
        seq = self.make_sequence([1, 2])
        seq += [3, 4]
        assert list(seq) == [1, 2, 3, 4]

    def test_remove(self):
        """حذف اولین تکرار"""
        seq = self.make_sequence([1, 2, 3, 2, 1])
        seq.remove(2)
        assert list(seq) == [1, 3, 2, 1]

    def test_remove_not_found(self):
        """حذف عنصر غیرموجود باید ValueError بدهد"""
        seq = self.make_sequence([1, 2, 3])
        with pytest.raises(ValueError):
            seq.remove(99)

    def test_pop_last(self):
        """pop آخرین عنصر"""
        seq = self.make_sequence([1, 2, 3])
        val = seq.pop()
        assert val == 3
        assert list(seq) == [1, 2]

    def test_pop_index(self):
        """pop با اندیس"""
        seq = self.make_sequence([1, 2, 3, 4])
        val = seq.pop(1)
        assert val == 2
        assert list(seq) == [1, 3, 4]

    def test_clear(self):
        """clear"""
        seq = self.make_sequence([1, 2, 3])
        seq.clear()
        assert len(seq) == 0
        assert list(seq) == []

    def test_reverse(self):
        """reverse درجا"""
        seq = self.make_sequence([1, 2, 3, 4, 5])
        seq.reverse()
        assert list(seq) == [5, 4, 3, 2, 1]


# ── استفاده از این تست‌ها ──

# برای GoodSequence که قبلاً ساختیم:
class TestGoodSequence(SequenceTests):
    def make_sequence(self, data):
        return GoodSequence(data)


# برای TypedList:
class TestTypedIntList(MutableSequenceTests):
    def make_sequence(self, data):
        return TypedList(int, data)


# برای SortedList (با محدودیت‌هایش):
class TestSortedList:
    """SortedList رفتار خاصی دارد -- تست‌های مخصوص"""

    def test_always_sorted(self):
        sl = SortedList([5, 3, 1, 4, 2])
        assert list(sl) == [1, 2, 3, 4, 5]

    def test_add_maintains_order(self):
        sl = SortedList([1, 3, 5])
        sl.add(2)
        assert list(sl) == [1, 2, 3, 5]
        sl.add(4)
        assert list(sl) == [1, 2, 3, 4, 5]

    def test_contains_is_fast(self):
        """جستجو باید کار کند"""
        sl = SortedList(range(1000))
        assert 500 in sl
        assert 1001 not in sl

    def test_irange(self):
        sl = SortedList([1, 3, 5, 7, 9])
        assert sl.irange(3, 7) == [3, 5, 7]
        assert sl.irange(None, 5) == [1, 3, 5]
        assert sl.irange(5, None) == [5, 7, 9]
```

### ۹.۲ — Property-Based Testing با hypothesis

```python
from hypothesis import given, strategies as st
from hypothesis import assume


class TestSequenceProperties:
    """
    تست‌های مبتنی بر خواص (property-based testing)
    hypothesis مقادیر تصادفی تولید می‌کند و خواص را بررسی می‌کند
    """

    @given(st.lists(st.integers()))
    def test_len_matches_iteration(self, data):
        """طول باید با تعداد عناصر iterate‌شده برابر باشد"""
        seq = GoodSequence(data)
        assert len(seq) == len(list(seq))

    @given(st.lists(st.integers(), min_size=1))
    def test_getitem_consistent_with_iter(self, data):
        """دسترسی با اندیس باید با iterate یکسان باشد"""
        seq = GoodSequence(data)
        iterated = list(seq)
        for i in range(len(seq)):
            assert seq[i] == iterated[i]

    @given(st.lists(st.integers(), min_size=1))
    def test_negative_index(self, data):
        """اندیس منفی باید از آخر حساب شود"""
        seq = GoodSequence(data)
        n = len(seq)
        for i in range(1, n + 1):
            assert seq[-i] == seq[n - i]

    @given(st.lists(st.integers()))
    def test_reversed_reversed_is_original(self, data):
        """دوبار معکوس کردن باید اصل را بدهد"""
        seq = GoodSequence(data)
        assert list(reversed(reversed(seq))) == list(seq)

    @given(st.lists(st.integers()), st.integers())
    def test_count_non_negative(self, data, value):
        """count باید همیشه غیرمنفی باشد"""
        seq = GoodSequence(data)
        assert seq.count(value) >= 0

    @given(st.lists(st.integers()), st.integers())
    def test_contains_consistent_with_count(self, data, value):
        """in باید با count > 0 سازگار باشد"""
        seq = GoodSequence(data)
        assert (value in seq) == (seq.count(value) > 0)

    @given(st.lists(st.integers(), min_size=1), st.integers())
    def test_index_finds_element(self, data, value):
        """اگر index موفق شد، آن اندیس باید مقدار درستی داشته باشد"""
        seq = GoodSequence(data)
        assume(value in seq)
        idx = seq.index(value)
        assert seq[idx] == value

    @given(st.lists(st.integers()))
    def test_slice_length(self, data):
        """طول slice باید درست باشد"""
        seq = GoodSequence(data)
        n = len(seq)
        if n > 0:
            for start in range(n):
                for stop in range(start, n + 1):
                    sliced = seq[start:stop]
                    assert len(list(sliced)) == stop - start
```

---

## بخش دهم: جمع‌بندی و بهترین شیوه‌ها

### ۱۰.۱ — چه وقت از کدام استفاده کنیم؟

```python
"""
راهنمای انتخاب نوع Sequence مناسب:

┌─────────────────────────────────────────────────────────────┐
│ آیا نیاز به تغییر دارید؟                                     │
│                                                             │
│  خیر ──→ آیا عناصر همنوع هستند؟                              │
│          │                                                  │
│          ├── خیر ──→ tuple                                  │
│          │                                                  │
│          └── بله ──→ array.array یا bytes                   │
│                                                             │
│  بله ──→ آیا عناصر همنوع هستند؟                              │
│          │                                                  │
│          ├── بله ──→ array.array یا bytearray               │
│          │                                                  │
│          └── خیر ──→ آیا نیاز به insert از ابتدا دارید؟    │
│                      │                                      │
│                      ├── بله ──→ deque                      │
│                      │                                      │
│                      └── خیر ──→ list                       │
└─────────────────────────────────────────────────────────────┘
"""

# ── خلاصه‌ی کاربرد ──

# list: وقتی به تغییر نیاز دارید و عناصر متنوع هستند
shopping = ["milk", "eggs", "bread"]

# tuple: وقتی داده ثابت و چندتایی دارید
point = (3.0, 4.0, 5.0)
rgb = (255, 128, 0)

# str: وقتی با متن کار می‌کنید
name = "Alice"

# range: وقتی به یک دنباله عددی نیاز دارید
for i in range(100):
    pass

# bytes: داده باینری خام، ثابت
header = b"\x89PNG\r\n\x1a\n"

# bytearray: داده باینری خام، قابل تغییر
buffer = bytearray(1024)

# array.array: اعداد همنوع -- کارایی بالا
from array import array
temperatures = array('f', [36.5, 37.1, 36.8])

# deque: وقتی از هر دو طرف insert/delete دارید
from collections import deque
task_queue = deque()

# memoryview: وقتی با داده باینری بزرگ کار می‌کنید
data = bytearray(10**6)
view = memoryview(data)
```

### ۱۰.۲ — اصول طراحی Sequence سفارشی

```python
"""
چک‌لیست طراحی Sequence سفارشی:

✅ ۱. همیشه __getitem__ و __len__ را پیاده‌سازی کنید
✅ ۲. __getitem__ باید slice را هندل کند
✅ ۳. __getitem__ باید اندیس منفی را هندل کند  
✅ ۴. IndexError درست پرتاب کنید
✅ ۵. نوع برگشتی slice باید همان نوع Sequence باشد
✅ ۶. اگر MutableSequence هستید، __setitem__ و __delitem__ و insert
✅ ۷. در صورت نیاز، متدهای Mixin را override کنید
✅ ۸. __repr__ مناسب بنویسید
✅ ۹. تست بنویسید با SequenceTests کلاس
✅ ۱۰. مستندسازی کنید
"""


# ── قالب پیاده‌سازی Sequence ──
from collections.abc import Sequence


class MySequenceTemplate(Sequence):
    """
    قالب پیشنهادی برای Sequence سفارشی.
    """

    def __init__(self, data=None):
        self._data = list(data) if data is not None else []

    # ── دو متد اجباری ──

    def __getitem__(self, index):
        # ۱. هندل کردن slice
        if isinstance(index, slice):
            return type(self)(self._data[index])

        # ۲. هندل کردن اندیس منفی
        if isinstance(index, int):
            if index < 0:
                index += len(self._data)
            if not 0 <= index < len(self._data):
                raise IndexError(
                    f"{type(self).__name__} index out of range"
                )
            return self._data[index]

        # ۳. نوع نامعتبر
        raise TypeError(
            f"indices must be integers or slices, "
            f"not {type(index).__name__}"
        )

    def __len__(self):
        return len(self._data)

    # ── متدهای توصیه‌شده ──

    def __repr__(self):
        return f"{type(self).__name__}({self._data!r})"

    def __eq__(self, other):
        if isinstance(other, type(self)):
            return self._data == other._data
        if isinstance(other, Sequence):
            return list(self) == list(other)
        return NotImplemented

    def __hash__(self):
        # اگر immutable است:
        return hash(tuple(self._data))
        # اگر mutable است، __hash__ = None


# ── قالب پیاده‌سازی MutableSequence ──
from collections.abc import MutableSequence


class MyMutableSequenceTemplate(MutableSequence):
    """
    قالب پیشنهادی برای MutableSequence سفارشی.
    """

    def __init__(self, data=None):
        self._data = list(data) if data is not None else []

    # ── پنج متد اجباری ──

    def __getitem__(self, index):
        if isinstance(index, slice):
            return type(self)(self._data[index])
        if isinstance(index, int):
            if index < 0:
                index += len(self._data)
            if not 0 <= index < len(self._data):
                raise IndexError(
                    f"{type(self).__name__} index out of range"
                )
            return self._data[index]
        raise TypeError(f"invalid index type: {type(index).__name__}")

    def __setitem__(self, index, value):
        if isinstance(index, slice):
            self._data[index] = list(value)
        elif isinstance(index, int):
            if index < 0:
                index += len(self._data)
            if not 0 <= index < len(self._data):
                raise IndexError(
                    f"{type(self).__name__} index out of range"
                )
            self._data[index] = value
        else:
            raise TypeError(f"invalid index type: {type(index).__name__}")

    def __delitem__(self, index):
        del self._data[index]

    def __len__(self):
        return len(self._data)

    def insert(self, index, value):
        self._data.insert(index, value)

    # ── متدهای توصیه‌شده ──

    def __repr__(self):
        return f"{type(self).__name__}({self._data!r})"

    def __eq__(self, other):
        if isinstance(other, Sequence):
            return list(self) == list(other)
        return NotImplemented

    # MutableSequence نباید hashable باشد:
    __hash__ = None
```

### ۱۰.۳ — خلاصه نهایی

```python
"""
آنچه یاد گرفتیم:

۱. اجداد Sequence:
   - Iterable   → __iter__
   - Sized      → __len__
   - Container  → __contains__
   - Collection → ترکیب سه تای بالا
   - Reversible → __reversed__

۲. Sequence = Collection + __getitem__ + ترتیب + اندیس
   متدهای رایگان: __contains__, __iter__, __reversed__, index, count

۳. MutableSequence = Sequence + __setitem__ + __delitem__ + insert
   متدهای رایگان: append, clear, reverse, extend, pop, remove, __iadd__

۴. فرزندان اصلی:
   - list:       mutable, ناهمگن, dynamic array
   - tuple:      immutable, ناهمگن, hashable
   - str:        immutable, یونیکد, rich API
   - range:      immutable, lazy, O(1) for contains
   - bytes:      immutable, باینری
   - bytearray:  mutable, باینری
   - memoryview: بدون کپی، انعطاف‌پذیر
   - array.array: mutable, همگن, کارا

۵. برای Sequence سفارشی:
   - دو متد برای Sequence: __getitem__ + __len__
   - پنج متد برای MutableSequence: + __setitem__ + __delitem__ + insert

۶. دام‌های مهم:
   - فراموش کردن slice در __getitem__
   - تغییر Sequence حین iterate
   - کپی سطحی vs عمیق
   - NaN در count و index
   - += در string (از join استفاده کنید)

۷. انتخاب درست:
   - ثابت + چندنوع  → tuple
   - متغیر + چندنوع → list  
   - فقط اعداد      → array.array
   - جستجوی سریع    → set (نه Sequence!)
   - insert از هر طرف → deque
   - داده باینری بزرگ → memoryview
"""
```

---

*پایان*

> این مقاله تلاش کرده کامل‌ترین بررسی از سلسله‌مراتب `Sequence` در پایتون را ارائه دهد.
> اگر سوالی دارید یا اشتباهی دیدید، خوشحال می‌شوم در نظرات بنویسید.